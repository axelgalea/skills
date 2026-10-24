# Testing

Read this when you write tests with `@effect/vitest`, or test time, retry, schedules, concurrency, workers, services, or config.

For RPC handlers, HTTP serialization, or AtomRpc client substitution, also read
[rpc-testing.md](rpc-testing.md). It uses the test conventions in this reference.

## Setup

Install `@effect/vitest` at the same version as `effect`. Install `vitest` 5, the range `@effect/vitest` accepts as a peer (`>=5.0.0 <6.0.0`). Run tests with `vitest run`, not `bun test`.

```ts
import { describe, expect, it } from "@effect/vitest"
import { Effect, Fiber } from "effect"
import { TestClock } from "effect/testing"
```

## Defaults

- Use `it.effect` by default. It provides `TestClock`, starts the clock at 0, and closes the test scope when the test ends.
- Use `it.live` only when real time or live services are the behavior under test.
- `it.effect` routes the console through `TestConsole`, so `Effect.log` output never prints, even with a logger layer provided. Use `it.live` when you need to see logs. Vitest 5.0.5 also hides console output from passing tests unless you run it with `--silent=false`.
- Use `it.effect.skip`, `it.effect.only`, and `it.effect.fails` as with plain vitest.
- Provide a fresh layer inside each test with `Effect.provide(...)`, so state never leaks between tests.
- Use `it.layer(layer)("suite", (it) => { ... })` only for a costly shared resource such as a database. All tests in the suite share its state.
- Assert typed failures, rollback, interruption, finalization, retry bounds, and idempotency where they apply.

## Virtual time

Do not sleep in tests. Fork the sleeping effect, then advance `TestClock`.

```ts
it.effect("advances virtual time", () =>
  Effect.gen(function* () {
    const fiber = yield* Effect.sleep("1 second").pipe(
      Effect.as("done"),
      Effect.forkScoped,
    )
    yield* TestClock.adjust("1 second")
    expect(yield* Fiber.join(fiber)).toBe("done")
  }),
)
```

Use `TestClock.setTime` or `TestClock.adjust` for sleeps, schedules, retries, leases, and timeouts.

## Synchronization

- `Deferred` for a one-shot ready or done signal.
- `Queue` to hand work or observed events across fibers.
- `Latch` for a reusable open and close gate.
- `Ref` for shared observation state.

```ts
it.effect("waits for the worker without sleeping", () =>
  Effect.gen(function* () {
    const ready = yield* Deferred.make<void>()
    const done = yield* Queue.unbounded<number>()

    yield* Effect.gen(function* () {
      yield* Deferred.succeed(ready, undefined)
      yield* Queue.offer(done, 1)
    }).pipe(Effect.forkScoped)

    yield* Deferred.await(ready)
    expect(yield* Queue.take(done)).toBe(1)
  }),
)
```

## Real layers first, fakes at the edge

Test your own services through their real `layer`. Write a fake only for an external boundary such as email, payments, or a provider API.

A fake backs two tags with one object. Production code depends on `Service`. Tests use `TestService` to inspect and control the fake.

```ts
export interface TestInterface extends Interface {
  readonly sentMessages: Effect.Effect<ReadonlyArray<Message>>
  readonly failNextSend: (error: SendError) => Effect.Effect<void>
}

export class TestService extends Context.Service<TestService, TestInterface>()(
  "@app/Notifier/Test",
) {}

export const testLayer = Layer.effectContext(
  Effect.gen(function* () {
    const sent = yield* Ref.make<ReadonlyArray<Message>>([])
    const nextFailure = yield* Ref.make<Option.Option<SendError>>(Option.none())

    const service = TestService.of({
      send: Effect.fn("Notifier.Test.send")(function* (message: Message) {
        const failure = yield* Ref.getAndSet(nextFailure, Option.none())
        if (Option.isSome(failure)) return yield* failure.value
        yield* Ref.update(sent, (messages) => [...messages, message])
      }),
      sentMessages: Ref.get(sent),
      failNextSend: Effect.fn("Notifier.Test.failNextSend")(function* (error: SendError) {
        yield* Ref.set(nextFailure, Option.some(error))
      }),
    })

    return Context.empty().pipe(
      Context.add(Service, service),
      Context.add(TestService, service),
    )
  }),
)
```

```ts
it.effect("records sent messages", () =>
  Effect.gen(function* () {
    const notifier = yield* Notifier.Service
    const control = yield* Notifier.TestService
    yield* notifier.send({ to: "ada" })
    expect(yield* control.sentMessages).toEqual([{ to: "ada" }])
  }).pipe(Effect.provide(Notifier.testLayer)),
)
```

- Use `Layer.succeed(Service, impl)` for a static stub that needs no inspection.
- Use `Layer.mock(Service, { ... })` only for a small partial stub where calling an omitted member should fail loudly.
- To test a service that orchestrates others, build its real layer with `Layer.provideMerge` over the leaf layers. The test can then reach the leaves for setup and assertions.

## Config in tests

- Test config decoding with `ConfigProvider.layer(ConfigProvider.fromUnknown({ ... }))`.
- Test code that only consumes config with `Layer.succeed(AppConfig.Service, { ... })`.
- Do not write to `process.env`.
