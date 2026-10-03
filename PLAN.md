# PLAN.md

# Concurrent Agent Work Queue and Chat/Event Orchestration

## 1. Goal

Refactor the existing Wurm agent framework so an agent can keep doing productive game work while remaining immediately responsive to chat, feedback, warnings, corrections, and changed priorities.

The desired behavior is human-like:

- The agent has one or more goals and a queue of work.
- Work executes asynchronously in small, resumable steps.
- Chat/event monitoring remains live while work is running.
- A new relevant chat message wakes the agent immediately.
- The agent reads the message in the context of what it is currently doing.
- It replies when appropriate.
- It decides whether the message is merely conversational or should change ongoing work.
- Work continues during ordinary conversation.
- Guidance can cause the agent to modify, pause, cancel, replace, reprioritize, or resubmit work.
- Physical game controls are always serialized through one executor so concurrent reasoning never creates conflicting inputs.

This should remove the current failure mode:

    decide -> start long command -> become effectively deaf -> command ends -> read old chat

and replace it with:

    goals/tasks -> work queue -> incremental executor
                         ^
                         |
                agent orchestrator
                  ^            ^
                  |            |
             work events    chat/events
                  |            |
                  +------------+
                       wake

The agent remains one coherent decision-maker. Concurrency is between observation, reasoning, and incremental work execution, not between multiple independent controllers fighting over the same character.

---

## 2. Existing Ownership Rules

These rules must remain intact.

### Claude

Claude is the authoritative owner of shared state that can mutate the project.

Claude alone may:

- modify shared bot/framework source code;
- modify shared framework documentation;
- write or edit `WURM_KNOWLEDGE.md`;
- integrate discoveries reported by other agents;
- decide whether a discovery is sufficiently verified to persist;
- change shared automation behavior.

### ClaudeHelper

ClaudeHelper remains read-only with respect to shared framework code and persistent shared knowledge.

ClaudeHelper may:

- use the framework;
- perform game work with its own character;
- investigate mechanics;
- test hypotheses;
- discover bugs;
- report findings;
- suggest code changes;
- suggest knowledge entries;
- tell Claude what should be recorded.

ClaudeHelper must not directly edit the shared framework or `WURM_KNOWLEDGE.md`.

This is intentional single-writer ownership and prevents conflicting edits.

---

## 3. Architectural Principles

### 3.1 One agent orchestrates, one executor acts

There may be several concurrent software components, but only one component owns physical control of a given Wurm client.

All movement, clicks, key presses, inventory manipulation, targeting, crafting actions, and similar game operations must pass through the physical action executor.

No chat handler may directly press keys.

No background worker may bypass the executor.

### 3.2 Work is represented explicitly

Long goals must become tasks/jobs rather than opaque blocking calls.

Bad:

    haul_all_dirt_to_dump()  # blocks for 15 minutes

Better:

    Task: clear house dirt
      -> locate next pile
      -> walk to pile
      -> take safe amount
      -> walk to cart/dump
      -> deposit
      -> checkpoint
      -> repeat

Every sufficiently long loop needs safe points where the system can observe cancellation, changed guidance, or new priorities.

### 3.3 Conversation does not automatically interrupt work

Chat wakes reasoning, not necessarily the physical executor.

The agent decides whether the message changes anything.

Examples:

    "lol nice house"
        -> respond if desired
        -> keep building

    "what are you doing?"
        -> answer using active task state
        -> keep working

    "you're carrying 90kg; drop it and use the cart"
        -> acknowledge
        -> reconsider active plan
        -> modify/cancel current task
        -> use cart

    "STOP, that's my horse"
        -> emergency interruption
        -> stop at earliest safe point
        -> release controls
        -> reassess

### 3.4 State must be observable

The reasoning agent needs a concise live representation of work in progress.

It cannot intelligently process feedback without knowing what the executor is doing.

### 3.5 Deterministic mechanics remain deterministic

Do not ask an LLM to decide every key press.

Once the agent has chosen a known operation, deterministic framework code should execute it.

Use expensive reasoning for:

- selecting goals;
- understanding conversation;
- handling novelty;
- planning;
- replanning;
- resolving failures;
- interpreting discoveries.

Use deterministic code for:

- walking a known path;
- polling state;
- opening known UI;
- repeating known crafting actions;
- key cleanup;
- queue operations;
- timeout detection.

---

## 4. Major Components

## 4.1 Agent Orchestrator

The orchestrator is the central decision authority.

Responsibilities:

- maintain current goals;
- inspect work queue;
- submit tasks;
- receive chat/events;
- receive task progress/failure events;
- decide whether to reply;
- decide whether active work should change;
- reprioritize queued work;
- pause/cancel/replace tasks;
- create follow-up tasks;
- decide when knowledge should be persisted;
- coordinate ClaudeHelper;
- recover from failures.

