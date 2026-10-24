---
name: effect-rpc-monorepo
description: Create or extend an Effect v4 monorepo with shared RPC contracts, a server app, and a React app that uses Effect Atom. Use for project layout, RPC features, middleware, transport wiring, AtomRpc queries and mutations, and tests across those packages. Does not require restructuring an existing app to add one RPC.
---

# Effect RPC monorepo

Keep shared schemas and RPC contracts in `packages/domain`. Put implementations
in `apps/server` and components and atoms in `apps/web`. Both apps depend on
domain. Neither app imports the other.

The examples target `effect@4.0.6`. Check the project's installed version and
source before adapting them. RPC and Atom APIs change between releases.

## Read for the task

- New project or package boundaries: read [references/layout.md](references/layout.md).
- Contracts, middleware, handlers, or server transport: read [references/rpc.md](references/rpc.md).
- Browser client, query state, mutations, or streams in React: read [references/web.md](references/web.md).
- Adding or changing a feature: read [references/add-feature.md](references/add-feature.md), then the references it needs.
- Handler, transport, or atom tests: read [references/testing.md](references/testing.md).

Use the `effect` skill when available for general service, schema, config, and
test conventions. This skill includes the RPC-specific instructions needed when
installed alone.

## Resolve project choices

Read the project's instructions, manifests, lockfile, and existing code first.
Preserve its package layout, runtime, database, auth, and transport unless the
task calls for changing them. Ask only about choices that affect the requested
work and have no answer in the project or user request.

For a new project, use these defaults unless the user chooses otherwise:

- Use the existing package manager. If there is no lockfile or other instruction, use `nub`. Check its installed workspace commands before writing setup commands.
- Use Node, a client-rendered Vite app, React, and TanStack Router.
- Use HTTP with NDJSON at `/rpc`. Proxy that path in Vite.
- If persistence is required, use SQLite for local development and tests. Choose the production database with the user. SQLite tests do not prove PostgreSQL queries work.
- Carry identity through RPC middleware when the feature requires a user. Reuse existing auth. A development token example is an explicit development option, not production authentication.

Pin `effect` and the runtime and test `@effect/*` packages used here to the same
release. Compiler tooling such as `@effect/tsgo` has its own version. For new
dependencies, verify the current stable release rather than copying `4.0.6`.

## Keep the implementation small

- Start with one shared domain package. Add another package when it has a concrete consumer.
- Define records with `Schema.Struct` and a same-name interface. Define failures with `Schema.TaggedError`.
- Prefix RPC and error tags with the feature name, such as `Todos.Get` and `Todos.NotFound`.
- Keep handlers thin. Put persistence and business rules in services.
- Bind each yielded service to a variable before calling its methods.
- Use `Effect.fn("TodoRepo.get")` for service methods with arguments. Use an Effect value with `Effect.withSpan("TodoRepo.list")` for an operation with no arguments.
- Keep atoms at module scope or in `Atom.family`. Import atoms from `effect/reactivity` and hooks from `@effect/atom-react`.
- Decode unknown data at its owning boundary. RPC validates wire payloads. Repositories decode database rows.

## Finish the requested work

Use the project's check command. For a monorepo, run package tests from each
changed package directory. Check the contract, handler, and web call sites
together. A transport change also needs a request through the real serializer
and route.

This skill contains instructions and examples. It does not ship a template or
generator. Do not call a scaffold or feature-generation command unless the
project actually provides it. Add setup utilities only when requested.
