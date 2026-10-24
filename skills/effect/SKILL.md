---
name: effect
description: Rules and patterns for TypeScript applications built on the latest Effect v4 release. Use when writing, editing, or reviewing code that imports `effect` or `@effect/*`, including services, layers, schemas, typed errors, config, tests with `@effect/vitest`, HTTP clients, RPC contracts and middleware, retry and polling, streams, caching, compiler setup, and React state with Effect Atom (`effect/reactivity`, `@effect/atom-react`).
---

# Effect

Verified against `effect@4.0.6`. Pin `effect` and its runtime and test
`@effect/*` packages to the same release. Compiler tooling such as `@effect/tsgo`
has its own version. Check `repos/effect` when an API is not covered here.

The user can explicitly invoke `effect-project` for project layout, package
ownership, and development workflow. Do not invoke it automatically. This skill
owns coding conventions and API guidance and works without the project skill.

## Read the reference for the task

Read only the files that match the task. If a task spans several, read all of them before editing.

| Task | File |
| --- | --- |
| Service modules, layers, runtime wiring, `Effect.fn`, background work | [references/services-layers.md](references/services-layers.md) |
| Records, brands, variants, optional keys, decoding, typed errors | [references/schema-errors.md](references/schema-errors.md) |
| Env vars, `Config`, `ConfigProvider`, config-backed layers | [references/config.md](references/config.md) |
| `@effect/vitest`, virtual time, fakes, test layers | [references/testing.md](references/testing.md) |
| Dependencies, `tsconfig`, package compilation, Effect language service | [references/project-setup.md](references/project-setup.md) |
| Outgoing HTTP calls and API adapters | [references/http-clients.md](references/http-clients.md) |
| RPC contracts, middleware, handlers, serializers, transports | [references/rpc.md](references/rpc.md) |
| RPC queries and mutations with `AtomRpc` | [references/rpc-atoms.md](references/rpc-atoms.md) |
| RPC handlers, HTTP transport, or AtomRpc tests | [references/rpc-testing.md](references/rpc-testing.md) |
| Retry, repeat, polling, backoff, timeouts | [references/scheduling.md](references/scheduling.md) |
| Streams, queues, pubsub, pagination, event consumers | [references/streams.md](references/streams.md) |
| Memoization, TTL caches, request dedupe, batching | [references/caching.md](references/caching.md) |
| React components with atoms, `AsyncResult`, hooks | [references/atom-react.md](references/atom-react.md) |

## Defaults for every task

- Each service lives in its own module that exports `Interface`, `Service`, `layer`, and its errors, and ends with `export * as UserRepo from "./user-repo.js"`. Consumers write `UserRepo.Service`.
- Wrap every public service method in `Effect.fn("UserRepo.get")`. An operation with no arguments is a plain Effect value named with `Effect.withSpan`, not a `() => Effect` function.
- Fail with a tagged error by writing `return yield* new NotFound({ id })`.
- Compose workflows with `Effect.gen(function* () { ... })`.
- Model records with `Schema.Struct` plus a same-name `interface`. Do not use `Schema.Class`.
- Define typed errors with `Schema.TaggedError`.
- Decode unknown input with `Schema.decodeUnknownEffect`.
- Read runtime config through `Config` in a layer, never `process.env` in application code.
- Use `Schedule` for retry and repeat, `Stream` for many values over time, and `Cache` for keyed memoization. Do not hand-roll loops, sleeps, or `Map` caches.

## Names that changed

Model training data and most online examples use v3 or v4 prerelease names.

| Old | Current |
| --- | --- |
| `effect/unstable/http`, `effect/unstable/reactivity`, `effect/unstable/cli` | `effect/http`, `effect/reactivity`, `effect/cli` |
| `import { TestClock } from "effect"` | `import { TestClock } from "effect/testing"` |
| `Context.Tag`, `Effect.Service`, `.Default` | `Context.Service` plus an explicit `layer` |
| `Schema.TaggedErrorClass` | `Schema.TaggedError` |
| `Config.string`, `Config.redacted`, `Config.mapOrFail` | `Config.String`, `Config.Redacted`, `Config.mapEffect` |
| `Effect.catchAll` | `Effect.catch`, or `Effect.catchTag` for one tag |
| `Effect.fork`, `Effect.forkDaemon` | `Effect.forkChild`, `Effect.forkDetach`, or `Effect.forkScoped` in a layer |
| `Either` | `Result` |
| `Atom` result type `Result` | `AsyncResult` |

## Boundary rules

- Keep HTTP handlers thin: decode input, call services, map typed errors to responses. Business rules live in services.
- Wrap SDKs, CLIs, and external APIs in a service at the adapter boundary.
- Keep network calls outside database transactions.
- Catch or retry only where the boundary has a truthful response. Retry only idempotent operations.
- Let exhausted failures stay visible unless the boundary has a real fallback.
- Do not use `Context.Reference` defaults to hide credentials, persistence, transports, or external services.

## Credit

Most reference material is adapted from Kit Langton's [`effect` skill](https://github.com/kitlangton/skills) and [effect-solutions](https://github.com/kitlangton/effect-solutions), both MIT licensed. See [NOTICE.md](NOTICE.md).
