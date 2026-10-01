# Anim+ Language — draft 0.1

This document defines intended behavior. “Must” describes a requirement for implementations, not a claim that existing code already supports it. Open questions are listed at the end.

## 1. Files and declarations

A `.anim` file is UTF-8. LF and CRLF are equivalent. Outside quoted strings, `#` starts a comment through the end of the line. Blank lines have no execution meaning.

```text
namespace common
let fadeTime = .25

fade
  find 'panel'
  anim opacity from 0 time fadeTime
```

A single optional `namespace` precedes declarations. Here the full name is `common.fade`; without a namespace, names are unqualified. Names are case-sensitive. Duplicate full names are errors.

An animation name occupies a line at column zero. Instructions follow inside its body. **Every indentation level is exactly two spaces.** Body instructions begin at two spaces. Tabs, odd indentation, and skipped levels are errors. Dedenting must return to an existing level. Indentation creates execution dependencies, not numeric delays.

Strings use single or double quotes; a backslash escapes the active quote or a backslash. Expressions occupy one token without whitespace. One instruction occupies one physical line; multiple instructions on a line are not supported.

Command names, built-in properties, and built-in actions are reserved declaration names. Indented actions require a preceding action parent at a shallower level; selection operations are not temporal parents.

A frontend converts source text into a program description. A visual tool may create the same description directly. Runtime implementations need not include a text parser.

## 2. Variables

```text
namespace common
let duration = .3
let stagger = .04
let distance = 40
let recovery = duration/2

show
  find 'card_*'
  anim pos.y from self-distance time duration delay index*stagger
  anim opacity from 0 time recovery
```

The declaration form is `let name = expression`, at column zero before animation definitions. A variable is a finite numeric binding, immutable throughout one run. It is not mutable state or arbitrary executable code.

Declarations are evaluated in source order. A default may reference earlier variables, not later ones or selection metrics. Duplicate names are errors. `index`, `depth`, `sibling`, `count`, and `self` are reserved and cannot be declared.

An implementation may accept overrides of declared variables when preparing a run. Overrides are finite numbers; undeclared override names are errors. Resolve each binding using its override, if provided, otherwise its expression; subsequent declarations see the resolved value. Values remain fixed until a new run is prepared.

Bindings belong to the source file. A named animation call uses the called file's bindings and does not mutate the caller's bindings.

An ordinary variable in an endpoint is absolute: `from distance` means the value of distance, not an offset. Relative values use expressions such as `from self-distance` or `to self+distance`. The signed-literal shorthand applies only to a single numeric literal, not to arbitrary variables.

Only numeric variables are part of draft 0.1. String bindings, assignments, local scopes, and runtime parameter expressions are not defined.

## 3. Selections and sections

A run receives a root object. Preparation performs selection once, producing stable ordered collections. Runtime does not search the scene every frame. Object references must not prevent destruction indefinitely.

A chain of selection operations belongs to one section. After an action, the next selection operation begins a new section from the original root:

```text
window_open
  find 'title'
  anim pos.y from -40 time .3

  find 'avatar_*'
  grid center
  anim scale from .2 time .4 delay index*.04
```

These independent sections start together. Blank lines do not define sections. Each action retains its own section's collection rather than consulting a mutable global collection during playback.

No explicit selection means no property transitions. A called animation may inherit an explicit collection. An empty result never falls back to the entire scene.

| Operation | Meaning |
|---|---|
| `find name` | Search the subtrees of current roots, including those roots |
| `depth n` | Collect descendants up to n levels; exclude current roots |
| `depth-only n` | Collect exactly n levels; n=0 selects current roots |
| `filter name` | Retain matching collection members |
| `ignore name` | Remove matching collection members |
| `of-type category` | Alias for filter type:category |
| `index 0,2,4` | Select immediate children by zero-based indices |
| `path 0/3/1` | Follow zero-based child indices |
| `parent` | Replace members with immediate parents within the target tree |
| `reverse` | Reverse collection order |
| `grid origin [x\|y]` | Recompute the index metric from geometry, without selecting objects |
| `find name n`, `path path n` | Append exactly one object from the section root with depth=n |

Initial traversal starts at the target root. Initial filter/ignore/of-type first collect the entire subtree, including its root. Reverse and grid require an established collection.

Traversal is depth-first in the adapter's child order. Deduplication preserves the first position. Traversal depth is local to the current roots; the expression metric depth remains relative to the section's original root unless manually assigned.

Only `*` is a wildcard, matching any substring including the empty string. Literal names match completely and case-sensitively. A dot and question mark are literal characters.

Find/filter/ignore accept `type:category`. Core categories are sprite, label, clickable, layout, and container. These are adapter predicates, not mandatory class names; categories may overlap. Spine is an extension category. Unknown categories are errors.

Manual append must find exactly one object. Re-appending the same object with a different manual depth is an error. Before grid, its index follows append order.

## 4. Parallel actions and Pipe

```text
phases
  find 'ball'
  anim pos.y to -160 time .4
    anim pos.y from -160 time .4
      anim event 'landed'
  anim pos.x to +260 time 2.4
```

X and the first Y action activate together. The second Y waits for the first; the event waits for the second. An action depends on the nearest preceding action at a shallower indentation level. Same-level actions are parallel. Selection does not consume time or become a predecessor.

A parent producing transitions for several objects completes its **group barrier** after all of them finish, including their delays and finite repeats. A separate queue per property or per object is not equivalent to this rule.