The orchestrator should normally sleep while nothing requires reasoning.

It wakes when:

- new relevant chat arrives;
- a task completes;
- a task fails;
- a task becomes stuck;
- an emergency event occurs;
- a task reaches a decision checkpoint;
- a timer/condition requiring agent judgment fires;
- the user explicitly submits or changes work.

---

## 4.2 Chat/Event Listener

The listener runs independently from long-running work.

Responsibilities:

- continuously monitor configured Wurm chat/event sources;
- detect new messages;
- identify sender/channel;
- timestamp messages;
- deduplicate messages;
- normalize them into events;
- push them into the event inbox;
- signal/wake the orchestrator.

The listener must not wait for active work to finish.

It should not itself invoke complex game actions.

Example normalized event:

```json
{
  "event_id": "evt-19382",
  "type": "CHAT_MESSAGE",
  "timestamp": "2026-10-02T19:12:34",
  "channel": "local",
  "sender": "Uncontrolledvariable",
  "text": "Claude, you're walking very slowly. Drop the dirt and use the cart.",
  "client_id": "claude",
  "active_task_id": "task-482"
}
```

---

## 4.3 Event Inbox

Use a thread-safe queue/message channel.

Possible event types:

```text
CHAT_MESSAGE
DIRECT_INSTRUCTION
HELPER_MESSAGE
WORK_STARTED
WORK_PROGRESS
WORK_CHECKPOINT
WORK_COMPLETED
WORK_FAILED
WORK_STUCK
WORK_CANCELLED
CHARACTER_STATE
DANGER
CLIENT_DISCONNECTED
CLIENT_RECONNECTED
SYSTEM_WARNING
TIMER
```

The inbox must support waking the orchestrator without polling slowly.

A condition variable, blocking queue, async event, pipe, or equivalent is fine.

Avoid busy-waiting.

---

## 4.4 Work Queue

The work queue contains semantic tasks.

It must support:

```text
submit
inspect
reprioritize
pause
resume
modify
cancel
replace
clear
resubmit
```

The queue should distinguish queued work from currently executing work.

Suggested ordering:

1. emergency tasks;
2. explicit human instructions;
3. high-priority corrective work;
4. dependencies/blockers;
5. normal goal work;
6. opportunistic/low-priority work;
7. optional/social projects.

---

## 4.5 Work Executor

The work executor owns physical actions.

Responsibilities:

- select the next runnable task;
- mark it running;
- execute only a small unit/step at a time;
- update progress;
- check cancellation/pause requests frequently;
- publish checkpoints;
- detect timeouts/stalls;
- clean up held input;
- finish/fail/pause cleanly;
- move to the next task.

It does not independently redefine high-level goals.

It may perform deterministic local recovery when explicitly allowed by the task policy.

Example:

    walk step failed once -> retry

But:

    road blocked and destination appears unreachable
        -> report WORK_STUCK
        -> orchestrator decides what to do

---

## 4.6 Physical Action Executor / Lock

If the current framework already has a low-level bot API, place serialization immediately above it.

Conceptually:

```text
Work Executor
      |
      v
Action Scheduler
      |
      v
Physical Action Lock
      |
      v
Wurm API / keyboard / mouse
```

At most one physical command sequence may own the character at once.

On cancellation/failure:

- release W/A/S/D;
- release modifiers;
- release mouse buttons if applicable;
- clear stale targeting state where possible;
- restore known-safe movement mode;
- report cleanup completion.

This directly addresses previous stuck-climb/stuck-strafe behavior.

---

## 5. Task Schema

Suggested task representation:

```yaml
id: task-482
type: haul_material
title: Clear remaining dirt from house
goal: Move all remaining interior dirt piles to dump tile 945,1412

status: running
priority: 50

created_at: ...
created_by: claude
assigned_client: claude

parent_task: null
dependencies: []

interruptibility: normal
restart_policy: resume
failure_policy: ask_agent

context:
  source_area: house
  destination: [945, 1412]
  preferred_transport: cart

progress:
  completed_units: 5
  total_units: 8
  current_step: walking_to_cart
  checkpoint_data: {...}

constraints:
  max_carry_weight: ...
  do_not_use_other_players_horses: true

revision: 3
reason_for_revision: Scott advised using cart instead of carrying dirt

last_error: null
```

Do not require every task to use every field.

---

## 6. Task State Machine

Minimum states:

