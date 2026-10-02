# Visual Game Agent: Learning and Automation Plan

Date: October 1, 2026

## Goal

Build a Linux-compatible agent that can be placed in a game, given an automation goal, and learn how to accomplish it through screenshots, ordinary keyboard and mouse inputs, experimentation, and visible feedback. Wurm is the initial proving ground; the architecture should support other games through separate profiles.

The agent should automate the work previously done by the human bot developer: inspect the interface, discover procedures, capture useful images, write deterministic automation, test it, repair it, and retain the results.

Experience must produce both persistent knowledge and persistent executable skills. The model develops and maintains the automation framework, while reliable routines handle familiar tasks cheaply.

## Interaction boundary

The game-facing interface consists of screen pixels and normal desktop input. Do not modify the game process, inject code, read process memory, call internal game methods, intercept protocol traffic, or use client/server source as a gameplay oracle. Existing source-assisted Wurm agents illustrate the learning pattern, but this project tests a visual-only implementation.

Local automation code, screenshots, configuration, and memory are available to the agent. Visible game text may be interpreted from screenshots. The initial platform is Linux Mint with X11; AutoHotkey is a source of workflow ideas rather than a runtime dependency.

## Learning loop

1. Read the goal, relevant memory, current checkpoint, and available skills.
2. Capture a fresh screenshot and validate capture quality and window geometry.
3. Identify the visible state, possible actions, uncertainties, and evidence of progress.
4. Reuse a suitable skill or formulate a small experiment with an expected visible result.
5. Send bounded keyboard or mouse inputs.
6. Capture the resulting state and compare it with the prediction.
7. Record the observation, failure, or confirmed result.
8. Convert a repeatable discovery into a deterministic skill with visual checkpoints.
9. Test and revise that skill before relying on it for repeated work.

An input sent successfully is not evidence that the intended game action succeeded. Completion requires a visible result tied to the goal.

## Architecture

| Component | Responsibility |
|---|---|
| Capture adapter | Capture full frames and crops, detect bad captures, retry and use fallback capture methods. |
| Perception | Model visual interpretation, template matching, and optional screenshot OCR; return targets and evidence. |
| Input adapter | Mouse movement, clicks, key presses, key release, and bounded holds on Linux. |
| Planner | Interpret goals, select skills, design experiments, and handle unexpected states. |
| Skill author | Create or revise visual templates and declarative procedures. |
| Skill executor | Run deterministic procedures with conditions, deadlines, branches, and bounded repetition. |
| Verifier | Compare observed results against explicit success and failure conditions. |
| Memory manager | Maintain gameplay knowledge, operating documentation, journal, and restart checkpoints. |
| Supervisor | Enforce focus checks, stop controls, resource limits, and action ownership. |

Use Python as the initial implementation language. Keep capture, perception, input, and model access behind adapters so they can be changed independently. Treat Wayland support as a separate adapter project with its own capture/input permissions and verification.

## Autonomous screenshots and recognition repair

The agent must be able to request a new screenshot whenever its observation is stale or incomplete. It can inspect an enlarged crop and create a template directly from its own captured frame.

Store template provenance: screenshot ID, original crop bounds, game profile, window size, UI scale, creation time, and intended meaning. Crops displayed at a different size must map back to original screen coordinates before input.

On failure, distinguish capture failure from recognition failure and gameplay failure:

- Blank, stale, or unavailable capture: retry, switch capture backend, and stop input if no valid observation can be obtained.
- Missing or ambiguous target: take a fresh frame, inspect another region, adjust the region or scale, or create a replacement template.
- Target recognized but expected result absent: investigate focus, timing, menu state, prerequisites, and whether the interpretation was wrong.
- Window moved or resized: recalibrate coordinates and invalidate assumptions tied to old geometry.

Never convert a low-confidence match into an unconditional click. Avoid retrying the same unsuccessful action indefinitely.

## Learned deterministic skills

Start with a constrained declarative format supporting mouse/key actions, visual conditions, bounded waits, branches, skill calls, and bounded repeats. This supplies AutoHotkey-equivalent workflows without requiring AutoHotkey on Linux.

Each skill records:

- Stable ID, revision, description, and game/profile compatibility.
- Preconditions and required templates.
- Parameters and permitted ranges.
- Ordered steps, visual checkpoints, timeouts, and recovery outcomes.
- Expected success condition and known failure states.
- Evidence of trials, successful runs, failed runs, and last verification.

Lifecycle: draft → tested → reusable. A failure can move a reusable skill to needs_revision. Preserve earlier revisions so repairs can be rolled back.

Prefer anchors and relative positions over absolute coordinates. A deterministic routine remains conditional: it checks the current screen before assuming that the next action is appropriate.

Example: identify an inventory item, open its context menu, recognize an action, select it, wait for a visible completion indicator, and verify the result. If the menu differs or the action fails, return control to the planner with a fresh screenshot and the failed checkpoint.

