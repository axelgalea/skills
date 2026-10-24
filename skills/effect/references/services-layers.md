# Services, layers, and modules

Read this when you define a service, write a layer, wire a runtime, or name an operation with `Effect.fn`.

## Module surface

Each service gets its own file. The file exports fixed role names and ends by exporting itself as a namespace.

```ts
// user-repo.ts
import { Context, Effect, Layer, Schema } from "effect"
import { SqlClient } from "effect/sql"

export const UserId = Schema.String.pipe(Schema.brand("UserId"))
export type UserId = typeof UserId.Type

export const User = Schema.Struct({ id: UserId, name: Schema.String })
export interface User extends Schema.Schema.Type<typeof User> {}

export class NotFound extends Schema.TaggedError<NotFound>()(
  "UserRepo.NotFound",
  { id: UserId },
) {}

export interface Interface {
  readonly get: (id: UserId) => Effect.Effect<User, NotFound | PersistenceError>
}

export class Service extends Context.Service<Service, Interface>()(
  "@app/UserRepo",
) {}

export const layer = Layer.effect(
  Service,
  Effect.gen(function* () {
    const sql = yield* SqlClient.SqlClient

    const get = Effect.fn("UserRepo.get")(function* (id: UserId) {
      // ...
    })

    return Service.of({ get })
  }),
)

export * as UserRepo from "./user-repo.js"
```

Consumers import the namespace and bind the service before calling it:

```ts
import { UserRepo } from "./user-repo.js"

const program = Effect.gen(function* () {
  const repo = yield* UserRepo.Service
  return yield* repo.get(id)
})
```

Rules:

- Do not name the tag class `UserRepo` inside `user-repo.ts`. The namespace carries the domain name.
- Sibling modules import a namespace from the file that owns it, not through their own barrel.
- Folder and package barrels re-export with `export { UserRepo } from "./user-repo.js"`.
- Do not use TypeScript `namespace` declarations.
- Export only the intended surface. Keep row codecs, local schemas, and helpers unexported.
- Prefix error tags with the domain, such as `"UserRepo.NotFound"`, so tags stay unique across services.
- The self-export makes `UserRepo.UserRepo === UserRepo`. This works in Vite, esbuild, `tsc`, and Node ESM.
- Use the import extension the project's `tsconfig` expects: `.js` under `NodeNext`, none under `bundler`.

## Write leaf services first

Define the tags for leaf services before any layer exists. Then write the service that uses them. Its layer type-checks before the leaves have an implementation, so you can model the dependency graph first.

```ts
export class Service extends Context.Service<Service, Interface>()("@app/Tickets") {}
// no layer yet: Events.layer can already yield* Tickets.Service
```

## Layer constructors

Pick the constructor that matches what the layer produces.

```ts
Layer.succeed(Service, impl)       // already-built value
Layer.sync(Service, () => impl)    // lazy synchronous value
Layer.effect(Service, makeEffect)  // effectful acquisition
```

- Default real implementations to `Layer.effect(Service, Effect.gen(...))` and return `Service.of({ ... })`.
- Use `Layer.effectContext(...)` when one acquisition supplies several tags, such as a test fake behind two tags.
- Use `Layer.unwrap(...)` when config or runtime discovery picks the layer.
- Use `Layer.fresh(...)` or `Effect.provide(layer, { local: true })` only when a test or operation needs its own acquisition.
- Use `Context.Reference` only for ambient values with a real safe default.

## Store parameterized layers in a constant

Effect memoizes layers by reference. Two calls to a layer constructor build two instances.

```ts
// Bad: two connection pools
const app = Layer.mergeAll(
  UserRepo.layer.pipe(Layer.provide(Postgres.layer({ url, poolSize: 10 }))),
  OrderRepo.layer.pipe(Layer.provide(Postgres.layer({ url, poolSize: 10 }))),
)

// Good: one pool shared by both repos
const postgres = Postgres.layer({ url, poolSize: 10 })
const app = Layer.mergeAll(
  UserRepo.layer.pipe(Layer.provide(postgres)),
  OrderRepo.layer.pipe(Layer.provide(postgres)),
)
```

## Long-lived work

A layer that starts a stream, listener, worker, or loop forks it into the layer scope. Layer acquisition must finish.

```ts
export const layer = Layer.effectDiscard(
  Effect.gen(function* () {
    const events = yield* Events.Service

    yield* events.stream.pipe(
      Stream.runForEach(handleEvent),
      Effect.forkScoped,
    )
  }),
)
```

- Use `Effect.forkScoped`, `FiberSet`, or `FiberMap` for scoped background work.
- If methods must fork into the layer lifetime, capture `Scope.Scope` during acquisition and use `Effect.forkIn(scope)`. Do not expose the scope.
- Do not run forever-work inline during acquisition.
- Do not add a public `start` method unless the domain needs manual lifecycle control.

## Runtime wiring

- Use `Layer.provide(...)` to hide a dependency.
- Use `Layer.provideMerge(...)` only when downstream code needs the dependency. Tests often do, to reach leaf services for setup and assertions.
- Use `Layer.mergeAll(...)` for independent layers.
- Build the app layer as a flat, ordered list of named layer constants.
- Provide the app layer once, at the entry point.
- Do not add `provideMerge` or `mergeAll` just to make the types compile.

## Effect.fn

Name every public service method, and every non-trivial internal one, with `Effect.fn("Domain.operation")`. It adds a span and call-site tracing.

An operation with no arguments is a plain Effect value, not a function. An Effect is already lazy, and `@effect/tsgo` flags `() => Effect` members. Name the value with `Effect.withSpan`.

```ts
export interface Interface {
  readonly getAll: Effect.Effect<ReadonlyArray<User>, UsersError>
  readonly findById: (id: UserId) => Effect.Effect<User, UsersError>
}

const getAll = query.pipe(Effect.withSpan("Users.getAll"))
const findById = Effect.fn("Users.findById")(function* (id: UserId) {
  // ...
})
```

Extra arguments to `Effect.fn` wrap the whole call. Each one receives the effect and the original arguments.

```ts
const readAttachment = Effect.fn("Attachment.read")(
  function* (id: string) {
    return yield* api.read(id)
  },
  (effect, id) =>
    effect.pipe(
      Effect.mapError((cause) =>
        new AttachmentError({ operation: "Attachment.read", attachmentId: id, cause }),
      ),
    ),
)
```

Use these wrappers for error mapping, local recovery, log annotations, retry, timeout, and cleanup. Keep the generator body on the main workflow. One or two wrappers is enough. Use `Effect.fnUntraced` only for internal helpers that need no span.

## Operation error helpers

When many methods map infrastructure failures into one error with an operation label, share one curried helper and name it after the error.

```ts
const operationError =
  <E>(make: (props: { operation: string; cause: unknown }) => E) =>
  (operation: string) =>
  <A, E0, R>(self: Effect.Effect<A, E0, R>) =>
    Effect.mapError(self, (cause) => make({ operation, cause }))

const persistenceError = operationError(PersistenceError.make)

const row = yield* query.pipe(persistenceError("UserRepo.findById"))
```

The label goes in the error payload. `Effect.fn` spans still give you the trace.
