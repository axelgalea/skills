# Config

Read this when you read env vars, `.env` files, or provider settings, or write a config-backed layer.

Read config through `Config` inside a layer. Application code depends on the resulting service, not on `Config` or `process.env`.

```ts
export interface Interface {
  readonly apiKey: Redacted.Redacted
  readonly model: Option.Option<string>
  readonly enabled: boolean
}

export class Service extends Context.Service<Service, Interface>()("@app/AppConfig") {}

export const layer = Layer.effect(
  Service,
  Effect.gen(function* () {
    const apiKey = yield* Config.Redacted("API_KEY")
    const model = yield* Config.option(Config.String("MODEL"))
    const enabled = yield* Config.Boolean("FEATURE_ENABLED").pipe(
      Config.withDefault(false),
    )

    return Service.of({ apiKey, model, enabled })
  }),
)
```

## Recipes

- Constructors are capitalized: `Config.String`, `Config.Boolean`, `Config.Int`, `Config.Duration`, `Config.URL`, `Config.Redacted`. Combinators such as `Config.option` and `Config.withDefault` stay lowercase.
- Use `Config.Redacted` for credentials. Read the value with `Redacted.value(secret)` at the call that needs it.
- Use `Config.schema(Schema, "NAME")` for refined or branded values.
- Use `Config.mapEffect` for a transformation that fails with `Config.ConfigError`.
- Use `Config.option` when absence means something.
- Use `Config.withDefault` for a missing value. A malformed value still fails.
- Use `Config.orElse` only when you mean to catch any config failure.

## Providers

The default provider is `ConfigProvider.fromEnv()`.

- `ConfigProvider.layer(provider)` replaces the provider for an app or a test suite.
- `ConfigProvider.layerAdd(provider)` adds a fallback. Pass `{ asPrimary: true }` to make the added provider win.
- `ConfigProvider.fromUnknown({ ... })` gives deterministic test config.
- `ConfigProvider.nested("APP")` scopes a provider under a prefix, so `PORT` reads `APP_PORT`.
- `ConfigProvider.constantCase` maps camelCase keys to `SCREAMING_SNAKE_CASE` env vars.
- Read `.env`, directory, and environment providers at startup, not inside business workflows.

## Config-backed library layers

A library-style service can offer a concrete `layer(options)` and a config-backed `layerConfig(options)`.

```ts
export const layerConfig = (config: Config.Wrap<ClientOptions>) =>
  Layer.effect(
    Service,
    Config.unwrap(config).pipe(
      Effect.map((options) => Service.of(makeClient(options))),
    ),
  )

const live = layerConfig({
  baseUrl: Config.String("BASE_URL"),
  apiKey: Config.Redacted("API_KEY"),
})
```

Tests pass concrete values to `layer(options)`. Production reads env through `layerConfig`.

## Config in tests

- To test config decoding, provide `ConfigProvider.layer(ConfigProvider.fromUnknown({ API_KEY: "k" }))`.
- To test code that only consumes config, provide `Layer.succeed(AppConfig.Service, { ... })` with the values the test needs.
- Do not write to `process.env` in tests.