Generated Python extensions can be a later capability when the declarative language proves insufficient. They need a controlled interface, validation, isolated testing, and explicit versioning. The initial v2 foundation uses declarative skills rather than arbitrary generated code execution.

## Persistent memory

| Artifact | Contents and update policy |
|---|---|
| `WURM_KNOWLEDGE.md` | Learned mechanics, prerequisites, visual cues, hypotheses, confidence, and corrections. Curated knowledge for Wurm. |
| `BOT.md` | How to operate the framework; capabilities, skill/template usage, limitations, and recovery procedures. |
| `JOURNAL.md` | Append-only timestamped record of experiments, issues, expected/observed results, solutions, and discoveries. |
| `skills/` | Machine-readable skill definitions, revisions, and verification status. |
| `templates/` | Visual reference images with provenance and compatibility metadata. |
| `state.json` | Current goal, checkpoint, pending action, expected outcome, and last observation. |
| `evidence/` | Selected screenshots and structured trial records supporting important findings. |

For another game, use its own knowledge file and profile. Separate general automation knowledge from game-specific facts.

Label hypotheses as hypotheses. Record contradictory observations and mark superseded facts rather than silently preserving incorrect conclusions. Every decision receives relevant memory and recent journal context; full files remain accessible without being inserted in full into every model request.

Save a pending-action checkpoint before sending input and save the observed outcome afterward. After a crash, inspect the game before repeating an action: the input may already have happened. Write checkpoints atomically and preserve the journal across restarts.

## Recovery and control

Provide preview mode, explicit live-input mode, an immediate stop mechanism, and reliable release of held keys. Suspend input when the target window loses focus. Bound actions, waits, skill depth, loop counts, and total execution time.

On a failed skill, retain the failed step, screenshot evidence, and attempted inputs. Return to observation and planning; revise the routine or its prerequisites instead of hiding the failure with endless retries.

Use one input owner per game window. Multi-agent coordination is a later extension: agents may share findings and propose skill revisions, but simultaneous keyboard/mouse control of one window must be prevented. Starting additional game instances or agents should remain within a configured host resource budget and user scope.

## Wurm proving ground

Begin with short tasks that have clear visual outcomes: recognize an inventory item, open a menu, select an action, operate a toolbelt slot, and detect action completion. Then test repetition and interruptions.

Use more difficult tasks to expose assumptions: changed menus, failed crafting attempts, inventory movement, horse feeding, and differing movement behavior when mounted or driving a cart. These are test scenarios, not claims that the visual agent already solves them.

Measure whether the agent can discover a procedure, retain it, reuse it after restart, and repair it when the interface changes. Progress should be demonstrated by saved artifacts and observed results.

## Implementation milestones

### 1. Establish the Linux control loop

Reliable capture, foreground-window detection, coordinate mapping, bounded input, preview/live modes, stop behavior, and before/after observations. Exit criterion: complete a simple user-directed interaction and show visible evidence of its outcome.

### 2. Make perception repairable

Fresh-frame requests, crop inspection, autonomous template creation, capture fallback, and geometry recalibration. Exit criterion: recover from a missing template or moved window without blindly repeating a click.

### 3. Turn discoveries into skills

Skill definition, execution, conditions, waits, branches, bounded loops, revisions, and verification records. Exit criterion: discover and save a short procedure, then repeat it through the executor without model reasoning for each input.

### 4. Persist learning and resume

Integrate all three Markdown memory files, structured skill evidence, and atomic checkpoints. Exit criterion: restart mid-task, inspect the actual screen, and resume without unnecessarily repeating a potentially completed action.

### 5. Demonstrate repair through experience

Introduce a menu/layout variation or failed action. Require the agent to diagnose it, revise its template or routine, test the repair, and document the result. Exit criterion: the corrected revision succeeds and the old failure remains in the journal.

### 6. Generalize beyond Wurm

Move game identity, bindings, window selection, and knowledge into profiles. Test a second game using the same capture/input/skill/memory interfaces. Exit criterion: new game procedures are learned through pixels and input without adding a privileged game API.

## Evaluation

Track task success rate, skill reuse success, intervention count, recovery success, model calls per completed task, execution time, and persistence across restarts. Compare a familiar task executed by a saved skill with the same task under model-driven step-by-step control.

Test changed resolution/UI scale, obstructed targets, unexpected dialogs, delayed results, focus loss, and interrupted execution. Promotion to reusable status should depend on observed trials in the intended conditions; one visual assessment is preliminary evidence rather than a guarantee.

## Current foundation versus intended outcome

The existing v2 operating documentation describes Linux X11 input, autonomous screenshot/crop requests, template authoring, declarative skills, visual assessment, capture recovery, and the three memory files. Those are a starting framework, not proof of general-purpose autonomous gameplay. This plan does not certify the implementation through a live game test.

The intended outcome is an agent that accumulates practical capabilities: successful experience becomes documented knowledge and reusable automation; failures become evidence for repair. This is capability improvement through tools and memory, without implying that the underlying model trains itself or increases its intrinsic intelligence.