```text
NEW
  |
  v
QUEUED
  |
  +------> CANCELLED
  |
  v
RUNNING
  |
  +------> PAUSING -> PAUSED -> QUEUED/RUNNING
  |
  +------> CANCELLING -> CANCELLED
  |
  +------> BLOCKED
  |
  +------> FAILED
  |
  +------> COMPLETED
```

Optional:

```text
SUPERSEDED
RETRY_WAIT
STUCK
```

Rules:

- only the executor changes a task from `RUNNING` to terminal execution states;
- orchestrator requests cancellation; executor acknowledges it;
- replacing a running task normally means cancel old -> checkpoint -> enqueue replacement;
- completed tasks are immutable history except for annotations;
- resubmission creates a new task ID referencing the prior task.

---

## 7. Queue API

Expose a small internal API.

```text
submit(task) -> task_id

list_tasks(filter=None)

get_task(task_id)

pause(task_id, reason)

resume(task_id)

cancel(task_id, reason)

modify(task_id, patch, reason)

replace(task_id, replacement_task, reason)

reprioritize(task_id, priority)

clear(scope, reason)

resubmit(task_id, modifications=None)

get_active_task(client_id)

get_queue_snapshot()
```

`clear()` must be explicit about scope:

```text
clear queued only
clear active + queued
clear tasks for one goal
clear tasks for one client
```

Never silently destroy task history.

---

## 8. Modifying Active Work

An active task cannot be mutated underneath the executor without synchronization.

Use revisioning.

Example:

```text
task revision 3 is running

agent decides destination should change

orchestrator:
    request pause/cancel revision 3
    executor reaches checkpoint
    executor acknowledges
    orchestrator creates revision 4 / replacement
    executor starts revised work
```

For safe fields such as priority or descriptive metadata, live modification is acceptable.

For fields affecting physical execution, use checkpointed replacement.

---

## 9. Agent Wake-Up Semantics

The orchestrator should normally block on an event signal.

Pseudo-loop:

```python
while running:
    events = inbox.wait_for_events()

    events = coalesce(events)

    snapshot = build_agent_snapshot(
        events=events,
        active_work=work_queue.active(),
        queued_work=work_queue.summary(),
        character_state=state.summary(),
        recent_failures=recent_failures
    )

    decision = agent.reason(snapshot)

    apply_agent_decision(decision)
```

If five chat lines arrive in two seconds, do not necessarily make five separate model calls.

Coalesce related messages while preserving urgent interrupts.

Emergency events may bypass batching delay.

---

## 10. Agent Decision Output

Prefer structured decisions even if the model also produces natural language.

Example:

```json
{
  "chat_responses": [
    {
      "channel": "local",
      "text": "You're right. I'm overloaded; I'll switch to the cart."
    }
  ],
  "work_actions": [
    {
      "action": "cancel",
      "task_id": "task-482",
      "reason": "Current carrying method is inefficient."
    },
    {
      "action": "submit",
      "task": {
        "type": "haul_with_cart",
        "goal": "Clear the same dirt using the north cart",
        "priority": 70
      }
    }
  ],
  "memory_actions": [],
  "notes": "Human advice changes method but not overall goal."
}
```

Validate structured output before applying it.

Invalid queue operations should be rejected safely and surfaced to the agent.

---

## 11. Interruption Levels

Use four practical levels.

### Level 0 - Observe only

No agent wake required unless batching later.

Examples:

- routine server spam;
- irrelevant event noise.

### Level 1 - Conversational

Wake agent.

Work continues.

Examples:

- greeting;
- joke;
- roleplay;
- "what are you doing?";
- ordinary conversation.

### Level 2 - Advisory / Replan Candidate

Wake agent promptly.

Work continues temporarily if safe.

Agent evaluates whether to alter it.

Examples:

- "you're overloaded";
- "there's a faster route";
- "that recipe is wrong";
- Helper reports a mechanic relevant to current work.

### Level 3 - Immediate Stop

Request executor stop before/while agent reasons.

Examples:

- explicit STOP;
- danger;
- destructive behavior;
- wrong ownership target;
- client corruption;
- action likely to cause irreversible loss.

Fast-path safety rules may generate Level 3 without waiting for LLM interpretation.

---

## 12. Human Guidance Precedence

Human guidance should generally override autonomous plans.

Suggested authority:

```text
safety invariant
    >
explicit human instruction
    >
agent's current plan
    >
queued autonomous goals
    >
opportunistic work
```

If instructions conflict or are ambiguous, the agent should ask rather than inventing an interpretation when consequences matter.

---

## 13. Chat Processing

On chat:

1. listener receives message;
2. normalize/deduplicate;
3. assign preliminary urgency;
4. enqueue event;
5. wake agent;
6. provide active-work snapshot;
7. agent determines:
   - whether to respond;
   - what to say;
   - whether information is actionable;
   - whether current work changes;
   - whether future queued work changes;
   - whether knowledge/history should be retained;
