# Atom + React examples

Every block below is a standalone module. All of them typecheck together against
`effect@4.0.6` and `@effect/atom-react@4.0.6` under `strict` and
`exactOptionalPropertyTypes`, and pass `@effect/tsgo` with the settings in
[project-setup.md](project-setup.md). The examples use
`https://jsonplaceholder.typicode.com` as a stand-in API.

## 1. Minimal fetch atom (no services)

```tsx
// UsersListMinimal.tsx
import { Cause, Effect, Schema } from "effect"
import { AsyncResult, Atom } from "effect/reactivity"
import { useAtomRefresh, useAtomValue } from "@effect/atom-react"

const User = Schema.Struct({ id: Schema.Finite, name: Schema.String })

// Effect atoms surface as AsyncResult
const usersAtom = Atom.make(
  Effect.tryPromise(async (signal): Promise<unknown> => {
    const res = await fetch("https://jsonplaceholder.typicode.com/users", {
      signal,
    })
    if (!res.ok) throw new Error(`HTTP ${res.status}`)
    return res.json()
  }).pipe(Effect.flatMap(Schema.decodeUnknownEffect(Schema.Array(User)))),
)

export function UsersList() {
  const result = useAtomValue(usersAtom)
  const refresh = useAtomRefresh(usersAtom)

  return (
    <div>
      <button type="button" onClick={() => refresh()}>
        Refresh
      </button>

      {AsyncResult.builder(result)
        .onInitial(() => <p>Loading…</p>)
        .onFailure((cause) => <p>Error: {Cause.pretty(cause)}</p>)
        .onSuccess((users, r) => (
          <div>
            {r.waiting && <p>Refreshing…</p>}
            <ul>
              {users.map((u) => (
                <li key={u.id}>{u.name}</li>
              ))}
            </ul>
          </div>
        ))
        .orNull()}
    </div>
  )
}
```

## 2. Service, typed errors, and runtime

The service module follows the module surface from
[services-layers.md](services-layers.md). `requestJson` decodes the body with a
schema, so the success type comes from the schema and a bad payload fails with
`SchemaError`. `getAll` takes no arguments, so it is a plain Effect value named
with `Effect.withSpan`.

```ts
// users.ts
import { Context, Effect, Layer, Schema } from "effect"

export const User = Schema.Struct({ id: Schema.Finite, name: Schema.String })
export interface User extends Schema.Schema.Type<typeof User> {}

export class HttpError extends Schema.TaggedError<HttpError>()("Users.HttpError", {
  status: Schema.Finite,
}) {}

export class NetworkError extends Schema.TaggedError<NetworkError>()(
  "Users.NetworkError",
  { cause: Schema.Defect() },
) {}

type UsersError = HttpError | NetworkError | Schema.SchemaError

export interface Interface {
  readonly getAll: Effect.Effect<ReadonlyArray<User>, UsersError>
  readonly findById: (id: string) => Effect.Effect<User, UsersError>
  readonly create: (name: string) => Effect.Effect<User, UsersError>
}

export class Service extends Context.Service<Service, Interface>()("@app/Users") {}

const BASE = "https://jsonplaceholder.typicode.com"

// Shared by every method: maps network, status, and decode failures to typed errors
const requestJson = <S extends Schema.Top>(
  schema: S,
  path: string,
  init?: RequestInit,
) =>
  Effect.tryPromise({
    try: (signal) => fetch(`${BASE}${path}`, { ...init, signal }),
    catch: (cause) => new NetworkError({ cause }),
  }).pipe(
    Effect.filterOrFail(
      (res) => res.ok,
      (res) => new HttpError({ status: res.status }),
    ),
    Effect.flatMap((res) =>
      Effect.tryPromise({
        try: (): Promise<unknown> => res.json(),
        catch: (cause) => new NetworkError({ cause }),
      }),
    ),
    Effect.flatMap(Schema.decodeUnknownEffect(schema)),
  )

export const layer = Layer.effect(
  Service,
  Effect.gen(function* () {
    // A zero-argument operation is a plain Effect value, named with a span
    const getAll = requestJson(Schema.Array(User), "/users").pipe(
      Effect.withSpan("Users.getAll"),
    )

    const findById = Effect.fn("Users.findById")(function* (id: string) {
      return yield* requestJson(User, `/users/${id}`)
    })

    const create = Effect.fn("Users.create")(function* (name: string) {
      return yield* requestJson(User, "/users", {
        method: "POST",
        headers: { "content-type": "application/json" },
        body: JSON.stringify({ name }),
      })
    })

    return Service.of({ getAll, findById, create })
  }),
)

export * as Users from "./users"
```

