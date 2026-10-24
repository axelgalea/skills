---
name: effect-atom-react
description: Data fetching and async state in React with Effect v4 Atoms (`effect/unstable/reactivity`) and `@effect/atom-react`. Use when writing or editing React code that uses Effect v4 atoms — `Atom.make`, `Atom.runtime`, `runtime.atom`, `runtime.fn`, `Atom.family`, `Atom.pull`, `AsyncResult`, `useAtomValue`, `useAtomSet`, `useAtomRefresh`, `useAtomSuspense`, `RegistryProvider` — or when wiring Effect services (`Context.Service` + `Layer`) into React components, fetching data, running mutations, invalidating queries, or paginating with streams.
---

# Effect Atom + React (Effect v4)

Atoms live in core Effect (`effect/unstable/reactivity`). React bindings live in
`@effect/atom-react`. Verified against `effect@4.0.0-rc.109` and
`@effect/atom-react@4.0.0-rc.109`.

```bash
pnpm add effect@rc @effect/atom-react@rc
pnpm add react@^19 scheduler@^0.27   # React 19 peer
```

## Imports

```ts
import { Cause, Context, Effect, Exit, Layer, Stream } from "effect"
import * as AsyncResult from "effect/unstable/reactivity/AsyncResult"
import * as Atom from "effect/unstable/reactivity/Atom"
import {
  RegistryProvider,
  useAtom,
  useAtomRefresh,
  useAtomSet,
  useAtomSuspense,
  useAtomValue,
} from "@effect/atom-react"
```

## Rules

- **Only Effect v4 APIs.** Never import from `@effect-atom/atom-react` or
  `@effect-atom/atom`, never use `Result` (it is `AsyncResult`), never use
  `Effect.Service` or `.Default` (use `Context.Service` + an explicit `layer`).
- Atom modules come from `effect/unstable/reactivity/*`; only hooks and
  `RegistryProvider` come from `@effect/atom-react`.
- Define atoms at **module scope** (or via `Atom.family`). Creating an atom inside a
  component body makes a new atom every render.
- Effect and Stream atoms produce `AsyncResult<A, E>`: `Initial` | `Success` |
  `Failure`, each with a `waiting` flag for in-flight/background work.
- Pass the `AbortSignal` from `Effect.tryPromise({ try: (signal) => ... })` to
  `fetch` so unmount/refresh cancels in-flight requests.
- Prefer services + `Atom.runtime(layer)` over bare `Atom.make(effect)` once more
  than one atom needs the same dependency.

## Core patterns

### Fetch atom

```ts
const usersAtom = Atom.make(
  Effect.tryPromise({
    try: async (signal) => {
      const res = await fetch("/api/users", { signal })
      if (!res.ok) throw new Error(`HTTP ${res.status}`)
      return (await res.json()) as Array<User>
    },
    catch: (cause) => cause as Error,
  }),
)
```

### Render an `AsyncResult`

```tsx
const result = useAtomValue(usersAtom)
const refresh = useAtomRefresh(usersAtom)

AsyncResult.builder(result)
  .onInitial(() => <p>Loading…</p>)
  .onFailure((cause) => <p>Error: {Cause.pretty(cause)}</p>)
  .onSuccess((users, r) => (
    <>
      {r.waiting && <p>Refreshing…</p>}
      <ul>{users.map((u) => <li key={u.id}>{u.name}</li>)}</ul>
    </>
  ))
  .orNull()
```

Builder terminators: `.orNull()`, `.orElse(() => x)`, `.render()`, and
`.exhaustive()` (only available once every case is handled). Other handlers:
`onWaiting`, `onInitialOrWaiting`, `onError`, `onErrorTag`, `onErrorIf`,
`onDefect`, `onInterrupt`. See [references/async-result.md](references/async-result.md).

### Service + runtime (recommended)

