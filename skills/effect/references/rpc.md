# RPC contracts, middleware, and transport

Read this for an RPC contract, middleware, handler, or transport change.
Examples import contracts from `@acme/domain`. In a single-package app, use
local imports instead. The APIs do not require a monorepo.

For browser query and mutation atoms, read [rpc-atoms.md](rpc-atoms.md). For
RPC-specific test setup, read [rpc-testing.md](rpc-testing.md).

## Shared contracts

Declare identity and middleware in domain. The server implements authentication.

```ts
// packages/domain/src/auth.ts
import { Context, Schema } from "effect"
import { RpcMiddleware } from "effect/rpc"

export const UserId = Schema.NonEmptyString.pipe(Schema.brand("UserId"))
export type UserId = typeof UserId.Type

export class CurrentUser extends Context.Service<CurrentUser, {
  readonly id: UserId
}>()("@acme/Auth/CurrentUser") {}

export class Unauthorized extends Schema.TaggedError<Unauthorized>()(
  "Auth.Unauthorized",
  {},
) {}

export class Middleware extends RpcMiddleware.Service<Middleware, {
  provides: CurrentUser
}>()("@acme/Auth/Middleware", { error: Unauthorized }) {}

export * as Auth from "./auth.ts"
```

```ts
// packages/domain/src/todos.ts
import { Schema } from "effect"
import { Rpc, RpcGroup } from "effect/rpc"
import { Auth } from "./auth.ts"

export const TodoId = Schema.NonEmptyString.pipe(Schema.brand("TodoId"))
export type TodoId = typeof TodoId.Type

export const Todo = Schema.Struct({
  id: TodoId,
  title: Schema.NonEmptyString,
  done: Schema.Boolean,
}).annotate({ identifier: "Todo" })
export interface Todo extends Schema.Schema.Type<typeof Todo> {}

export class NotFound extends Schema.TaggedError<NotFound>()(
  "Todos.NotFound",
  { id: TodoId },
) {}

export class Rpcs extends RpcGroup.make(
  Rpc.make("Todos.List", { success: Schema.Array(Todo) }),
  Rpc.make("Todos.Get", {
    payload: { id: TodoId },
    success: Todo,
    error: NotFound,
  }),
  Rpc.make("Todos.Create", {
    payload: { title: Todo.fields.title },
    success: Todo,
  }),
).middleware(Auth.Middleware) {}

export * as Todos from "./todos.ts"
```

Apply constraints before brands. `Rpc.make` accepts a schema or a field map as
the payload. An omitted payload is `Schema.Void`. Middleware contributes its
error schema, so each procedure need not repeat `Unauthorized`.

```ts
// packages/domain/src/app-rpcs.ts
import { RpcGroup } from "effect/rpc"
import { Todos } from "./todos.ts"

export const AppRpcs = RpcGroup.make().merge(Todos.Rpcs)
```

Merge each new feature group here. Use unique feature-prefixed tags across all
groups. Both client and server import this same contract value.

## Middleware implementation

The middleware receives the handler effect and request metadata. Authenticate
with the project's session or token service, then provide `Auth.CurrentUser`
while running that effect. A client-supplied user ID is not authentication.
Declare other per-request services in the middleware's `requires` type when
needed. Services acquired while constructing the layer can stay in its closure.

The following layer is a fixed-identity test helper. It does not authenticate
requests and belongs only in tests.

```ts
// apps/server/src/testing/test-identity.ts
import { Effect, Layer } from "effect"
import { Auth } from "@acme/domain"

export const TestIdentity = Layer.succeed(
  Auth.Middleware,
  Auth.Middleware.of((effect) =>
    Effect.provideService(effect, Auth.CurrentUser, {
      id: Auth.UserId.make("test-user"),
    }),
  ),
)
```

For the real layer, check absent and invalid credentials before running the
handler. Fail with `Auth.Unauthorized`. Read credential configuration through
`Config`. Test the real middleware separately with accepted and rejected
credentials. Enforce resource ownership in the service or repository with the
authenticated identity.

## Repository boundary and handlers

This is the repository's public contract. Implement `layer` with the chosen
storage. Decode rows there, and map infrastructure failures there.