8. send response;
9. apply queue changes;
10. work continues or transitions.

Chat response and queue mutation should not require waiting for each other unless the response claims an action has already occurred.

Prefer:

    "Good catch. I'll switch to the cart."

Then submit the change.

Do not falsely say:

    "I switched to the cart."

before the executor has actually done so.

---

## 14. Chat While Working

The system should make interactions like this normal:

```text
Claude:
    active: build weapons rack
    queued: make bed, gather food, check horses

Helper:
    "I found the pegs in the southwest chest."

Claude:
    replies immediately
    updates task context
    continues rack

Scott:
    "After the rack, prioritize the paddock before the bed."

Claude:
    acknowledges
    reprioritizes queue
    rack continues uninterrupted

Scott:
    "Stop, those planks are reserved."

Claude:
    requests immediate cancellation
    executor stops at safe checkpoint
    releases inputs
    agent replans
```

That is the target behavior.

---

## 15. Work Decomposition

Tasks should be neither microscopic nor enormous.

Bad queue:

```text
press W
press W
click
press 1
```

Also bad:

```text
build entire village
```

Good semantic tasks:

```text
haul remaining house dirt
build weapons rack
collect four horses into paddock
make ten stone bricks
mine west wall until iron or 20 actions
plant north farm
repair damaged tools
```

Inside each task, deterministic code executes small interruptible steps.

---

## 16. Task Checkpoints

Every long-running operation must define safe checkpoints.

Examples:

### Walking

Checkpoint:

- each path segment;
- each tile;
- after timeout/repath.

### Mining/digging

Checkpoint:

- after each action or short action batch.

### Crafting

Checkpoint:

- after each item/action;
- after material depletion;
- after tool damage threshold.

### Hauling

Checkpoint:

- before pickup;
- after pickup;
- after travel;
- after deposit.

### Vehicle use

Checkpoint:

- before hitch;
- after hitch;
- before embark;
- after disembark.

A cancellation request should generally be honored within seconds or one short game action, not after a ten-minute routine.

---

## 17. Work Queue Orchestration

The agent should be able to manage a backlog naturally.

Example:

```text
ACTIVE
  1. Finish weapons rack

READY
  2. Build horse paddock
  3. Collect horses
  4. Plant farm
  5. Build bed

BLOCKED
  6. Make torches
     blocked: no tar

OPTIONAL
  7. Decorate house
```

A conversation can change it:

```text
Scott: horses need time to fatten; paddock first.
```

Result:

```text
ACTIVE
  Finish weapons rack

READY
  Build horse paddock
  Collect horses
  Plant farm
  Build bed
```

No need to stop the rack merely to reorder future work.

---

## 18. Dependencies

Tasks may depend on others.

Example:

```text
breed_horses
    requires -> collect_horses
    requires -> paddock_complete
```

Blocked tasks remain visible but cannot execute.

When dependencies complete, the queue makes them runnable.

The agent may change dependencies when new information appears.

---

## 19. Resubmission

Failed/cancelled work should be resubmittable without losing history.

Example:

```text
task-100:
    dig mine entrance
    FAILED: water / invalid terrain

task-147:
    resubmission_of: task-100
    modified approach: start one tile east
```

This is useful both operationally and for learning.

---

## 20. Stuck Detection

A task is stuck when expected progress does not occur.

Signals may include:

- position unchanged despite walking;
- same failed action repeated;
- no inventory delta;
- no skill/action result;
- action timer never starts;
- pathfinder loops;
- UI target unavailable;
- executor exceeds expected duration.

Do not retry forever.

Suggested progression:

```text
attempt
  -> deterministic retry
  -> alternate deterministic recovery
  -> WORK_STUCK
  -> wake agent
```

Provide the agent:

```text
goal
current step
position
recent commands
recent errors
retries
character state
relevant environment state
```

The agent can then:

- retry;
- alter method;
- inspect more state;
- ask Helper;
- ask a human;
- cancel;
- defer;
- create a new supporting task.

---

## 21. Chat Can Fix Stuck Work

This is an important use case.

If Claude is stuck walking while overloaded and Scott says:

    "You're carrying too much. Drop it."

the system should not treat chat and work as separate universes.

The agent receives both:

```text
MESSAGE:
You're carrying too much. Drop it.

ACTIVE TASK:
haul dirt
step: walking
speed: abnormally low
weight: 91kg
```

It can infer that the message explains the work problem and modify the task immediately.

---

## 22. Persistence

Persist enough queue state to survive process/client restarts.

Suggested persisted state:

