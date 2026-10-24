# Streams

Read this when you work with `Stream`, event sources, async iterables, queues, pubsub, pagination, backpressure, throttling, or long-lived consumers.

`Stream<A, E, R>` emits many `A` values over time, can fail with `E`, and needs services `R`. Streams are pull-based, so the consumer controls demand.

Use a stream for sources with many ordered values: gateway events, provider callbacks, subscriptions, paginated APIs, files, and scheduled ticks that carry values. Do not use a stream only to loop forever. For one repeated effect that emits nothing, use `Effect.repeat` with a `Schedule`. See [scheduling.md](scheduling.md).

## Sources

| Source | Constructor |
| --- | --- |
| In-memory values | `Stream.make(...)`, `Stream.fromIterable(...)` |
| Open-ended test fixture | `Stream.fromIterable(...).pipe(Stream.concat(Stream.never))` |
| Callback boundary, one consumer per item | `Queue` plus `Stream.fromQueue(...)` |
| Broadcast, every subscriber sees every event | `PubSub` plus `Stream.fromPubSub(...)` |
| Latest value plus updates | `SubscriptionRef` plus `SubscriptionRef.changes(ref)` |
| Ticks from a schedule | `Stream.fromSchedule(...)` |
| Paginated API | `Stream.paginate(...)` |
| Async iterable with no Effect source | `Stream.fromAsyncIterable(...)` |
| Stream built after reading services or config | `Stream.unwrap(...)` |

`Stream.paginate(initial, step)` takes a step that returns `Effect<readonly [ReadonlyArray<A>, Option<S>]>`. The stream emits the `A` items, not the page arrays. `Option.none()` ends pagination after that page's items. Empty pages are allowed.

## Transformations

- `Stream.map` for pure work, `Stream.mapEffect` for effectful work.
- `Stream.mapEffect(f, { concurrency })` for bounded parallel work. Add `unordered: true` when order does not matter.
- `Stream.flatMap(f)` for one input to many outputs. Pass `{ concurrency }` to run inner streams in parallel.
- `Stream.filter` and `Stream.filterEffect` to keep matching values.
- `Stream.mapAccum` and `Stream.mapAccumEffect` for stateful transforms.

## Consumers

- `Stream.runForEach(f)` for side effects.
- `Stream.runDrain` to run without keeping values.
- `Stream.runFold(...)` to fold into one value.
- `Stream.take(n)` plus `Stream.runCollect` for small finite streams and tests.
- Do not call `Stream.runCollect` on an unbounded or production event stream.

## Long-lived consumers

The layer that owns a consumer forks it into its scope.

```ts
export const layer = Layer.effectDiscard(
  Effect.gen(function* () {
    const gateway = yield* Gateway.Service

    yield* gateway.events.pipe(
      Stream.filter(isMessageEvent),
      Stream.runForEach(handleEvent),
      Effect.forkScoped,
    )
  }),
)
```

Keep stream failures visible unless the owning layer has a truthful recovery policy.

## Service shape

Expose streams from the interface. Keep queues and refs private to the implementation or the test service.

```ts
export interface Interface {
  readonly events: Stream.Stream<ProviderEvent, ProviderError>
  readonly status: Stream.Stream<ProviderStatus>
}
```

## Backpressure and buffers

Rely on the stream's own backpressure first. Add `Stream.buffer({ capacity, strategy })` only to decouple producer and consumer:

- `"suspend"` applies backpressure when full.
- `"dropping"` drops new values when full.
- `"sliding"` keeps the newest values and drops the oldest.
- `capacity: "unbounded"` only when something else bounds growth.

Use `Stream.debounce(...)` to wait for a quiet period. Use `Stream.throttle(...)` or `Stream.throttleEffect(...)` to cap the rate.

## Errors

- Translate errors at boundaries with `Stream.mapError`.
- Recover from typed errors with `Stream.catchTag`, `Stream.catchIf`, or `Stream.catchFilter`.
- Use `Stream.catchCause` only at a supervision boundary.

## Work keyed by id

For a stream of work keyed by session, channel, or id, where each key needs ordered processing and different keys can run in parallel, write one named helper on `FiberMap`. Do not scatter fiber bookkeeping through consumers.

## Tests

- `Stream.fromIterable` for finite fixtures and `Stream.empty` for no events.
- A test-owned `Queue` with `Stream.fromQueue` when the test drives events step by step.
- `Stream.take(n)` plus `Stream.runCollect` for assertions.
- Coordinate with `Deferred`, `Queue`, `Latch`, and `TestClock`, never real sleeps.