Use `SqlSchema` or a schema decoder at the SQL boundary. Account for the
database's representation, such as SQLite integers for boolean columns.
`SqliteClient.layer({ filename })` supplies both the SQLite and generic
`SqlClient` services. A fresh `":memory:"` acquisition gives a test its own
database. Run its migrations before testing queries.

```ts
// apps/server/src/todos/todo-repo.ts, public contract
import { Context, Effect, Schema } from "effect"
import { Auth, Todos } from "@acme/domain"

export class PersistenceError extends Schema.TaggedError<PersistenceError>()(
  "TodoRepo.PersistenceError",
  { operation: Schema.String, cause: Schema.Defect() },
) {}

export interface Interface {
  readonly list: (userId: Auth.UserId) => Effect.Effect<ReadonlyArray<Todos.Todo>, PersistenceError>
  readonly get: (userId: Auth.UserId, id: Todos.TodoId) => Effect.Effect<Todos.Todo, Todos.NotFound | PersistenceError>
  readonly create: (userId: Auth.UserId, title: string) => Effect.Effect<Todos.Todo, PersistenceError>
}

export class Service extends Context.Service<Service, Interface>()("@acme/TodoRepo") {}

export * as TodoRepo from "./todo-repo.ts"
```

```ts
// apps/server/src/todos/todos-handlers.ts
import { Effect } from "effect"
import { Auth, Todos } from "@acme/domain"
import { TodoRepo } from "./todo-repo.ts"

export const layer = Todos.Rpcs.toLayer(
  Effect.gen(function* () {
    const repo = yield* TodoRepo.Service
    return Todos.Rpcs.of({
      "Todos.List": () => Effect.gen(function* () {
        const user = yield* Auth.CurrentUser
        return yield* repo.list(user.id).pipe(
          Effect.catchTag("TodoRepo.PersistenceError", Effect.die),
        )
      }),
      "Todos.Get": ({ id }) => Effect.gen(function* () {
        const user = yield* Auth.CurrentUser
        return yield* repo.get(user.id, id).pipe(
          Effect.catchTag("TodoRepo.PersistenceError", Effect.die),
        )
      }),
      "Todos.Create": ({ title }) => Effect.gen(function* () {
        const user = yield* Auth.CurrentUser
        return yield* repo.create(user.id, title).pipe(
          Effect.catchTag("TodoRepo.PersistenceError", Effect.die),
        )
      }),
    })
  }),
)

export * as TodosHandlers from "./todos-handlers.ts"
```

Read `CurrentUser` inside each handler. Middleware provides it per request, not
while constructing the handler layer. Pass identity to storage operations that
need ownership checks.

`Todos.NotFound` passes through as a declared failure. This example turns
`PersistenceError` into a defect because its contract has no recovery for that
failure. If the client can act on an outage, declare a domain error and map it
instead. Never expose database causes as a public error schema. Configure server
observability and transport defect encoding for the application's needs.

## HTTP server

Keep the route layer separate from the listening server so tests can use it.

```ts
// apps/server/src/rpc.ts
import { RpcSerialization, RpcServer } from "effect/rpc"
import { Layer } from "effect"
import { AppRpcs } from "@acme/domain"

export const layer = RpcServer.layerHttp({
  group: AppRpcs,
  path: "/rpc",
  protocol: "http",
}).pipe(Layer.provide(RpcSerialization.layerNdjson))

export * as Rpc from "./rpc.ts"
```

## Streams and transport choices

Add a streaming procedure only when the feature needs a stream. For example,
`Rpc.make("Todos.Changes", { success: Todo, stream: true })` declares a stream
of todos. Its handler returns a `Stream`, not an array or an Effect of an array.
Declare stream errors in the RPC definition if they can occur.

HTTP with NDJSON supports streaming responses. Explicitly set `protocol:
"http"` because `RpcServer.layerHttp` defaults to WebSocket. Match the client
serializer and protocol. Check buffering, cancellation, and timeouts through
the deployed proxy when streaming matters.

For a persistent socket connection, use the server's `"websocket"` protocol
with `RpcClient.layerProtocolSocket`, a platform socket layer, and the same
serializer. WebSocket is a separate transport choice. It does not by itself
implement subscriptions, replay, reconnect consistency, or query invalidation.