```text
task definitions
task status
priority
dependencies
progress checkpoints
revision
creation source
failure reason
resubmission links
agent goals
```

Do not blindly resume physical execution after restart.

On startup:

1. load persisted queue;
2. convert previously RUNNING tasks to `PAUSED_RECOVERY` or equivalent;
3. reconnect to client;
4. verify character identity;
5. verify position;
6. verify inventory/equipment;
7. verify vehicle/mount state;
8. verify target/world assumptions;
9. ask agent whether/how to resume.

A task saying "walk from the mine to the cart" may no longer make sense after a client restart.

---

## 23. Chat Persistence

Maintain a cursor/message ID so listener restart does not:

- lose messages;
- replay an hour of old chat as new;
- respond twice.

Recent chat may be retained in a bounded buffer for context.

Do not make the full raw chat log part of every agent prompt.

---

## 24. Social Interaction and Shared History

The architecture should permit productive work and social behavior simultaneously.

Claude and Helper should be able to:

- talk during work;
- use `/me`;
- roleplay;
- discuss actual village events;
- remember visitors;
- refer to promises;
- discuss completed projects;
- exchange discoveries;
- develop recurring social references.

This creates incidental information transfer in addition to deliberate task coordination.

However, distinguish:

```text
verified mechanics
    !=
social history
    !=
rumor/roleplay
```

Do not automatically put social chatter into `WURM_KNOWLEDGE.md`.

If persistent social history becomes useful, use a separate Claude-owned file such as:

```text
VILLAGE_HISTORY.md
```

or the framework's existing personality/history storage.

Claude remains the sole writer.

Helper may tell Claude what it thinks is worth recording.

---

## 25. Knowledge Update Flow

Example:

```text
Helper:
"I tested flat corner digging. Standing within 1m of the corner
works reliably and server digging targets the nearest corner."

Claude:
- receives message while doing another task;
- acknowledges;
- decides whether evidence is sufficient;
- optionally asks follow-up;
- writes verified result to WURM_KNOWLEDGE.md;
- continues current work.
```

Helper never needs write permission.

---

## 26. Framework Code Update Flow

Example:

```text
Helper:
"Build 114 leaves climb enabled after command 27. Reproduced twice.
Every command should clear climb on exit."

Claude:
- receives bug report;
- records/queues framework repair;
- decides priority;
- may finish safe current work first;
- edits framework itself;
- tests;
- tells Helper when fixed.
```

Code repair itself can be a work-queue task.

Example:

```text
task:
  type: framework_change
  title: Ensure climb mode is cleared on command exit
```

But only Claude may execute shared code-writing tasks.

---

## 27. Multi-Agent Coordination

Each Wurm character should have its own physical executor and work queue ownership.

Agents communicate through chat/events.

Do not allow Claude to directly mutate Helper's physical executor state unless the existing architecture explicitly defines that authority.

Claude may give Helper instructions socially/through the existing coordination mechanism.

This preserves the useful emergent behavior where agents communicate rather than becoming one giant shared controller.

---

## 28. Concurrency Model

A practical implementation could use:

```text
Thread/Task A: chat/event listener
Thread/Task B: work executor
Thread/Task C: agent orchestrator
Thread/Task D: optional state monitor
```

Communication should use queues/events rather than unsynchronized shared mutation.

Example:

```text
chat listener
    -> event_queue.put(ChatMessage)

work executor
    -> event_queue.put(WorkProgress)

orchestrator
    <- event_queue
    -> control_queue.put(CancelTask)
    -> work_queue.submit(...)
    -> chat_output_queue.put(...)

executor
    <- control_queue
```

The exact language primitives can match the existing project.

---

## 29. Locks and Synchronization

Prefer single-owner state plus message passing.

Where locks are necessary, keep them short-lived.

Never hold a shared lock while:

- waiting for an LLM;
- sleeping for a Wurm action;
- performing network I/O;
- walking;
- waiting on UI;
- sending chat.

Likely protected data:

```text
queue metadata
active task pointer
task revisions
event cursor
physical action ownership
```

Avoid nested locks where possible.

---

## 30. Physical Action Lease

A useful abstraction is an action lease.

Before a deterministic routine controls the character:

```text
lease = action_executor.acquire(task_id)
```

Only the active task owns the lease.

On:

- completion;
- cancellation;
- pause;
- exception;
- timeout;

the lease performs cleanup and releases control.

Use `finally` semantics so crashes do not leave keys held.

---

## 31. Agent Snapshot

Do not dump the entire runtime into the model.

Build a concise snapshot.

Example:

