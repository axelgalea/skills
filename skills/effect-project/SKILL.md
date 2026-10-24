---
name: effect-project
description: Structure an Effect v4 project with the repository's package boundaries, feature folders, application wiring, and development workflow. Use only when the user explicitly invokes effect-project. Covers CLI, server-only, and full-stack web projects. Use the effect skill for coding and API guidance.
disable-model-invocation: true
---

# Effect project

Apply this skill only when the user explicitly invokes it, such as with
`$effect-project`. Do not select it automatically for project structure tasks
or because another skill mentions it.

Use this skill with the `effect` skill from this repository. Install both.
Read [the Effect skill](../effect/SKILL.md), then only its references needed for
the task. It owns coding conventions and API patterns. This skill owns project
structure and the workflow across modules and packages.

## Choose the project shape

Read the project's instructions, manifests, lockfile, and existing code first.
For a new project, apply the conventions below. For an existing project, keep
its structure unless the user asks to reorganize it. Place a new feature within
the existing boundaries rather than creating another architecture beside them.

- A CLI starts with one package and a command entry point that composes service layers.
- A server-only project starts with one app. Keep its contracts local until another consumer needs them.
- A full-stack web project starts with `apps/server`, `apps/web`, and `packages/domain`. Both apps depend on domain. Neither app imports the other.

Add packages and app processes when the requested behavior has a consumer for
them. A CLI does not need the full-stack layout.

## Read for the task

- Project layout, package ownership, persistence placement, and development workflow: read [references/layout.md](references/layout.md).
- Server and browser composition in the full-stack layout: read [references/app-wiring.md](references/app-wiring.md).
- Adding or changing a feature: read [references/add-feature.md](references/add-feature.md).
- Checks across packages and application boundaries: read [references/verification.md](references/verification.md).

For TypeScript and compiler setup, use
[Effect project setup](../effect/references/project-setup.md). For RPC APIs,
use [Effect RPC](../effect/references/rpc.md). For browser RPC state, use
[RPC client atoms](../effect/references/rpc-atoms.md). Do not duplicate those
instructions in project documentation.

## Project defaults

Use the existing package manager. If there is no lockfile or other instruction,
use `nub`. Check its installed workspace commands before writing setup commands.
Use Node for a new CLI or server unless the user chooses another runtime.

For a new web app, use Vite, React, TanStack Router, and Effect Atom. Use HTTP
with NDJSON at `/rpc` and a Vite proxy for that path. Keep a feature at matching
paths in domain, server, and web so its contract and implementation are easy to
find. The full-stack example follows these defaults.

If persistence is required, use SQLite for local development and tests unless
the project specifies another database. Resolve the production database when
it affects the work. Reuse the project's authentication. Put new authentication
behind server middleware when the feature requires an authenticated user.

Keep commands for setup, migrations, development, and checks in package scripts.
Document the commands that exist. CI runs the same checks as local development.
Use [verification.md](references/verification.md) before finishing.

This skill contains instructions and examples. It does not ship setup utilities
or a generator. Add those only when requested.
