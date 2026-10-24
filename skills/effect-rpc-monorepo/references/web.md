# Browser client and atoms

Read this when wiring the browser client or changing a query, mutation, or
component. The contracts in [rpc.md](rpc.md) define these procedure names.

## One client service

```ts
// apps/web/src/api-client.ts
import { Layer } from "effect"
import { FetchHttpClient } from "effect/http"
import { AtomRpc } from "effect/reactivity"
import { RpcClient, RpcSerialization } from "effect/rpc"
import { AppRpcs } from "@acme/domain"

export class Service extends AtomRpc.Service<Service>()("@acme/ApiClient", {
  group: AppRpcs,
  protocol: RpcClient.layerProtocolHttp({ url: "/rpc" }).pipe(
    Layer.provide([FetchHttpClient.layer, RpcSerialization.layerNdjson]),
  ),
}) {}

export * as ApiClient from "./api-client.ts"
```

Configure Vite to proxy `/rpc` to the server during development. The browser
then uses one origin. Keep production routing consistent with that URL or
configure the client URL for the deployment.

`FetchHttpClient.layer` works in the browser without a platform-browser
dependency. Do not call `fetch` separately for the same application data.

Reuse the application's auth mechanism. Same-origin cookies follow the browser's
fetch rules. For request headers, use the query's `headers` option or the
mutation invocation's `headers` field. A shared HTTP transformation can attach a
token using `layerProtocolHttp({ transformClient })` and `HttpClient.mapRequest`.
Read the current token for each request if it can change. Rebuild or refresh
user-specific atom state when the signed-in user changes.

## Queries and mutations

```ts
// apps/web/src/todos/todo-atoms.ts
import { Atom } from "effect/reactivity"
import { Todos } from "@acme/domain"
import { ApiClient } from "../api-client.ts"

export const todosAtom = ApiClient.Service.query("Todos.List", undefined, {
  reactivityKeys: ["todos"],
})

export const todoAtom = Atom.family((id: Todos.TodoId) =>
  ApiClient.Service.query("Todos.Get", { id }, { reactivityKeys: ["todos"] }),
)

export const createTodoAtom = ApiClient.Service.mutation("Todos.Create")
```

Use stable family keys such as branded strings. Define atoms at module scope.
Do not construct a new atom each render.

```tsx
// apps/web/src/todos/todo-list.tsx
import { useAtomSet, useAtomValue } from "@effect/atom-react"
import { AsyncResult } from "effect/reactivity"
import { createTodoAtom, todosAtom } from "./todo-atoms.ts"

export function TodoList() {
  const result = useAtomValue(todosAtom)
  const mutation = useAtomValue(createTodoAtom)
  const createTodo = useAtomSet(createTodoAtom)

  return AsyncResult.builder(result)
    .onInitial(() => <p>Loading...</p>)
    .onErrorTag("Auth.Unauthorized", () => <p>Sign in to see your todos.</p>)
    .onFailure(() => <p>Could not load todos.</p>)
    .onSuccess((todos) => (
      <>
        <ul>{todos.map((todo) => <li key={todo.id}>{todo.title}</li>)}</ul>
        <button
          disabled={mutation.waiting}
          onClick={() => createTodo({
            payload: { title: "New todo" },
            reactivityKeys: ["todos"],
          })}
        >
          Add
        </button>
        {AsyncResult.isFailure(mutation) && <p>Could not create the todo.</p>}
      </>
    ))
    .orNull()
}
```

The query payload is the procedure payload. The mutation setter takes an
invocation object with a `payload` field. Its `reactivityKeys` invalidate
queries with matching keys in the same atom runtime. This is local client
invalidation, not a notification to other browsers. In Effect 4.0.6,
`Reactivity.mutation` invalidates keys after success. A failed mutation does not
invalidate them.

Mount one `RegistryProvider` from `@effect/atom-react` around the application.
Use the same registry for queries and mutations that share invalidation keys.
Keep failures visible in the UI. Do not render raw infrastructure causes to the
user.

## Streaming queries

`AtomRpc.Service.query` on a streaming procedure returns a pull atom. Use
`useAtomValue` to read it and `useAtomSet` to request another pull. Its success
value is a pull result with accumulated items, not the unary result type.
Inspect the installed `AtomRpc` and `Atom.PullResult` definitions before wiring
the view.

For continuous server updates, decide whether the view consumes events or
invalidates a list query. Define who owns the stream and when it stops. Do not
claim that adding `Todos.Changes` automatically refreshes `Todos.List`.