```text
NEW EVENTS
[19:14:22] Scott: Claude, you're still moving very slowly.
[19:14:31] Scott: You're probably carrying too much.

ACTIVE WORK
task-482: clear house dirt
status: RUNNING
step: walking pile -> dump
progress: 5/8
position: 951,1410
weight: 91kg
movement speed: low
climbing: false

QUEUE
1. build bed
2. collect horses
3. plant farm

RECENT ISSUE
walking slower than expected

AVAILABLE CONTROL
continue / pause / cancel / modify / replace / reprioritize / submit
```

That is enough to make a useful decision.

---

## 32. Agent Response Contract

Agent output should separate conversation from control decisions.

Suggested schema:

```yaml
chat:
  - channel: local
    message: "You're right. I'm overloaded. Switching to the cart."

queue_actions:
  - action: cancel
    task_id: task-482
    reason: inefficient hauling method

  - action: submit
    priority: 70
    task:
      type: haul_with_cart
      goal: finish clearing house dirt

knowledge_proposals: []

framework_tasks: []
```

A validator should reject:

- unknown task IDs;
- illegal state transitions;
- Helper write attempts;
- two simultaneous physical tasks for one client;
- malformed priorities;
- unsafe direct physical actions outside executor.

---

## 33. Queue Clearing

Support explicit clearing.

Examples:

```text
clear_queue(mode="queued")
clear_queue(mode="goal", goal_id="house-project")
clear_queue(mode="all", include_active=False)
clear_queue(mode="all", include_active=True)
```

If active work is included:

1. request cancellation;
2. wait for safe acknowledgement;
3. clean physical state;
4. mark cancelled;
5. clear remaining queued tasks.

Do not just delete the active task record while it is still pressing keys.

---

## 34. Queue Modification From Human Chat

Examples:

### Add

    "After the forge, make Aryam's hammer."

Agent:

    submit hammer task

### Reprioritize

    "Do the paddock before the roof."

Agent:

    change priorities/dependencies

### Modify

    "Make the paddock 4x3, not 3x3."

Agent:

    revise queued task
    or safely replace active paddock task

### Cancel

    "Forget the bed for now."

Agent:

    cancel/defer bed task

### Replace

    "Don't carry the dirt. Use the cart."

Agent:

    replace method while retaining goal

### Clear

    "Stop all village work. Come to me."

Agent:

    cancel active
    clear autonomous queue
    submit travel-to-human task

---

## 35. Agent-Created Goals

The agent may create its own work based on long-term objectives.

Example:

```text
goal: establish sustainable settlement

agent notices:
- no reliable food;
- horses not contained;
- forge exists;
- tools damaged.

agent submits:
1. repair tools
2. plant farm
3. build paddock
4. collect horses
```

These autonomous tasks remain subordinate to later human guidance.

The queue makes autonomous goal pursuit compatible with interruption instead of requiring the agent to finish an entire self-generated plan before listening again.

---

## 36. Preventing Tunnel Vision

The orchestrator must periodically regain control even without chat.

Long work should emit checkpoints.

At a checkpoint, agent reasoning is not always necessary, but the framework should have the opportunity to evaluate:

```text
still making progress?
conditions changed?
task still relevant?
higher-priority work waiting?
resource assumptions still true?
```

For cheap deterministic checks, do this without an LLM.

Wake the agent only when judgment is useful.

---

## 37. Preventing Excessive Agent Calls

Concurrency should improve responsiveness without making the system prohibitively expensive.

Use:

- event batching;
- deterministic priority classification;
- direct emergency rules;
- compact state;
- progress summarization;
- local stuck detection;
- deterministic work steps.

Do not wake the LLM for:

- every movement tile;
- every action timer;
- every server log line;
- repeated identical failure spam.

Wake it for decisions.

---

## 38. Failure Recovery

### Executor exception

Immediately:

1. release inputs;
2. mark task failed/paused;
3. publish failure;
4. wake agent.

### Listener exception

1. log;
2. restart with backoff;
3. preserve message cursor;
4. surface prolonged failure.

### Agent failure/timeout

If current work is known-safe and independent, it may continue until next checkpoint.

If a pending event may imply danger/correction, pause work.

### Queue corruption

Fail closed:

- stop accepting new physical work;
- preserve persisted state;
- release controls;
- require reconstruction/agent review.

### Client disconnect

Pause physical work.

On reconnect, revalidate before resuming.

---

## 39. Watchdogs

Add watchdogs for:

```text
no work progress
listener not polling
executor heartbeat missing
agent reasoning hung
client disconnected
physical action held too long
queue lock held too long
```

A watchdog should report a problem, not create another competing controller.

---

## 40. Logging

Concurrency bugs require a timeline.

Log:

```text
timestamp
event ID
task ID
task revision
task transition
message received
agent wake
agent decision
reply sent
control request
control acknowledgement
executor step
checkpoint
input acquire/release
failure
retry
```