```ts
class Users extends Context.Service<Users>()("app/Users", {
  make: Effect.gen(function* () {
    const getAll = Effect.tryPromise({ /* ... */ })
    const findById = (id: string) => Effect.tryPromise({ /* ... */ })
    const create = (name: string) => Effect.tryPromise({ /* ... */ })
    return { getAll, findById, create } as const
  }),
}) {
  // v4 does not generate a layer for you
  static readonly layer = Layer.effect(this, this.make)
}

const runtime = Atom.runtime(Users.layer)

const usersAtom = runtime.atom(
  Effect.gen(function* () {
    const users = yield* Users
    return yield* users.getAll
  }),
)
```

### Parameterized queries — `Atom.family`

```ts
const userAtom = Atom.family((id: string) =>
  runtime.atom(
    Effect.gen(function* () {
      const users = yield* Users
      return yield* users.findById(id)
    }),
  ),
)

// in a component: same id → same atom instance
const result = useAtomValue(userAtom(id))
```

Use primitive keys (string/number). Families are memoized by `Equal`/`Hash`, so
fresh object literals won't hit the cache unless they implement `Equal`.

### Mutations — `runtime.fn`

```ts
const createUserAtom = runtime.fn(
  Effect.fnUntraced(function* (name: string) {
    const users = yield* Users
    return yield* users.create(name)
  }),
  { reactivityKeys: ["users"] }, // invalidates atoms tagged with "users"
)

const createUser = useAtomSet(createUserAtom)                                // fire-and-forget
const createUserP = useAtomSet(createUserAtom, { mode: "promise" })          // Promise<A>, rejects on failure
const createUserExit = useAtomSet(createUserAtom, { mode: "promiseExit" })   // Promise<Exit<A, E>>

const status = useAtomValue(createUserAtom) // AsyncResult of the latest call
```

### Invalidate queries after mutations

Tag the query with `Atom.withReactivity`; any `runtime.fn` with matching
`reactivityKeys` refreshes it on completion.

```ts
const usersAtom = runtime
  .atom(Effect.gen(function* () { return yield* (yield* Users).getAll }))
  .pipe(Atom.withReactivity(["users"]))
```

### Pagination / infinite scroll — `Atom.pull`

```tsx
const pagesAtom = Atom.pull(pagesStream) // runtime.pull(...) when the stream needs services

const [result, pull] = useAtom(pagesAtom)
// Success value: { items: NonEmptyArray<A>, done: boolean }; call pull() to load the next chunk
```

### Suspense

```tsx
const users = useAtomSuspense(usersAtom).value // suspends until Success
```

Options: `{ suspendOnWaiting: true }` to also suspend during refreshes,
`{ includeFailure: true }` to return `Failure` instead of throwing.

### Registry

Hooks use a default registry. Wrap the app in `<RegistryProvider>` for SSR, tests,
or scoped disposal of atom state.

## Hook cheat sheet

| Hook | Returns | Use for |
| --- | --- | --- |
| `useAtomValue(atom)` | value / `AsyncResult` | reading (subscribes) |
| `useAtomRefresh(atom)` | `() => void` | re-running a query |
| `useAtomSet(atom, { mode? })` | setter | writing / triggering `fn` atoms |
| `useAtom(atom)` | `[value, setter]` | read + write (e.g. `Atom.pull`) |
| `useAtomSuspense(atom)` | `AsyncResult.Success` | Suspense-based reads |
| `useAtomMount(atom)` | `void` | keep an atom alive without reading it |
| `useAtomSubscribe(atom, f)` | `void` | side effects on change |

## Mental model

- **Atom** — reactive source of truth.
- **Effect / Stream atom** — async work exposed as `AsyncResult`.
- **`Atom.runtime(layer)`** — wires Effect services into atoms (`.atom`, `.fn`, `.pull`).
- **Hooks** — React bridge via `RegistryContext`.

## References

- [references/examples.md](references/examples.md) — complete, typechecked components
  for every pattern above (fetch, service runtime, family, mutation, invalidation,
  typed errors, pull, Suspense).
- [references/async-result.md](references/async-result.md) — `AsyncResult` states and
  every builder handler.
