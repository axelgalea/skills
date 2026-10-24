# Layout and setup

Read this for a new project or a change to package boundaries.

## Package ownership

```text
packages/domain/src/
  index.ts
  app-rpcs.ts
  auth.ts
  todos.ts
apps/server/src/
  main.ts
  server.ts
  rpc.ts
  auth-middleware.ts
  db.ts
  todos/
    todo-repo.ts
    todos-handlers.ts
    todos-handlers.test.ts
apps/web/src/
  main.tsx
  api-client.ts
  routes/
  todos/
    todo-atoms.ts
    todo-list.tsx
    todo-atoms.test.ts
```

The domain package depends on `effect`. It owns schemas, branded IDs, tagged
errors, RPC groups, and middleware declarations. It has no React, SQL, platform,
Node, or DOM dependency. Disable ambient Node types with `types: []` and omit
the DOM libraries in its TypeScript config.

The server owns service implementations, repositories, migrations, middleware
implementations, and transport wiring. The web app owns the client, atoms,
routes, and components. Route files compose feature components.

Both apps declare a dependency on the domain workspace. Do not declare a
dependency between the apps. Use import restrictions for boundaries the
package graph cannot enforce, such as importing a server file by relative path.

Start with one file per domain feature. Split it when the feature needs several
modules. Keep the public namespace stable. Use kebab-case file names.

```ts
// packages/domain/src/index.ts
export { AppRpcs } from "./app-rpcs.ts"
export { Auth } from "./auth.ts"
export { Todos } from "./todos.ts"
```

Feature modules end with a namespace export, such as
`export * as Todos from "./todos.ts"`. Service modules expose `Interface`,
`Service`, `layer`, and their errors, then export their namespace.

## Runtime and TypeScript

Source-only internal packages can export `./src/index.ts` when both the server
runtime and web bundler support TypeScript sources. Do not publish that export
as a JavaScript library without a separate build decision.

Use `strict`, `exactOptionalPropertyTypes`, and `verbatimModuleSyntax`. Give
each package its own config. Use project references for a repository-wide
contract check.

- Domain and Node server configs use `module: "NodeNext"`. Write `.ts` in relative imports and use `rewriteRelativeImportExtensions` for emitted files.
- Web configs use `module: "preserve"`, `moduleResolution: "bundler"`, and the React JSX setting.
- Referenced projects need `composite`. A project consumed through build references must emit the declarations its consumers need. A web config with `noEmit` can be checked separately if the solution's build graph does not accept it as a reference.
- Use `erasableSyntaxOnly` if Node runs `.ts` files by stripping types. Verify support in the actual Node version. TypeScript path aliases alone do not configure Node's runtime resolution.

When available, use the Effect skill's project setup reference for
`@effect/tsgo`. Check the installed setup command before changing compiler
configuration. Do not disable unrelated diagnostics to make an example pass.

## Dependencies and source

Use `effect/rpc`, `effect/http`, `effect/sql`, and `effect/reactivity`. These
are v4 paths. Do not copy `effect/unstable/*` or `@effect-atom/*` imports from
older examples.

Choose platform adapters for the runtime. Node uses `@effect/platform-node`.
React hooks use `@effect/atom-react`. SQLite on Node uses
`@effect/sql-sqlite-node`. Tests use `@effect/vitest` and its supported Vitest
version. Read each package's peer dependencies when installing it.

Keep shallow library clones in the gitignored `repos/` directory. Inspect
`repos/effect/packages/effect/src/rpc` and `src/reactivity` for API questions.
Match the checkout to the installed version before treating it as evidence.
Read the router source only if the task uses router APIs.

## Persistence

Keep the SQL client layer and migration layer in `db.ts`. Run migrations before
serving requests. Reuse the same named database layer constant in the service
graph so repositories share its acquisition.

`SqliteClient.layer({ filename })` supplies both the SQLite and generic
`SqlClient` services. A fresh `":memory:"` layer gives a test its own database.
Decode rows with `SqlSchema` or a schema decoder in the repository. Account for
the database's representation, such as SQLite integers for boolean columns.
Do not pass raw rows to handlers with a cast.

Keep SQL and decoding failures in a repository error, such as
`TodoRepo.PersistenceError`. Keep provider calls outside database transactions.
If production uses a different database, test that database's queries and
migrations too.

## Development commands

Reuse existing scripts. For a new project, document commands that actually
exist for installation, migration, development, type checking, lint, and tests.
Have CI run the same checks. Do not add a generator, tracing stack, or setup
script merely to match the report's proposed tree.

Use per-worktree data paths and configurable ports when the development setup
must support concurrent worktrees. Put configuration keys in `.env.example`
and read them through `Config` in layers.

RPC and service spans need an installed exporter or logger to become a readable
trace file. Do not claim `.logs/server.jsonl` exists unless the project writes
it. Add app-specific instructions only where they change work in that app.
