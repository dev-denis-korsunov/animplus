# Implementation contracts and distribution

## Responsibilities

1. **Frontend:** reads .anim, validates names/expressions/two-space indentation, and creates a program. A visual tool may build the same program without source text.
2. **Preparation:** resolves variables, composing, collections, baselines, transitions, and Pipe dependencies. Validates capabilities before side effects.
3. **Runtime:** activates actions and continuations, updates properties or delegates transitions to an execution backend. No mandatory editor, timeline tracks, pose cache, or parser.
4. **Debug:** projects prepared requests onto real-object tracks and evaluates poses for seeking.

Preparation does not emit events/sound. A runtime may use an existing transition system but must preserve the language contract.

## Scene adapter

The adapter provides:

- stable object identity and lifetime-aware references;
- a root, parents, and ordered children;
- names and category predicates;
- logical property/channel reads and writes;
- shared conflict identity for whole-vector and component writes;
- post-layout geometry for grid;
- documented axes, units, scene boundaries, and layout-controlled properties.

Core semantics must not depend on a platform's native selectors or wildcard rules. Unsupported properties/categories are diagnosed.

## Execution backend

| Operation | Responsibility |
|---|---|
| create transition | Target, channels, endpoints, timing, and owner |
| pipe | Explicit predecessor group handles |
| create action | One-shot or asynchronous operation with completion |
| cancel run | Cancel only this owner's work |
| status | Pending, Active, Completed, Cancelled, Failed, Replaced, TargetDestroyed |
| release run | Release retained handles/callbacks |

An owner identifies one run, not the target object. Multiple runs may affect independent channels of one object. Failure/replacement/cancellation must not trigger a continuation as if it completed.

Pipe works across properties. A parent group may have several parallel children. A property-name queue is not a sufficient implementation.

Process zero-duration chains iteratively, without recursive stack growth. Host time uses seconds. Completion reports one terminal result.

## Capabilities and extensions

An implementation reports its supported standard version, properties, categories, easing, grid, seek, and extensions. Unsupported features are errors before side effects.

Unknown asynchronous durations may be supported live: Pipe waits for actual completion. Debug displays unknown timing and restricts seeking rather than guessing.

Stop cancels the run. Reset additionally restores its baseline when requested. Restoration must not overwrite channels now owned by a different run.

## Distribution

The standard is a development contract, not a required runtime dependency. Each implementation consumes a pinned standard revision and shared conformance fixtures.

Distribute frontend, core runtime, adapters, and debug tools separately when their dependencies differ. Installing one implementation must not require unrelated implementations or authoring tools. Allow ready-to-use packages and source integration where appropriate.

Shared presets can be distributed as a separate .anim collection. Application-specific animation files belong to the consuming application.

Standard and implementation versions are independent. Each implementation release declares supported contract version, capabilities, limitations, and conformance results. A standard update does not require simultaneous implementation releases.

## Initial implementation slice

First validate a reference implementation against a simple object tree: variables, find/depth, scalar/component properties, baseline, linear easing, parallel/Pipe, event, and cancellation. Other implementations pass the same fixtures. Grid, full easing, extensions, and editors follow after their semantics are pinned down.
