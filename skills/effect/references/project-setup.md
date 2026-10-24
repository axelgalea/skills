# Project setup

Read this when you set up Effect dependencies, compiler diagnostics, or package
compilation. The user can explicitly invoke `effect-project` for application
layout and package ownership. Do not invoke it automatically.

## Dependencies

Verify the current stable release for a new project. Pin `effect` and its
runtime and test `@effect/*` packages to the same release. Compiler tooling such
as `@effect/tsgo` has its own version. Check peer dependencies before choosing
React, Vitest, or platform package versions.

Choose runtime adapters for the actual runtime. Node uses
`@effect/platform-node`. React hooks use `@effect/atom-react`. SQLite on Node
uses `@effect/sql-sqlite-node`. Install only what the application uses.

## Effect language service

TypeScript 7 is the native compiler. Its Effect language service is `@effect/tsgo`. It adds Effect diagnostics to the editor and to `tsc`.

```bash
npx @effect/tsgo setup --project tsconfig.json --non-interactive --accept-defaults --apply --typescript
```

The setup command:

1. Adds `@effect/tsgo` to `devDependencies`.
2. Adds the plugin and its `$schema` to `tsconfig.json`.
3. Adds `"prepare": "effect-tsgo patch --typescript --no-oxlint"` to `package.json`. The patch makes `tsc` report Effect diagnostics, and `prepare` reapplies it after each install.

Then run the package manager's install command. Run `npx @effect/tsgo setup --help` for editor flags such as `--vscode`.

Use `effect-tsgo` as the only TypeScript language server in the editor. Running it next to plain `tsgo` duplicates every diagnostic.

### Rules to turn off

The defaults flag two things this skill does on purpose:

- `unstableApiUsage` warns on every Atom, HTTP client, SQL client, and `@effect/vitest` call, because Effect marks those modules unstable.
- `schemaClassMissingBrand` asks for a brand type argument on every `Schema.TaggedError`. The `_tag` literal already keeps error classes apart, and Effect's own source leaves them unbranded.

```jsonc
{
  "$schema": "./node_modules/@effect/tsgo/schema.json",
  "compilerOptions": {
    "plugins": [
      {
        "name": "@effect/language-service",
        "diagnosticSeverity": {
          "unstableApiUsage": "off",
          "schemaClassMissingBrand": "off"
        }
      }
    ]
  }
}
```

Keep the other defaults. They catch real mistakes: a missing `return` before `yield*` on a failing effect, `Effect.fail` around a yieldable error, a zero-argument service member that returns an Effect, and `Schema.Number` where `Schema.Finite` fits.

## tsconfig

Start from these options:

```jsonc
{
  "compilerOptions": {
    "target": "ES2022",
    "moduleDetection": "force",
    "strict": true,
    "exactOptionalPropertyTypes": true,
    "noUnusedLocals": true,
    "noImplicitOverride": true,
    "verbatimModuleSyntax": true,
    "skipLibCheck": true,
    "sourceMap": true
  }
}
```

- `exactOptionalPropertyTypes` keeps `{ x?: number }` from accepting `{ x: undefined }`, which matches `Schema.optionalKey`.
- `verbatimModuleSyntax` keeps `import type` as written.

Pick module settings by who compiles the code:

| Compiler | Settings | Used for |
| --- | --- | --- |
| A bundler (Vite, esbuild, Rollup) | `"module": "preserve"`, `"moduleResolution": "bundler"`, `"noEmit": true` | web apps |
| `tsc` | `"module": "NodeNext"`, and `"type": "module"` in `package.json` | Node and Bun apps, CLIs, libraries |

- `NodeNext` requires file extensions in relative imports. Add `"rewriteRelativeImportExtensions": true` to write `.ts` in imports and emit `.js`.
- Libraries add `"declaration": true` and `"declarationMap": true`. `declarationMap` fails without `declaration` or `composite`.
- Monorepo packages add `"composite": true` and `"incremental": true`.

## Package compilation

Give each package its own config. Use project references when one compiler
check should find stale contracts across packages. A project consumed through
build references must emit the declarations its consumers need. A web config
with `noEmit` can be checked separately if the solution's build graph does not
accept it as a reference.

For a runtime-independent domain package, omit DOM libraries and set
`types: []` so ambient Node types do not admit `process` or other Node globals.
Set the ECMAScript library according to the runtime target.

Source-only internal packages can export `./src/index.ts` when the runtime and
bundler support TypeScript sources. A published JavaScript library needs its
own build and export configuration.

Use `erasableSyntaxOnly` if Node runs `.ts` files by stripping types. Verify
support in the actual Node version. TypeScript path aliases alone do not
configure Node's runtime resolution. Use package exports for internal package
imports that must work in both the runtime and the bundler.
