# Anim+

**An implementation-independent language for procedural animation.**

Anim+ describes **which objects to select**, **what to animate**, and **what should happen after a transition completes**. Source files use the `.anim` extension; the technical project name is `animplus`.

```text
namespace common
let revealTime = .2
let waveDelay = .05

reveal_cards
  find 'card_*'
  grid center
  anim scale from .2 time .3 delay index*waveDelay easy.back-out
  anim opacity from 0 time revealTime delay index*waveDelay
    event 'cards_visible'
```

Scale and opacity start together, spreading out from the center. The event runs once after **the entire opacity group**, including its per-object delays, finishes. It does not wait for scale: indentation identifies the specific predecessor.

## Status

**Draft 0.1: a proposed standard, not a released implementation.** This repository contains the language contract, examples, and conformance requirements. It does not contain a runtime, parser, published installation packages, or executable conformance harness.

Examples describe intended behavior, not verified support by every implementation. Compatibility with earlier animation formats is not a goal.

## Core rules

- Selection produces an ordered collection of objects and their `index`, `depth`, `sibling`, and `count` metrics.
- Property actions create a transition for each selected object.
- Actions at the same indentation level start in parallel.
- An indented action uses **Pipe**: it starts after its nearest preceding action at a shallower level completes.
- **One indentation level is exactly two spaces.** Tabs and skipped levels are errors.
- A new selection operation after actions starts a new section from the original root.
- No selection, or an empty collection, means no property transitions. The root is never selected implicitly.
- `self` is the original property value captured before any writes in a run.
- File-level `let` declarations provide reusable numeric variables.
- A timeline is a control/debugging projection, not a mandatory runtime structure.

## Example

```text
namespace demo
let travel = 40

show_title
  find 'title'
  anim opacity from 0 time .2
  anim pos.y from self-travel time .3 easy.back-out
    event 'title_arrived'
```

An omitted `to` resolves to `self`. A signed literal in `from`/`to` is relative: `from -40` means `from self-40`, while `from 40` is absolute. A variable is an ordinary expression: use `self+travel` for a relative variable value.

## Documentation

- [Language and execution semantics](docs/language.md)
- [Extensions and custom action contract](docs/extensions.md)
- [Implementation contracts and distribution](docs/runtime-and-adapters.md)
- [Conformance requirements](conformance/README.md)
- [Reusable examples](examples/common.anim)
- [Ball bounce](examples/bounce.anim)

## Repository boundaries

This repository defines **the standard**. Implementations live separately and may share frontend, runtime, adapter, and tooling components. The standard does not prescribe implementation languages or target platforms.

An implementation reports the standard version it supports, its capabilities/extensions, and its conformance results. Unsupported operations must produce diagnostics instead of silently changing the meaning of a program.

The standard version and an implementation's package version are independent. Standard changes require an updated contract, example, and conformance scenario. Git is the change history; no implementation journal is maintained.

## Licensing

A license has not been selected. Until one is added, do not assume permission for unrestricted redistribution or inclusion in other products.
