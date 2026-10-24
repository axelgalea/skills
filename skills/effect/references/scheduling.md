# Scheduling and retry

Read this when you write retries, repeats, polling workers, backoff, jitter, timeouts, or rate-limit-aware policies.

Use `Schedule` with `Effect.retry` and `Effect.repeat`. Do not write `while (true)` loops with sleeps.

## Core rules

- `Effect.retry(...)` retries typed failures. It does not retry defects or interruptions.
- `Effect.repeat(...)` repeats successes. A failure stops it unless the repeated effect handles the failure first.
- The effect runs once before the schedule takes its first step. `Schedule.recurs(3)` means three retries after the first run.
- `Schedule.spaced(...)` waits after each run finishes. `Schedule.fixed(...)` keeps a fixed cadence.
- Use `Schedule.exponential(...)` or `Schedule.fibonacci(...)` for backoff. Add `Schedule.jittered` so many clients do not retry at the same moment.
- Bound a delay schedule with `Schedule.upTo({ times })`.
- `Schedule.tap(({ input, output, duration }) => ...)` sees each step. Use it to log retries.
- Call `Schedule.setInputType<E>()` before input-dependent combinators when the input type would otherwise be `unknown`.
- Use `Effect.retryOrElse(...)` when exhausted retries need a fallback or a report.
- Retry only at the narrowest boundary with proven idempotency.

## Retry and timeout together

Time out each attempt, retry, then cap the whole operation.

```ts
const resilient = callApi.pipe(
  Effect.timeout("2 seconds"),
  Effect.retry(Schedule.exponential("100 millis").pipe(Schedule.upTo({ times: 3 }))),
  Effect.timeout("10 seconds"),
)
```

## Polling workers

Handle expected pass failures inside the pass, then repeat.

```ts
const pass = runPass().pipe(
  Effect.tapError((error) => Effect.logError("Worker.pass_failed", error)),
  Effect.ignore,
)

const run = pass.pipe(Effect.repeat(Schedule.spaced("1 second")))
```

Typed failures are logged and the worker continues. Defects still escape to supervision.

Catch causes only at a supervision boundary whose policy is "report any non-interrupt failure and continue":

```ts
const logNonInterruptCauseAndContinue = (message: string) =>
  Effect.catchCauseIf(
    (cause: Cause.Cause<unknown>) => !Cause.hasInterrupts(cause),
    (cause) => Effect.logError(message, cause),
  )
```

When only typed failures should recover, use `Effect.catchTag`, `Effect.catchIf`, `Effect.catchFilter`, or `Effect.retry`.

## Per-item failure isolation

Catch item-level failures around each item so one bad item does not stop the batch.

```ts
yield* Effect.forEach(
  items,
  (item) =>
    processItem(item).pipe(
      Effect.tapError((error) =>
        Effect.logError("Worker.item_failed", error).pipe(
          Effect.annotateLogs({ itemId: item.id }),
        ),
      ),
      Effect.ignore,
    ),
  { discard: true, concurrency: 5 },
)
```

Do this only when skipping the item, or retrying it later, is the product policy.

## Reusable retry policy

```ts
const projectionRetrySchedule: Schedule.Schedule<unknown, ProjectionError> =
  Schedule.exponential("100 millis").pipe(
    Schedule.jittered,
    Schedule.upTo({ times: 5 }),
  )

const reconcileWithRetry = (target: Target) =>
  reconcile(target).pipe(
    Effect.retryOrElse(
      projectionRetrySchedule.pipe(
        Schedule.tap(({ input: error }) =>
          Effect.logWarning("Projection.reconcile.retrying").pipe(
            Effect.annotateLogs({ operation: error.operation }),
          ),
        ),
      ),
      (error) => Effect.logError("Projection.reconcile.stopped", error),
    ),
  )
```

## Provider retry-after

Ask before adding this. It is extra robustness that only pays off when a provider rate-limits you in practice. When a typed provider error carries `retryAfterMs`, wait for the longer of the backoff delay and the provider's delay.

```ts
type RateLimited = { readonly retryAfterMs?: number | undefined }

const providerRetrySchedule: Schedule.Schedule<RateLimited, RateLimited> =
  Schedule.exponential("200 millis").pipe(
    Schedule.jittered,
    Schedule.upTo({ times: 5 }),
    Schedule.setInputType<RateLimited>(),
    Schedule.passthrough,
    Schedule.modifyDelay(({ input, duration }) =>
      Effect.succeed(
        input.retryAfterMs === undefined
          ? duration
          : Duration.max(duration, Duration.millis(input.retryAfterMs)),
      ),
    ),
  )
```

For HTTP-level `429` handling, see [http-clients.md](http-clients.md).

## Timeouts and delays

- `Effect.timeout(...)` for a real deadline.
- `Effect.delay(...)` to start one operation later.
- `Effect.sleep(...)` in production code only when waiting is the domain behavior.
- In tests, use `TestClock`. See [testing.md](testing.md).