The atoms live in their own module and reach the service through its namespace.

```ts
// user-atoms.ts
import { Effect } from "effect"
import { Atom } from "effect/reactivity"
import { Users } from "./users"

// AtomRuntime from a Layer: provides Users.Service to runtime.atom / .fn / .pull
export const runtime = Atom.runtime(Users.layer)

// Query, tagged so mutations with reactivityKeys ["users"] refresh it
export const usersAtom = runtime
  .atom(
    Effect.gen(function* () {
      const users = yield* Users.Service
      return yield* users.getAll
    }),
  )
  .pipe(Atom.withReactivity(["users"]))

// Parameterized query: one stable atom per id
export const userAtom = Atom.family((id: string) =>
  runtime.atom(
    Effect.gen(function* () {
      const users = yield* Users.Service
      return yield* users.findById(id)
    }),
  ),
)

// Mutation: invalidates everything tagged "users" when it completes
export const createUserAtom = runtime.fn(
  Effect.fnUntraced(function* (name: string) {
    const users = yield* Users.Service
    return yield* users.create(name)
  }),
  { reactivityKeys: ["users"] },
)
```

## 3. List with typed error handling

```tsx
// UsersList.tsx
import { AsyncResult } from "effect/reactivity"
import { useAtomRefresh, useAtomValue } from "@effect/atom-react"
import { usersAtom } from "./user-atoms"

export function UsersList() {
  const result = useAtomValue(usersAtom)
  const refresh = useAtomRefresh(usersAtom)

  return (
    <div>
      <button type="button" onClick={() => refresh()}>
        Refresh
      </button>
      {AsyncResult.builder(result)
        .onInitial(() => <p>Loading…</p>)
        .onErrorTag("Users.HttpError", (e) => <p>Server returned {e.status}</p>)
        .onErrorTag("Users.NetworkError", () => <p>Network unavailable</p>)
        .onErrorTag("SchemaError", () => <p>Unexpected response shape</p>)
        .onDefect(() => <p>Unexpected error</p>)
        .onInterrupt(() => null)
        .onSuccess((users, r) => (
          <div>
            {r.waiting && <p>Refreshing…</p>}
            <ul>
              {users.map((u) => (
                <li key={u.id}>{u.name}</li>
              ))}
            </ul>
          </div>
        ))
        .exhaustive()}
    </div>
  )
}
```

`.exhaustive()` only typechecks once every case (initial, each error tag,
defects, interrupts, success) is handled. Drop the `SchemaError` handler and
`.exhaustive()` stops compiling. Use `.orNull()` when you don't need that
guarantee.

## 4. Parameterized fetch with `Atom.family`

```tsx
// UserProfile.tsx
import { Cause } from "effect"
import { AsyncResult } from "effect/reactivity"
import { useAtomRefresh, useAtomValue } from "@effect/atom-react"
import { userAtom } from "./user-atoms"

export function UserProfile({ id }: { id: string }) {
  const result = useAtomValue(userAtom(id))
  const refresh = useAtomRefresh(userAtom(id))

  return (
    <div>
      <button type="button" onClick={() => refresh()}>
        Refresh
      </button>
      {AsyncResult.builder(result)
        .onInitial(() => <p>Loading…</p>)
        .onFailure((cause) => <p>Error: {Cause.pretty(cause)}</p>)
        .onSuccess((user) => (
          <h1>
            {user.id}: {user.name}
          </h1>
        ))
        .orNull()}
    </div>
  )
}
```

