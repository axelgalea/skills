# Application wiring

Read this when composing the apps in the full-stack layout. Use the
[Effect RPC reference](../../effect/references/rpc.md) for contracts, middleware,
repository boundaries, handlers, and the `rpc.ts` route layer used below.
Use [RPC client atoms](../../effect/references/rpc-atoms.md) for the browser
client and query and mutation definitions.

## Server composition

Place the shared `auth.ts`, `todos.ts`, and `app-rpcs.ts` contracts from the RPC
reference in `packages/domain/src`. Export them from the domain barrel.
Put `todo-repo.ts` and `todos-handlers.ts` in `apps/server/src/todos`.
Keep `rpc.ts` separate from the listener so transport tests can serve the same
routes on an ephemeral port.

The database layer owns acquisition and migrations. The repository layer uses
it. Handler layers use repositories. The route layer receives the handlers,
auth middleware, and serializer. The listener serves those routes.

```ts
// apps/server/src/server.ts
import { createServer } from "node:http"
import { NodeHttpServer } from "@effect/platform-node"
import { Config, Layer } from "effect"
import { HttpRouter } from "effect/http"
import { AuthMiddleware } from "./auth-middleware.ts"
import { Db } from "./db.ts"
import { Rpc } from "./rpc.ts"
import { TodoRepo } from "./todos/todo-repo.ts"
import { TodosHandlers } from "./todos/todos-handlers.ts"

const HttpLive = NodeHttpServer.layerConfig(createServer, {
  port: Config.Port("PORT").pipe(Config.withDefault(3000)),
})
const RepoLive = TodoRepo.layer.pipe(Layer.provide(Db.layer))
const HandlersLive = TodosHandlers.layer.pipe(Layer.provide(RepoLive))
const RoutesLive = Rpc.layer.pipe(
  Layer.provide([HandlersLive, AuthMiddleware.layer]),
)

export const layer = HttpRouter.serve(RoutesLive).pipe(Layer.provide(HttpLive))

export * as Server from "./server.ts"
```

`Db.layer`, `TodoRepo.layer`, and `AuthMiddleware.layer` are application
implementations. Supply their dependencies when composing them. Do not fix
missing services by merging unrelated layers.

```ts
// apps/server/src/main.ts
import { NodeRuntime } from "@effect/platform-node"
import { Layer } from "effect"
import { Server } from "./server.ts"

Layer.launch(Server.layer).pipe(NodeRuntime.runMain)
```

## Browser composition

Put the `ApiClient.Service` from the RPC atom reference in
`apps/web/src/api-client.ts`. Put feature atoms and components in
`apps/web/src/todos`. Keep route files focused on composing those components.
The web app imports contracts from domain and never imports server modules.

Use `/rpc` with HTTP and NDJSON for a new web app unless the project calls for
another transport. Proxy `/rpc` to the server in Vite during development. Route
that same path to the server in production, or configure the client URL for
the deployment. Reuse existing authentication in both apps.

Keep the client service shared across features. A new procedure in `AppRpcs`
needs feature atoms and a handler, not another client runtime or listener.
Choose reactivity keys according to which views a mutation changes. Server
streams need an explicit consumer or invalidation step to update those views.

## Other project shapes

A server-only app keeps the service, repository, route, and listener separation.
Put contracts in that app until a second consumer needs them. A CLI composes
its service layers at its command entry point and does not create a web app,
HTTP listener, or RPC contract without a requirement for one.