Several children of one parent activate together. They wait for the parent action, not for all descendants in its branch. An empty transition group completes immediately at activation. Dependencies may cross selection sections; composing does not reset temporal predecessors.

For repeat=-1, the parent barrier is released after the first cycle on every target; looping transitions remain active until cancellation. Cancellation, failure, replacement, and target destruction are not successful completion and must not trigger dependent actions.

## 5. Property transitions

```text
anim property [from expression] [to expression] [time expression] [delay expression] [repeat n] [easy.curve]
```

| Property | Value |
|---|---|
| pos.x, pos.y, pos.z | Position in logical scene units |
| opacity | Opacity 0..1 |
| scale, scale.x, scale.y | Scale multiplier; scalar scale affects both axes |
| rot | Rotation in degrees |
| skew.x, skew.y | Shear in degrees |

Adapters declare coordinate axes, units, channel mapping, and layout interactions. Equal source does not guarantee identical screen movement under different coordinate conventions. Unsupported properties are errors.

Defaults: from=self, to=self, time=.25 seconds, delay=0, repeat=0, easy.linear.

All affected baseline properties are captured before any writes. Self is stable across phases. From self to self has no visual effect.

A single signed literal in from/to is relative: -40 means self-40; +40 means self+40. An unsigned 40 is absolute. Use 0-40 for negative absolute values. Signs in time/delay are ordinary arithmetic signs.

From is applied at activation, before delay. A waiting child does not write From early. After delay, the value interpolates from From to To. A completed transition leaves To in place.

Repeat=2 means three cycles. Delay applies only before the first cycle. Each repeat restarts from From; there is no implicit yoyo. Time=0 writes To immediately after delay. Infinite repeat with time=0 is invalid.

Overlapping writes to the same channel of the same object are errors in core 0.1. Sequential Pipe phases are allowed. Whole scale conflicts with scale.x/scale.y; independent X and Y are allowed. Unknown asynchronous intervals require runtime validation. An implementation must not silently substitute an overwrite or additive policy.

## 6. Expressions, easing, and waves

Expressions support decimal numbers with a dot, + - * /, parentheses, min(...), max(...), declared variables, and:

- index: zero-based order; a fractional distance after grid;
- depth: object depth, including manual overrides;
- sibling: zero-based child index within its parent;
- count: the size of the prepared collection;
- self: the baseline value of the current property.

Examples: delay index*.05, time max(.1,1/count), from self-40. Values are evaluated from the run's prepared context. Division by zero, NaN/Infinity, and negative time/delay are errors before side effects.

Easing families: linear; sin, quad, cubic, quart, quint, expo, circ, elastic, back, bounce with -in/-out/-in-out. Easy.bezier(x1,y1,x2,y2) requires X in 0..1 and permits unrestricted Y for overshoot. Exact formulas/constants need reference vectors before stable conformance. A matching curve name alone is insufficient. Spring is an extension.

Grid forms rows/columns from the collection's minimum-depth layer. Deeper members inherit their selected upper-layer ancestor's cell; otherwise they use the nearest cell. With no axis, index is Euclidean cell distance (diagonal sqrt(2)); x/y limits distance to one axis.

Origins: start, end, center, edges, an object name, or a path. Center is the grid's geometric center. Edges measures distance to the nearest edge. An external origin object maps to its nearest cell. Reverse after grid preserves grid indices. Grid does not sort the collection. Invalid geometry is diagnosed, not replaced by traversal indices.

## 7. Events, extensions, and reuse

Anim event 'ready' delay .1 time .2 emits one event after delay. Time defines the barrier after emission, not handler duration. Repeat is invalid for events. Events need no selection. Object-specific metrics are unavailable unless an implementation defines an explicit single-target context; count can be zero.

Sound and Spine are extensions:

- anim sound 'open': one trigger per action, not per object; time defines an explicit barrier.
- anim spine 'show' 0: an action on selected objects. Without time, the adapter supplies completion; with time, use the explicit barrier. Interrupted is not Completed.

```text
namespace common

pop
  anim scale from .2 time .3 easy.back-out

cards
  find 'card_*'
  anim pop
```

Anim pop calls another definition with the current collection. Unqualified names resolve within the current namespace; a full name is explicit. Built-in property/action names are reserved. Local selection starts from each passed root. Children of the call wait for the called animation's terminal branches. Recursive calls and missing definitions are preparation errors.

## 8. Time and seeking

For a finite transition: start=activation+delay; end=start+time*(repeat+1). A child's activation is the maximum completion time of its parent group. Program duration is the maximum terminal action end, not the sum of source lines.

An infinite program has no finite duration. A loop's first-cycle barrier allows continuation but is not the loop's lifetime. Unknown asynchronous duration remains unknown; a preview horizon is not a claimed completion time.

Seek is an optional capability. Evaluation begins from baseline, applies completed phases, and evaluates active transitions. Seeking repeatedly to the same time produces the same pose. Events and sound are suppressed during seek. Without an evaluator for an extension, preview reports the limitation rather than fabricating a correct pose.

## 9. Open questions before stable 1.0

- Complete EBNF, numeric ranges, identifiers, and strict escapes.
- Reference easing vectors and repeat-boundary conventions.
- Grid clustering, irregular layouts, nearest-cell ties, and edges definition.
- Partial target-loss policy and scene boundaries.
- Portable program schema and a source version marker.
- Extension registration without name-resolution ambiguity.

Until these are resolved, implementations must not claim complete stable conformance.