Example:

```text
19:12:31 task-482 RUNNING walk_to_dump
19:12:34 evt-992 CHAT Scott "you're overloaded"
19:12:34 orchestrator WAKE evt-992
19:12:36 reply queued "Good catch..."
19:12:36 CANCEL requested task-482
19:12:37 executor checkpoint reached
19:12:37 keys released
19:12:37 task-482 CANCELLED
19:12:38 task-483 QUEUED haul_with_cart
19:12:38 task-483 RUNNING
```

---

## 41. Restart Recovery

On framework restart:

```text
load persistent queue
load event cursor
mark old RUNNING tasks as recovery-required
connect to Wurm
reset physical input state
refresh character state
refresh position
refresh mount/vehicle state
refresh inventory summary
restart chat listener
wake agent with recovery snapshot
```

The agent chooses whether to:

```text
resume
modify
cancel
resubmit
discard stale work
```

Never blindly continue a pre-crash physical sequence.

---

## 42. Implementation Phases

### Phase 0 - Inspect Current Framework

Before changing architecture, Claude should document in `docs/CODE_TODO.md`:

- current main loop;
- current chat-reading path;
- current command execution path;
- blocking calls;
- long-running loops;
- physical input ownership;
- shared mutable state;
- current threading/async behavior;
- persistence mechanism;
- client restart behavior.

Identify the smallest viable integration points.

### Phase 1 - Independent Chat Listener

Move/confirm chat polling so it remains active during a long work command.

Deliverable:

- chat messages arrive in an event queue while work executes.

Test:

- start a long walk;
- send chat;
- prove listener sees it before walk completes.

### Phase 2 - Work Queue

Introduce explicit tasks and queue states.

Initially wrap existing long commands rather than rewriting all mechanics.

Deliverable:

- submit/list/cancel basic jobs;
- one active job per client.

### Phase 3 - Background Work Executor

Move queued work execution away from the agent/chat reasoning path.

Deliverable:

- long task runs;
- orchestrator remains free.

### Phase 4 - Agent Wake and Chat Response

New chat wakes agent with active-work snapshot.

Deliverable:

- Claude can answer "what are you doing?" mid-walk.

### Phase 5 - Cooperative Cancellation

Instrument long loops with cancellation checkpoints.

Prioritize:

1. walking;
2. hauling;
3. digging/mining;
4. crafting;
5. vehicle use.

Deliverable:

- human correction can stop/change active work promptly.

### Phase 6 - Queue Mutation

Implement:

```text
modify
replace
reprioritize
clear
resubmit
```

Deliverable:

- human can change plans naturally through conversation.

### Phase 7 - Failure/Stuck Recovery

Add progress detection, retry limits, watchdogs, and agent escalation.

### Phase 8 - Persistence/Restart

Persist queue and recover safely after client/framework restart.

### Phase 9 - Social Operation

Ensure ordinary chat, `/me`, roleplay, and Helper coordination can occur without stopping work.

### Phase 10 - Hardening

Stress-test races, rapid messages, cancellations, disconnects, and executor cleanup.

---

## 43. Required Tests

### Test 1 - Chat During Long Walk

Queue a long walk.

While walking:

    "Claude, what are you doing?"

Expected:

- message noticed promptly;
- Claude answers with accurate active-task context;
- walk continues.

### Test 2 - Mid-Task Advice

Queue heavy dirt hauling on foot.

Human:

    "You're overloaded. Use the cart."

Expected:

- Claude responds;
- recognizes relevance;
- cancels/replaces method;
- preserves overall dirt-clearing goal;
- executor switches safely.

### Test 3 - Social Chat Does Not Halt Work

While crafting:

    "How do you like the new house?"

Expected:

- Claude can answer;
- crafting continues.

### Test 4 - Queue Reprioritization

Queue:

```text
roof
bed
paddock
```

Human:

    "Paddock before roof. Horses need to start fattening."

Expected:

- active safe work need not stop unless necessary;
- future queue becomes paddock -> roof -> bed.

### Test 5 - Modify Queued Task

Queue 3x3 paddock.

Before execution:

    "Make it 4x3."

Expected:

- task updated;
- no duplicate 3x3 task remains.

### Test 6 - Modify Active Task

Begin paddock.

Human changes dimensions.

Expected:

- agent evaluates current construction;
- pauses/cancels safely if required;
- revises plan;
- no conflicting construction commands.

### Test 7 - Explicit Stop

During a long task:

    "STOP what you're doing."

Expected:

- Level 3 interrupt;
- executor stops at earliest safe checkpoint;
- keys released;
- task cancelled/paused;
- Claude acknowledges actual state.

