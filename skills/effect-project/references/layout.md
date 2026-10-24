# Layout and development workflow

Read this for a new project or a change to package boundaries. Use only the
applications the project needs.

## CLI and server-only projects

Start a CLI with `src/main.ts` as its entry point. Put command handling and
service composition there or in command modules when several commands need
them. Organize service and domain modules by feature. Keep runtime adapters
outside the domain modules.

Start a server-only project with `src/main.ts`, `src/server.ts`, and feature
folders. Keep contracts in the app until another app needs them. A database
module belongs in the server that owns its acquisition and migrations.

## Full-stack layout

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

The domain package owns schemas, branded IDs, tagged errors, RPC groups, and
middleware declarations. It depends on `effect`. It has no React, SQL,
platform, Node, or DOM dependency. Apply the compiler boundaries in
[Effect project setup](../../effect/references/project-setup.md#package-compilation).

The server owns service implementations, repositories, migrations, middleware
implementations, and transport wiring. The web app owns the client, atoms,
routes, and components. Route files compose feature components.

Both apps declare a dependency on the domain workspace. Do not declare a
dependency between the apps. Use import restrictions for boundaries the
package graph cannot enforce, such as importing a server file by relative path.
Start with one shared domain package. Extract another shared package when a
second consumer needs it.

Use matching feature names across the packages. Start with one file per domain
feature. Split it when the feature needs several modules. Keep the public
namespace stable. Use the Effect skill's module export conventions.

```ts
// packages/domain/src/index.ts
export { AppRpcs } from "./app-rpcs.ts"
export { Auth } from "./auth.ts"
export { Todos } from "./todos.ts"
```

## Package compilation and library source

Give each package its own TypeScript config. Use project references for a
repository-wide contract check. Use source-only internal exports when both the
server runtime and web bundler support them. Follow
[Effect project setup](../../effect/references/project-setup.md#package-compilation)
for the compiler and runtime requirements.

Keep shallow library clones in the gitignored `repos/` directory. Match each
checkout to the installed version before using it to answer API questions.
Read a library only when the current task uses it. Do not import from or edit
the clones.

## Persistence ownership

Keep SQL client acquisition and migrations in `db.ts`. Run migrations before
serving requests. Reuse a named database layer constant so repositories share
its acquisition. Repositories own queries and row decoding. The Effect skill
owns service and schema conventions.

Give tests a fresh database acquisition. If production uses another database,
test its queries and migrations too. SQLite tests do not prove PostgreSQL
queries work.

## Development workflow

Keep one root `AGENTS.md` for project-wide rules. Add app instructions only for
rules that apply there. Document the exact package commands for installation,
migrations, development, type checking, lint, and tests. Have CI run the same
checks. Reuse existing scripts before adding new ones.

Use the compiler and Effect diagnostics from the Effect skill's project setup
reference. Keep an example feature small enough that the next feature can
follow its contract, handler, atoms, and tests.

Use per-worktree data paths and configurable ports when development must
support concurrent worktrees. Put configuration keys in `.env.example`. Read
them through the Effect skill's configuration patterns.

Reuse existing logs and traces for debugging. RPC and service spans need an
exporter or logger before an agent can read them in a file. Document the actual
output path when the project writes one. Add a generator, setup utility, or
tracing integration only when requested or needed by the current task.
