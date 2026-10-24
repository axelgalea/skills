# HTTP clients

Read this when you write outgoing HTTP calls, an API adapter, status handling, HTTP retries, or rate limiting.

Use the Effect HTTP client from `effect/http`: `HttpClient`, `HttpClientRequest`, `HttpClientResponse`, and `FetchHttpClient`. Effect marks these modules unstable, so their APIs can change between releases. Check `repos/effect/packages/effect/src/http` after an upgrade.

## Adapter shape

An adapter service owns the whole boundary: build the request, attach auth, run it, check the status, decode the body, map every failure to a domain error, and retry where the operation is idempotent.

```ts
export const Repo = Schema.Struct({ id: Schema.Int, full_name: Schema.String })
export interface Repo extends Schema.Schema.Type<typeof Repo> {}

export class GitHubError extends Schema.TaggedError<GitHubError>()("GitHub.Error", {
  operation: Schema.String,
  cause: Schema.Defect(),
}) {}

export interface Interface {
  readonly getRepo: (owner: string, name: string) => Effect.Effect<Repo, GitHubError>
}

export class Service extends Context.Service<Service, Interface>()("@app/GitHub") {}

export const layer = (token: Redacted.Redacted) =>
  Layer.effect(
    Service,
    Effect.gen(function* () {
      const client = (yield* HttpClient.HttpClient).pipe(
        HttpClient.mapRequest(HttpClientRequest.prependUrl("https://api.github.com")),
        HttpClient.mapRequest(HttpClientRequest.bearerToken(Redacted.value(token))),
        HttpClient.mapRequest(HttpClientRequest.acceptJson),
        HttpClient.filterStatusOk,
        HttpClient.retryTransient({ times: 3 }),
      )

      const getRepo = Effect.fn("GitHub.getRepo")(
        function* (owner: string, name: string) {
          const response = yield* client.get(`/repos/${owner}/${name}`)
          return yield* HttpClientResponse.schemaBodyJson(Repo)(response)
        },
        Effect.mapError((cause) => new GitHubError({ operation: "GitHub.getRepo", cause })),
      )

      return Service.of({ getRepo })
    }),
  ).pipe(Layer.provide(FetchHttpClient.layer))
```

Keep network calls out of business services and database transactions.

## Useful APIs

- `HttpClient.mapRequest(...)` and `mapRequestEffect(...)` configure a client once for every request.
- `HttpClientRequest.prependUrl`, `bearerToken`, and `acceptJson` set base URL, auth, and headers.
- `HttpClientRequest.bodyJson(...)` encodes a JSON body as an effect. `schemaBodyJson(schema)` encodes it through a schema.
- `HttpClient.filterStatusOk` or `HttpClientResponse.filterStatusOk` turns non-2xx responses into failures before decoding.
- `HttpClientResponse.schemaBodyJson(schema)` decodes the body. `schemaJson(schema)` decodes status, headers, and body. `schemaNoBody(schema)` decodes status and headers.

## Retry and rate limits

`HttpClient.retryTransient(...)` retries transport errors, timeouts, and the statuses `408`, `429`, `500`, `502`, `503`, and `504`. It waits at least as long as a `Retry-After` header asks. Put it after `filterStatusOk` so status failures reach it as errors.

`HttpClient.withRateLimiter(...)` paces requests and learns from rate-limit and `Retry-After` headers. It needs a `RateLimiter` plus an initial window, limit, and key. It adds `RateLimiterError` to the error channel and retries `429` with no limit by default. Set `times` to bound those retries, or `0` to turn them off.

Use an operation-level `Effect.retry(...)` when retry depends on domain errors, provider payloads, or idempotency rules. See [scheduling.md](scheduling.md).

## Raw fetch

Use raw `fetch` only for a platform transport, an API the Effect client cannot reach, or code that must not depend on unstable Effect modules. Keep it inside an adapter service.

```ts
export const request = Effect.fn("Provider.request")(function* (input: RequestInput) {
  const response = yield* Effect.tryPromise({
    try: (signal) => fetch(input.url, { signal, headers: input.headers }),
    catch: (cause) => new ProviderError({ operation: "Provider.request", cause }),
  })

  if (!response.ok) {
    return yield* new ProviderRejected({ operation: "Provider.request", status: response.status })
  }

  const json = yield* Effect.tryPromise({
    try: (): Promise<unknown> => response.json(),
    catch: (cause) => new ProviderError({ operation: "Provider.decodeJson", cause }),
  })

  return yield* Schema.decodeUnknownEffect(ResponseSchema)(json).pipe(
    Effect.mapError((cause) => new ProviderError({ operation: "Provider.decodeResponse", cause })),
  )
})
```

- Pass the `signal` from `Effect.tryPromise` to `fetch` so interruption cancels the request.
- Check the status before decoding the body.
- Type `response.json()` as `Promise<unknown>` and decode it with a schema. Do not cast it.
- Keep provider details needed for diagnosis, but redact secrets and private payloads.
