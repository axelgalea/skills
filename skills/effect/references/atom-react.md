# Effect Atom + React

Read this when you write React code with Effect atoms: `Atom.make`, `Atom.runtime`, `runtime.atom`, `runtime.fn`, `Atom.family`, `Atom.pull`, `AsyncResult`, and the `@effect/atom-react` hooks.

Atoms live in core Effect (`effect/reactivity`). React bindings live in `@effect/atom-react`, which releases with the same version number as `effect`. Its peer dependencies are React 19 and `scheduler`.

```ts
import { AsyncResult, Atom } from "effect/reactivity"
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

- Never import from `@effect-atom/atom`, `@effect-atom/atom-react`, or `effect/unstable/reactivity`.
- Atom state is `AsyncResult`, not `Result`.
- Only hooks and `RegistryProvider` come from `@effect/atom-react`.
- Define atoms at module scope or through `Atom.family`. An atom created in a component body is a new atom on every render.
- Keep services in their own modules (see [services-layers.md](services-layers.md)) and atoms in a separate module that imports them.
- Prefer `Atom.runtime(layer)` over bare `Atom.make(effect)` once more than one atom needs the same service.
- Pass the `AbortSignal` from `Effect.tryPromise` to `fetch` so unmount and refresh cancel requests.
- Decode response bodies with a schema. Do not cast `res.json()`.

## Fetch atom

```ts
const User = Schema.Struct({ id: Schema.Finite, name: Schema.String })

const usersAtom = Atom.make(
  Effect.tryPromise(async (signal): Promise<unknown> => {
    const res = await fetch("/api/users", { signal })
    if (!res.ok) throw new Error(`HTTP ${res.status}`)
    return res.json()
  }).pipe(Effect.flatMap(Schema.decodeUnknownEffect(Schema.Array(User)))),
)
```

The one-argument form of `Effect.tryPromise` fails with `Cause.UnknownError`. Use the `{ try, catch }` form when the UI needs typed errors.

## Render an `AsyncResult`

Effect and Stream atoms hold `AsyncResult<A, E>`: `Initial`, `Success`, or `Failure`. Each state has a `waiting` flag for in-flight or background work.

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

Builder terminators: `.orNull()`, `.orElse(() => x)`, `.render()`, and `.exhaustive()`, which only exists once every case is handled. Other handlers: `onWaiting`, `onInitialOrWaiting`, `onError`, `onErrorTag`, `onErrorIf`, `onDefect`, `onInterrupt`. See [async-result.md](async-result.md).

## Service + runtime

The service module follows the module surface from [services-layers.md](services-layers.md). The atoms module builds a runtime from its layer.

```ts
// users.ts
export interface Interface {
  readonly getAll: Effect.Effect<ReadonlyArray<User>, UsersError>
  readonly findById: (id: string) => Effect.Effect<User, UsersError>
  readonly create: (name: string) => Effect.Effect<User, UsersError>
}

export class Service extends Context.Service<Service, Interface>()("@app/Users") {}

export const layer = Layer.effect(Service, Effect.gen(function* () { /* ... */ }))

export * as Users from "./users"
```

```ts
// user-atoms.ts
import { Users } from "./users"

export const runtime = Atom.runtime(Users.layer)

export const usersAtom = runtime.atom(
  Effect.gen(function* () {
    const users = yield* Users.Service
    return yield* users.getAll
  }),
)
```

The full `users.ts`, with a `requestJson` helper that maps network, status, and decode failures to typed errors, is in [atom-react-examples.md](atom-react-examples.md#2-service-typed-errors-and-runtime).

## Parameterized queries with `Atom.family`

```ts
export const userAtom = Atom.family((id: string) =>
  runtime.atom(
    Effect.gen(function* () {
      const users = yield* Users.Service
      return yield* users.findById(id)
    }),
  ),
)

// in a component: same id, same atom instance
const result = useAtomValue(userAtom(id))
```

Families memoize by `Equal` and `Hash`. In v4 these compare primitives, plain objects, arrays, maps, sets, and dates by content, so `userAtom({ id })` returns the same atom for the same `id` even when the object literal is new. Do not mutate a key after passing it. `Hash` caches the hash per object, so a mutated key misses the cache.

## Mutations with `runtime.fn`

```ts
export const createUserAtom = runtime.fn(
  Effect.fnUntraced(function* (name: string) {
    const users = yield* Users.Service
    return yield* users.create(name)
  }),
  { reactivityKeys: ["users"] }, // refreshes atoms tagged with "users"
)

const createUser = useAtomSet(createUserAtom)                                // fire and forget
const createUserP = useAtomSet(createUserAtom, { mode: "promise" })          // Promise<A>, rejects on failure
const createUserExit = useAtomSet(createUserAtom, { mode: "promiseExit" })   // Promise<Exit<A, E>>, never rejects

const status = useAtomValue(createUserAtom) // AsyncResult of the latest call
```

## Invalidate queries after mutations

Tag the query with `Atom.withReactivity`. A `runtime.fn` with matching `reactivityKeys` refreshes it when it completes.

```ts
export const usersAtom = runtime
  .atom(
    Effect.gen(function* () {
      const users = yield* Users.Service
      return yield* users.getAll
    }),
  )
  .pipe(Atom.withReactivity(["users"]))
```

## Pagination with `Atom.pull`

```tsx
const pagesAtom = Atom.pull(pagesStream) // runtime.pull(...) when the stream needs services

const [result, pull] = useAtom(pagesAtom)
// Success value: { items: NonEmptyArray<A>, done: boolean }; pull() loads the next chunk
```

Items accumulate across pulls. Pass `{ disableAccumulation: true }` to keep only the latest chunk.

## Suspense

```tsx
const users = useAtomSuspense(usersAtom).value // suspends until Success
```

Pass `{ suspendOnWaiting: true }` to also suspend during refreshes, and `{ includeFailure: true }` to get `Failure` back instead of a throw.

## Registry

Hooks use a default registry. Wrap the app in `<RegistryProvider>` for SSR, tests, or scoped disposal of atom state.

## Hooks

| Hook | Returns | Use for |
| --- | --- | --- |
| `useAtomValue(atom)` | value or `AsyncResult` | reading (subscribes) |
| `useAtomRefresh(atom)` | `() => void` | re-running a query |
| `useAtomSet(atom, { mode? })` | setter | writing, or triggering `fn` atoms |
| `useAtom(atom)` | `[value, setter]` | read and write, such as `Atom.pull` |
| `useAtomSuspense(atom)` | `AsyncResult.Success` | Suspense reads |
| `useAtomMount(atom)` | `void` | keeping an atom alive without reading it |
| `useAtomSubscribe(atom, f)` | `void` | side effects on change |

## Examples

[atom-react-examples.md](atom-react-examples.md) has complete components for every pattern above. They typecheck against `effect@4.0.6` and pass `@effect/tsgo` with the settings in [project-setup.md](project-setup.md).
