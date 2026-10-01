# Extensions — custom actions

An extension adds an executable command to Anim+. It must preserve the execution contract: a custom action can be a Pipe parent or child just like a property transition. An implementation may supply extensions itself or allow applications to register them. These example profiles do not require every implementation to provide their underlying services.

## Syntax and registration

```text
show_chest
  find 'chest'
  spine 'open' 0
    event 'chest_opened'
  sound 'open.window.ui'
```

Spine and sound activate together. The event waits for Spine's successful completion, not for sound. Each command is standalone: **do not prefix custom calls with `anim`**. Property transitions and named animation references continue to use `anim`.

Register commands before preparing a program. A registration declares:

- A unique case-sensitive command name and extension version. Core commands, properties, and declaration keywords cannot be shadowed. Registered command names are reserved animation/variable declaration names in that program. Duplicate registrations fail.
- An argument schema, defaults, validation, and supported common options. There is no implicit support for `repeat`, `time`, or property options.
- Whether it runs once per action or once per selected object, plus required categories and context metrics. A per-object action with an empty collection completes immediately without invoking handlers; a once-per-action operation may run without a selection.
- Its timing policy: immediate, explicit duration, or actual asynchronous completion. Unknown duration remains unknown, never zero.
- The affected channels/resources and conflict rules, owner/cancellation behavior, and any deterministic preview evaluator.

Unknown commands, unsupported options/categories, invalid expressions, and missing capabilities fail during preparation, before side effects. Bare identifiers never implicitly call animation definitions: those references use `anim name`. Arguments follow the normal quoting, variables, expressions, comments, and two-space indentation rules.

## Execution handle and Pipe

Every activation supplies the extension with its run owner, activation identity, cancellation signal, prepared arguments, and declared collection/context. It returns a handle even if it completes synchronously. This is a behavioral contract, not a prescribed API or implementation language.

The handle must expose cancellation and terminal outcome: `Completed`, `Cancelled`, `Failed`, `Replaced`, or `TargetDestroyed`. It reports success at most once. Only successful completion releases dependent actions. Returning from a callback, issuing a trigger, or resolving an unrelated asynchronous operation is not automatically completion unless the registered policy explicitly defines it that way.

For per-object operations, the action's group barrier waits for every member. In draft 0.1, an unsuccessful member fails or cancels that action and its continuations; it cannot be silently removed from the barrier. Same-level children of a successful parent activate together. Completion is queued through the scheduler, including synchronous results, so long immediate chains do not grow the call stack.

Cancellation is owner-scoped, idempotent, and prevents not-yet-started effects. Cancel owned work, detach listeners, release retained resources, and invalidate activation callbacks. A late or duplicate completion callback must not emit events, write values, or resume a cancelled/replaced chain. Exceptions and rejected asynchronous operations become `Failed`, not `Completed`.

An extension may outlive its Pipe barrier only if its declared policy distinguishes **barrier completion** from **resource lifetime**. The run keeps that resource owned and cancellable until it ends. A released barrier is not permission to forget an active operation or claim the whole run has finished. Independent lifetime failures do not retroactively undo children already released, but are reported and stop remaining dependent work.

## Example: event

```text
notify
  event 'ready' delay .1 time .2
    event 'after_ready'
```

`event` emits one named notification per action after `delay` (default 0), without requiring a selection. `time` defaults to 0 and defines the barrier interval after emission; it does not wait for asynchronous subscribers. Above, ready is emitted at .1 and its child at .3. Subscriber errors fail the action. Asynchronous subscriber completion needs a separately registered action with an asynchronous handle, not an implicit change to event semantics.

Repeat is invalid. File variables and `count` are available; per-object metrics and `self` are unavailable. Seek must not emit notifications.

## Example: sound

```text
play_feedback
  sound 'open.window.ui' time .2
    event 'feedback_interval_finished'
```

`sound` issues one named trigger per action after `delay` (default 0), without requiring a selection. `time` defaults to 0 and specifies the Pipe barrier after triggering, **not** the audio clip's duration. Repeat is invalid. The context matches event's once-per-action context.

If playback continues beyond the barrier, retain an owner-scoped playback handle and track its lifetime separately. Cancellation stops only owned playback. A service unable to provide ownership/cancellation/lifetime guarantees must declare this profile unsupported rather than silently leaking playback. Seek must not trigger sound.

## Example: spine

```text
open_chest
  find type:spine
  spine 'open' 0
    event 'opened'
```

`spine '<animation>' [track]` runs on the selected Spine objects; track defaults to 0 and must be a non-negative integer. `delay` defaults to 0. Without `time`, each handle completes from the actual animation completion signal. A duration that cannot be known during preparation stays unknown; Pipe still waits live. Replacing or interrupting playback is not success.

With an explicit non-negative `time`, the barrier uses that interval after playback starts. Any playback still active beyond it remains owned and cancellable with a separately tracked lifetime. This option must be explicitly supported by the registration; a timer cannot erase interruption/failure before the barrier. This example profile does not define repeat/loop options; another version may add them with explicit first-cycle barrier and lifetime rules.

The group waits for all selected objects. Missing animations or unsupported targets are diagnosed, not skipped. A preview requires an evaluator that can apply the requested animation's pose without playback side effects.

## Debugging and duration

Known barrier times contribute to dependency timing. Unknown barriers propagate unknown activation times to their children; tools may show estimates as estimates, never as guaranteed end times. Program duration includes resource lifetime, not just the last released barrier.

Preparation and seeking cannot invoke live extension side effects. A deterministic visual evaluator may support seeking from baseline. If the evaluator or asynchronous timing is unavailable, report the unsupported preview region instead of pretending the action completed or fabricating a pose.
