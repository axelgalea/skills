# RPC and atom tests

Read this for handler, serialization, routing, or browser state tests.

Read [testing.md](testing.md) for test services, fresh layers, deterministic
synchronization, and configuration. This reference covers the RPC transport
and AtomRpc test APIs. Examples use `@acme/domain` for shared contracts.
Adapt the imports to the application.

## Handler behavior

`RpcTest.makeClient` connects the real RPC client and server machinery without
a serializer or network. It requires the group's handlers, middleware, and
`Scope`. `it.effect` supplies the scope.

```ts
// apps/server/src/todos/todos-handlers.test.ts
import { expect, it } from "@effect/vitest"
import { Effect, Layer } from "effect"
import { RpcTest } from "effect/rpc"
import { Todos } from "@acme/domain"
import { Db } from "../db.ts"
import { TestIdentity } from "../testing/test-identity.ts"
import { TodoRepo } from "./todo-repo.ts"
import { TodosHandlers } from "./todos-handlers.ts"

const RepoTest = TodoRepo.layer.pipe(Layer.provide(Db.testLayer))
const HandlersTest = TodosHandlers.layer.pipe(Layer.provide(RepoTest))
const TestLive = Layer.mergeAll(HandlersTest, TestIdentity)

it.effect("creates a todo and retrieves it", () =>
  Effect.gen(function* () {
    const client = yield* RpcTest.makeClient(Todos.Rpcs)
    const created = yield* client["Todos.Create"]({ title: "Write the test" })
    expect(created.title).toBe("Write the test")
    expect(created.done).toBe(false)
    expect(yield* client["Todos.Get"]({ id: created.id })).toEqual(created)
  }).pipe(Effect.provide(TestLive)),
)

it.effect("returns the declared failure for a missing todo", () =>
  Effect.gen(function* () {
    const client = yield* RpcTest.makeClient(Todos.Rpcs)
    const id = Todos.TodoId.make("missing")
    const error = yield* client["Todos.Get"]({ id }).pipe(Effect.flip)
    expect(error).toEqual(new Todos.NotFound({ id }))
  }).pipe(Effect.provide(TestLive)),
)
```

`Db.testLayer` is an application layer that creates and migrates a fresh test
database. `TestIdentity` is the helper from [rpc.md](rpc.md). This exercises the
real handlers and repository, but bypasses authentication. Add tests with the
real auth middleware for accepted and rejected credentials. Add a second user's
case for ownership behavior. Do not replace your own repository with a fake to
claim persistence coverage.

## HTTP transport

Add a test through the route and serializer when changing transport wiring or
public schema encoding. Use `NodeHttpServer.layerTest` for an ephemeral server
and an HTTP client whose base URL points to it.

```ts
// apps/server/src/rpc.test.ts
import { expect, it } from "@effect/vitest"
import { NodeHttpServer } from "@effect/platform-node"
import { Effect, Layer } from "effect"
import { HttpClient, HttpClientRequest, HttpRouter } from "effect/http"
import { RpcClient, RpcSerialization } from "effect/rpc"
import { AppRpcs } from "@acme/domain"
import { Rpc } from "./rpc.ts"
import { TestLive } from "./testing/rpc-layers.ts"

const RoutesTest = HttpRouter.serve(Rpc.layer).pipe(Layer.provide(TestLive))
const ProtocolTest = RpcClient.layerProtocolHttp({
  url: "",
  transformClient: HttpClient.mapRequest(HttpClientRequest.appendUrl("/rpc")),
}).pipe(
  Layer.provide(RpcSerialization.layerNdjson),
)
const HttpTest = Layer.mergeAll(RoutesTest, ProtocolTest).pipe(
  Layer.provide(NodeHttpServer.layerTest),
)

it.live("calls the RPC through HTTP and NDJSON", () =>
  Effect.gen(function* () {
    const client = yield* RpcClient.make(AppRpcs)
    const todo = yield* client["Todos.Create"]({ title: "Over HTTP" })
    expect(todo.title).toBe("Over HTTP")
    expect(yield* client["Todos.Get"]({ id: todo.id })).toEqual(todo)
  }).pipe(Effect.provide(HttpTest)),
)
```

Extract `TestLive` into a test module if both test files use it. It supplies
the same real handlers and repository with the chosen test identity. Keep the
normal fixed-port listener out of this graph. The test HTTP client already
prepends its server origin. Use an empty RPC URL and append `/rpc` after that
transformation. Passing `/rpc` as the RPC URL would prepend it to the absolute
test URL in Effect 4.0.6. NDJSON needs the same serializer at both ends.
In-memory RPC tests alone cannot catch encoding mistakes.

## Web atoms

Test the actual client atoms with a controlled RPC implementation at the
server boundary. Keep handler behavior tests separate from these client tests.

Build a test layer for the existing `ApiClient.Service` with
`Layer.effect(ApiClient.Service, RpcTest.makeClient(AppRpcs, { flatten: true }))`.
Provide domain-compatible test handlers and middleware to that layer. Flattening
matters because `AtomRpc.Service` uses a callable client, not the object client
used in the handler tests above.

Create a fresh registry and substitute the runtime layer before the first read:

```ts
const registry = AtomRegistry.make({
  initialValues: [Atom.initialValue(ApiClient.Service.runtime.layer, TestClient)],
})
```

`TestClient` is the completed layer described above. Use
`Effect.acquireRelease` to dispose the registry with `registry.dispose()`.
Mount the query with `AtomRegistry.mount(registry, queryAtom)` so it stays
observed. Read completion through `AtomRegistry.getResult(registry, atom,
{ suspendOnWaiting: true })`. These functions take the registry as an argument.
Provide `AtomRegistry.AtomRegistry` if the tested effect also requires that
service. Set a mutation through `registry.set` with its
`{ payload, reactivityKeys }` invocation object. Await the mutation result and
then the query's refreshed result.

Assert that the rendered data or actual query result changes after a mutation.
Use `Deferred` or the controlled handler's completion signal if the test needs
to distinguish the first response from the refresh. Do not sleep or assert only
that two atoms contain the same key string.

For streams, test the items and declared failure. If the change affects stream
lifetime, check cancellation or finalization with a deterministic signal.