## 5. Mutation with `runtime.fn`

```tsx
// CreateUserButton.tsx
import { Exit } from "effect"
import { useAtomSet, useAtomValue } from "@effect/atom-react"
import { createUserAtom } from "./user-atoms"

export function CreateUserButton() {
  // mode: "promiseExit" returns Promise<Exit<User, Users.HttpError | Users.NetworkError | SchemaError>>
  const createUser = useAtomSet(createUserAtom, { mode: "promiseExit" })
  // the fn atom's own AsyncResult tracks the latest call
  const status = useAtomValue(createUserAtom)

  return (
    <button
      type="button"
      disabled={status.waiting}
      onClick={async () => {
        const exit = await createUser("Ada")
        if (Exit.isSuccess(exit)) {
          console.log("created", exit.value)
        }
      }}
    >
      {status.waiting ? "Saving…" : "Create user"}
    </button>
  )
}
```

Because `createUserAtom` declares `reactivityKeys: ["users"]`, `usersAtom`
(tagged with `Atom.withReactivity(["users"])`) refetches after a successful
call.

Setter modes:

- default (`"value"`): `(arg) => void`, fire and forget.
- `"promise"`: `(arg) => Promise<A>`, rejects on failure.
- `"promiseExit"`: `(arg) => Promise<Exit<A, E>>`, never rejects.

## 6. Infinite scroll and pagination with `Atom.pull`

```tsx
// PageList.tsx
import { Cause, Stream } from "effect"
import { AsyncResult, Atom } from "effect/reactivity"
import { useAtom } from "@effect/atom-react"

// Pulls one chunk at a time; use runtime.pull when the stream needs services
const pagesAtom = Atom.pull(Stream.make(1, 2, 3, 4, 5))

export function PageList() {
  const [result, pull] = useAtom(pagesAtom)

  return AsyncResult.builder(result)
    .onInitial(() => <p>Loading…</p>)
    .onFailure((cause) => <p>Error: {Cause.pretty(cause)}</p>)
    .onSuccess(({ items, done }, r) => (
      <div>
        <ul>
          {items.map((n) => (
            <li key={n}>{n}</li>
          ))}
        </ul>
        {!done && (
          <button type="button" onClick={() => pull()}>
            Load more
          </button>
        )}
        {r.waiting && <p>Loading more…</p>}
      </div>
    ))
    .orNull()
}
```

Items accumulate across pulls by default. Pass `{ disableAccumulation: true }`
to keep only the latest chunk.

## 7. Suspense

```tsx
// UsersSuspense.tsx
import { Suspense } from "react"
import { useAtomSuspense } from "@effect/atom-react"
import { usersAtom } from "./user-atoms"

function UsersSuspenseList() {
  // suspends until Success; failures throw to the nearest error boundary
  const users = useAtomSuspense(usersAtom).value
  return (
    <ul>
      {users.map((u) => (
        <li key={u.id}>{u.name}</li>
      ))}
    </ul>
  )
}

export function UsersPage() {
  return (
    <Suspense fallback={<p>Loading…</p>}>
      <UsersSuspenseList />
    </Suspense>
  )
}
```

## 8. App shell with `RegistryProvider`

Hooks work against a default registry. Use `RegistryProvider` for SSR, tests, or
scoped disposal:

```tsx
// App.tsx
import { RegistryProvider } from "@effect/atom-react"
import { CreateUserButton } from "./CreateUserButton"
import { UserProfile } from "./UserProfile"
import { UsersList } from "./UsersList"

export function App() {
  return (
    <RegistryProvider>
      <CreateUserButton />
      <UsersList />
      <UserProfile id="1" />
    </RegistryProvider>
  )
}
```
