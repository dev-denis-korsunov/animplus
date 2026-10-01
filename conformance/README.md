# Conformance requirements

These are required scenarios, not an implemented certification suite. Machine-readable fixtures and a harness still need to be created.

A fixture contains source, initial object identities/names/categories/properties/geometry, variable overrides, observation times, expected values, events, and terminal statuses. A mock scene verifies language behavior independently of any implementation platform. Adapters additionally require real-object tests.

## Core scenarios

| Scenario | Expected behavior |
|---|---|
| Comments | Full-line and inline // comments are ignored; quoted // stays literal; single / works in expressions and paths; # is not a comment marker |
| Variables | Finite numeric declarations, earlier-binding defaults, immutable run values |
| Overrides | Declared numeric overrides resolve before dependent defaults |
| Variable errors | Duplicate/reserved/undefined names, forward references, and non-finite values fail |
| Indentation | Exactly two-space levels; reject tabs, odd levels, and skipped levels |
| Wildcard | Case-sensitive full matching; only * is special |
| Traversal | Find panel then depth 1 selects its children |
| No selection | No property transitions are created |
| Empty selection | A failed find never falls back to all objects |
| Sections | Selection after actions restarts at root; independent sections are parallel |
| Manual collection | Append order controls index; manual depth is retained |
| Baseline | Capture self before all writes; self-to-self is a no-op |
| Relative | Signed literals are relative; variables are absolute unless explicitly using self |
| Parallel | Independent channels update together |
| Pipe | A child property waits for a different parent property |
| Group barrier | Children wait for all parent targets, including delays/repeats |
| Fork | Same-depth children of one parent activate together |
| Repeat | repeat=2 means three cycles; delay only precedes the first |
| Loop | First-cycle barrier releases children without stopping the loop |
| Zero time | To is applied after delay; long chains do not overflow the stack |
| Event | One event per action, not per target |
| Cancel | One owner's cancellation preserves another owner and suppresses dependent events |
| Failure | Failed/Replaced/TargetDestroyed are not Completed |
| Conflict | Whole-vector conflicts with components; independent X/Y do not |
| Duration | Maximum end, not sum of lines |
| Named call | Inherited collection; missing references/recursion fail |
| Extension syntax | Direct event/sound/spine commands; anim-prefixed forms are errors |
| Extension registration | Unknown commands and reserved/duplicate registrations fail before side effects |
| Extension barrier | Synchronous and asynchronous success release children exactly once; target groups wait for all members |
| Extension interruption | Failure, replacement, cancellation, and target loss suppress dependent actions |
| Late completion | Duplicate callbacks and completion after cancellation never resume the chain |
| Extension timing | Unknown duration is not zero; explicit time follows the declared barrier policy |

Numerical fixture: opacity 0→1, linear easing, time=1, delay=.2. At t=0 and .2, opacity is 0; at .7 it is .5; at 1.2 it is 1. A zero-delay child activates at 1.2. Fixtures specify numeric tolerances.

## Optional capabilities

- Grid: diagonal sqrt(2), center/edges, axes, ancestor cells, missing geometry.
- Easing: reference samples, endpoints, overshoot, and cubic Bezier.
- Seek: .1→.8→.1 gives identical poses; future children do not apply From; no events.
- Bounce: independent X motion, sequential Y phases, squash/stretch, and no cumulative baseline drift.
- Async: unknown duration holds Pipe; interruption cancels dependent work; synchronous completion cannot overflow a long chain.
- Extension seek: suppress external effects; require a pose evaluator for visual actions; report unsupported/unknown preview instead of guessing.

Boundary fixtures distinguish the state before and after processing actions at time t. Do not compare arbitrary frame indices between implementations without aligned clocks and ordering rules.

Every semantic change requires updated scenarios. Before stable release, turn these requirements into portable fixtures and run them in each implementation's CI.
