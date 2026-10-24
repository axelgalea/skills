# AsyncResult

`import * as AsyncResult from "effect/unstable/reactivity/AsyncResult"`

Every Effect/Stream atom (`Atom.make(effect)`, `runtime.atom`, `runtime.fn`,
`Atom.pull`) holds an `AsyncResult<A, E>`.

## States

| `_tag` | Fields | Meaning |
| --- | --- | --- |
| `"Initial"` | `waiting` | No value yet. `waiting: true` while the first run is in flight. |
| `"Success"` | `value`, `waiting`, `timestamp` | Latest value. `waiting: true` during a background refresh. |
| `"Failure"` | `cause: Cause<E>`, `previousSuccess: Option<Success<A, E>>`, `waiting` | Failed. `previousSuccess` keeps the last good result, if any. |

`waiting` lives on every state, so `result.waiting` works as a spinner flag
without narrowing first.

## Builder

`AsyncResult.builder(result)` returns a chainable, type-narrowing matcher. Each
handler removes its case from the remaining type; handlers that no longer apply
disappear from the API.

| Method | Handles |
| --- | --- |
| `onInitial((initial) => …)` | `Initial` |
| `onInitialOrWaiting((result) => …)` | `Initial`, or any state with `waiting: true` |
| `onWaiting((result) => …)` | any state with `waiting: true` (does not narrow) |
| `onSuccess((value, success) => …)` | `Success` — second arg exposes `waiting`, `timestamp` |
| `onFailure((cause, failure) => …)` | every `Failure` (typed errors, defects, interrupts) |
| `onError((error, failure) => …)` | typed failures `E` |
| `onErrorTag("Tag", (error, failure) => …)` | typed failures with `_tag === "Tag"`; also accepts an array of tags |
| `onErrorIf(predicate, (error, failure) => …)` | typed failures matching a predicate/refinement |
| `onDefect((defect, failure) => …)` | defects (unexpected throws) |
| `onInterrupt((interruptors, failure) => …)` | interruptions |

Terminators:

| Method | Returns |
| --- | --- |
| `.orNull()` | handled output, or `null` for unhandled cases |
| `.orElse(() => fallback)` | handled output, or the fallback |
| `.render()` | handled output; `null` only if `Initial`/`Success` is unhandled |
| `.exhaustive()` | handled output; only exists once every case is handled |

Handler order matters for overlapping handlers: the first matching handler wins.
Put `onWaiting` first if a refresh should replace the content, or read
`r.waiting` inside `onSuccess` to overlay a refresh indicator instead.

## Matching without the builder

```ts
AsyncResult.match(result, {
  onInitial: () => "loading",
  onFailure: (failure) => `error: ${failure.cause}`,
  onSuccess: (success) => `got ${success.value}`,
})

AsyncResult.matchWithError(result, {
  onInitial: () => "loading",
  onError: (error) => `typed error: ${String(error)}`,
  onDefect: (defect) => `defect: ${String(defect)}`,
  onSuccess: (success) => `got ${success.value}`,
})
```

`AsyncResult.matchWithWaiting` is the same as `matchWithError` but replaces
`onInitial` with `onWaiting`, called for `Initial` or any `waiting` state.
