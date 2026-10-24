# Caching, memoization, and request dedupe

Read this when you memoize lookups, cache per-key results with a TTL, dedupe concurrent identical calls, or consider request batching.

Use `effect/Cache` instead of a `Map` with timestamps and a prune loop.

## Core rules

- `Cache.make({ capacity, lookup, timeToLive })` caches per-key lookups with one TTL for every entry.
- `Cache.makeWith(lookup, { capacity, timeToLive: (exit, key) => ... })` computes the TTL per entry from the lookup's `Exit`. Use it to cache successes and skip failures.
- Concurrent `Cache.get` calls for the same missing key share one lookup. Do not add your own in-flight tracking.
- `capacity` is required and bounds the cache. Do not write eviction loops.
- `timeToLive` defaults to forever. Set it when freshness matters.
- Return a zero TTL to skip caching a transient failure or a degraded fallback. A short TTL on a stable failure, such as not found, protects the upstream.
- `Cache.invalidate(cache, key)` and `Cache.refresh(cache, key)` handle explicit staleness. `Cache.has` checks without a lookup.
- Build the cache once in the owning layer and share the handle. A cache built per call caches nothing.
- The cache captures the lookup's services at construction. Pass `requireServicesAt: "lookup"` only when each caller should supply them. This changes the cache's requirements type.
- For one value with no key, use `Effect.cached(effect)` or `Effect.cachedWithTTL(effect, ttl)`.
- For cached resources that need cleanup, such as connections, use `ScopedCache`.

## Cache successes, skip degraded results

```ts
const makeResolver = Effect.gen(function* () {
  const cache = yield* Cache.makeWith(
    (channelRef: string) => resolveUncached(channelRef), // never fails, returns { where, cacheable }
    {
      capacity: 300,
      timeToLive: (exit) =>
        Exit.isSuccess(exit) && exit.value.cacheable ? "10 minutes" : Duration.zero,
    },
  )
  return (channelRef: string) =>
    Cache.get(cache, channelRef).pipe(Effect.map((resolved) => resolved.where))
})
```

Rows that point at the same key during one burst share a single provider call.

## Acquire clients in the layer, not the lookup

A cache cannot fix a lookup that builds an SDK client or authenticates on every miss. Build the client once for the layer's lifetime.

```ts
// Bad: every cache miss builds a new client
const lookup = (id: string) =>
  getRecord(id).pipe(Effect.provide(apiClientLayer(options)))

// Good: built once; a miss is one API call.
// Layer.build needs a Scope, so run this inside the owning layer.
const context = yield* Layer.build(apiClientLayer(options))
const lookup = (id: string) => Context.get(context, ApiClient).getRecord(id)
```

## Request batching

`Effect.request` with a `RequestResolver` helps only when the backend has a real batch endpoint, such as SQL `IN (...)` or a batch GET.

| Situation | Use |
| --- | --- |
| Same key requested repeatedly over time | `Cache` |
| Same key requested concurrently in one burst | `Cache` |
| Many keys, backend has a batch endpoint | `Effect.request` plus `RequestResolver` |
| Many keys, per-item endpoint only | `Effect.forEach(..., { concurrency: n })`, optionally through a `Cache` |

- `RequestResolver.batchN(resolver, n)` bounds batch size.
- `RequestResolver.makeGrouped` groups requests that resolve through different targets.
- Against a per-item REST endpoint, a resolver still makes one call per item and gains nothing.