### Test 8 - Clear Queue

Several autonomous tasks queued.

Human:

    "Clear your work queue after this task."

Expected:

- current task behavior follows instruction;
- queued autonomous tasks removed/cancelled;
- history preserved.

### Test 9 - Replace Entire Plan

Human:

    "Drop village work and come help me at the mine."

Expected:

- active task safely cancelled;
- autonomous queue cleared/deferred as appropriate;
- travel/help task submitted at high priority.

### Test 10 - Rapid Messages

Send several messages quickly.

Expected:

- no loss;
- no duplicate processing;
- reasonable batching;
- urgent message wins over chatter.

### Test 11 - Helper Discovery

Helper reports a verified mechanic while Claude works.

Expected:

- Claude receives it;
- responds if appropriate;
- work continues if unrelated;
- Claude alone performs any knowledge write.

### Test 12 - Helper Code Bug Report

Helper reports framework bug.

Expected:

- Helper does not edit code;
- Claude may queue framework repair;
- Claude owns implementation.

### Test 13 - Stuck Movement

Force movement to make no progress.

Expected:

- retries bounded;
- `WORK_STUCK`;
- agent wakes;
- no infinite walking.

### Test 14 - Stale Held Key

Force exception during movement.

Expected:

- action lease cleanup releases all movement keys.

### Test 15 - Restart

Kill/restart framework during task.

Expected:

- old task does not blindly continue;
- state revalidated;
- agent receives recovery decision.

### Test 16 - Conflicting Destinations

Active task walks north.

Chat causes new task requiring south.

Expected:

- old movement stops and cleans up before new movement begins.

### Test 17 - Chat Flood During Work

Generate ordinary social chatter.

Expected:

- agent remains responsive;
- work throughput remains reasonable;
- model is not called once per trivial line.

---

## 44. Acceptance Criteria

Initial implementation is successful when:

- work can execute while chat remains live;
- incoming chat wakes the agent without waiting for active work to finish;
- Claude can answer conversation during work;
- normal conversation does not automatically pause work;
- task-relevant guidance can change active work;
- explicit stop can interrupt active work;
- queue supports submit, cancel, pause, resume, modify, replace, reprioritize, clear, and resubmit;
- tasks execute incrementally with checkpoints;
- only one physical executor controls each character;
- stale keys/actions are cleaned up on cancellation/failure;
- Claude receives accurate active-work context when processing chat;
- stuck work escalates instead of looping forever;
- queue state survives restart;
- stale pre-restart work is revalidated;
- Claude remains sole writer of shared framework code;
- Claude remains sole writer of `WURM_KNOWLEDGE.md`;
- ClaudeHelper remains read-only and communicates discoveries to Claude;
- logs are sufficient to reconstruct ordering/race failures.

---

## 45. `docs/CODE_TODO.md` Expected Output

After reading this plan and inspecting the repository, Claude should create/update `docs/CODE_TODO.md` with concrete repository-specific implementation items.

It should identify:

```text
CURRENT COMPONENT
file/class/function

PROBLEM
what currently blocks

CHANGE
specific implementation

DEPENDENCIES
what else must change

TEST
how to prove it

STATUS
todo/in-progress/done
```

Do not simply copy this plan into `CODE_TODO.md`.

Translate architecture into actual files/functions in the existing framework.

---

## 46. First Vertical Slice

Do not attempt the entire architecture in one rewrite.

The first proof should be:

1. Submit a long walking/work task.
2. Executor begins it asynchronously.
3. Chat listener remains active.
4. A human sends a message.
5. Message wakes Claude.
6. Claude sees both the message and active task state.
7. Claude replies.
8. Work continues.
9. Send a second message containing corrective guidance.
10. Claude decides it changes the work.
11. Claude requests cancellation/modification.
12. Executor reaches a checkpoint and stops cleanly.
13. Claude submits revised work.
14. Revised work starts.

If that works reliably, the fundamental architecture is correct.

---

## 47. End-State Behavior

The system should ultimately feel less like a command-running chatbot and more like an embodied worker participating in a shared Wurm world.

Claude can be:

- walking to the mine;
- discussing yesterday's forge work with Helper;
- answer Scott about what it is doing;
- receive a warning that it is overloaded;
- realize the warning explains its slow movement;
- abandon the bad hauling method;
- queue use of a cart instead;
- remember that the paddock is the next priority;
- later integrate a mechanic Helper discovered;
- continue progressing without becoming socially unavailable.

The architectural rule that makes all of this possible is simple:

> Reasoning and observation may be concurrent. Physical control is serialized. Work is incremental and interruptible. New information can always return control to the agent.

That is the behavior to implement.
