# Schema, data modeling, and errors

Read this when you touch data models, wire contracts, brands, variants, optional fields, decoders, or typed errors.

## Records

Use `Schema.Struct` plus a same-name `interface`.

```ts
export const User = Schema.Struct({
  id: UserId,
  name: Schema.NonEmptyString,
  email: Schema.optionalKey(Schema.String),
})

export interface User extends Schema.Schema.Type<typeof User> {}
```

- Do not use `Schema.Class` or `Schema.TaggedClass` for new data models.
- Add `.annotate({ identifier: "User" })` only when tooling reads it: HTTP API, RPC, OpenAPI, JSON Schema, or codegen.

## Construction versus decoding

| Call | Input | Failure |
| --- | --- | --- |
| `User.make(input)` | trusted typed value | throws a plain `Error` |
| `User.makeEffect(input)` | typed value to validate | `SchemaIssue.Issue` |
| `User.makeOption(input)` | typed value, details not needed | `Option.none()` |
| `Schema.decodeUnknownEffect(User)(input)` | unknown wire value | `Schema.SchemaError` |
| `Schema.decodeUnknownSync(User)(input)` | scripts, tests, startup only | throws |
| `Schema.decodeUnknownResult(User)(input)` | pure code without Effect | `Result` |

- `make` and `makeEffect` apply defaults and type-side checks. They do not decode an encoded value. Decode a wire timestamp through its string-to-`DateTime` codec, not by passing the string to `make`.
- Wrap a constructor issue when the boundary needs `SchemaError`: `Effect.mapError((issue) => new Schema.SchemaError(issue))`.
- Use `Schema.fromJsonString(User)` to parse and decode a JSON string in one step.
- `Schema.NumberFromString` decodes `"12abc"` to `NaN` and `""` to `0`. Use `Schema.FiniteFromString` to reject `NaN`. It still decodes `""` to `0`, so check for an empty string first when that must fail.

## Reusing fields

```ts
export const CreateUserInput = Schema.Struct({
  name: User.fields.name,
  email: User.fields.email,
})

export const StoredUser = User.pipe(
  Schema.fieldsAssign({ createdAt: Schema.DateTimeUtcFromString }),
)
```

- Reuse `.fields`, `Schema.fieldsAssign`, and `.mapFields` only when contracts are related.
- Use `Schema.encodeKeys` when only the key names differ between decoded and encoded forms.
- Keep explicit mapping when the translation involves joins, validation, or domain rules.

## Optional fields and defaults

- `Schema.optionalKey(...)`: the key may be absent.
- `Schema.optional(...)`: only when explicit `undefined` is part of the contract.
- `Schema.NullOr`, `UndefinedOr`, `NullishOr`: only when the encoded contract carries those values.
- Keep defaulted values required, and apply the default in the constructor or decoder.

## Numbers

`Schema.Number` accepts `NaN`, `Infinity`, and `-Infinity`. Use `Schema.Finite` or `Schema.Int` for domain numbers.

## Brands

Brand IDs and value objects. Add constraints before the brand.

```ts
export const UserId = Schema.NonEmptyString.pipe(Schema.brand("UserId"))
export type UserId = typeof UserId.Type

export const Port = Schema.Int.check(
  Schema.isBetween({ minimum: 1, maximum: 65535 }),
).pipe(Schema.brand("Port"))
```

## Variants

Use `Data.TaggedEnum` for internal state. Use `Schema.TaggedUnion` when the union crosses a boundary and needs decoding, encoding, or JSON Schema.

```ts
type Step = Data.TaggedEnum<{
  Continue: { readonly cursor: number }
  Finished: { readonly count: number }
}>

export const Step = Data.taggedEnum<Step>()

const label = Step.$match(Step.Continue({ cursor: 10 }), {
  Continue: ({ cursor }) => `continue at ${cursor}`,
  Finished: ({ count }) => `finished ${count}`,
})
```

```ts
export const Event = Schema.TaggedUnion({
  Started: { runId: RunId },
  Finished: { runId: RunId, result: Schema.Json },
})

export type Event = typeof Event.Type

const event = Event.cases.Started.make({ runId })
const label = Event.match(event, {
  Started: ({ runId }) => `started ${runId}`,
  Finished: ({ runId }) => `finished ${runId}`,
})
```

- Use `Schema.TaggedStruct` for one reusable `_tag` variant.
- For an external discriminator such as `type`, use `Schema.Struct({ type: Schema.tag("created"), ... })` and `Schema.toTaggedUnion("type")` for union helpers.
- Use `Schema.tagDefaultOmit(...)` only when the encoded form omits the tag.

## Typed errors

```ts
export class PersistenceError extends Schema.TaggedError<PersistenceError>()(
  "UserRepo.PersistenceError",
  {
    operation: Schema.String,
    cause: Schema.Defect(),
  },
) {}
```

- Tagged errors are yieldable. Write `return yield* new NotFound({ id })`, not `yield* Effect.fail(new NotFound({ id }))`. The `return` tells TypeScript the generator stops there.
- Recover from one tag with `Effect.catchTag`, several with `Effect.catchTags`, and all typed errors with `Effect.catch`.
- Use typed errors for failures the caller can handle: not found, validation, permission, rate limit.
- Use defects for bugs and broken invariants. Convert with `Effect.orDie` where nothing can recover, such as missing config at startup.
- Wrap unknown library errors in a `Schema.Defect()` field.
- Map infrastructure failures into domain errors at the service boundary.
- Use a schema union of errors for a public API or transport error surface.
- Do not use cause-level recovery when typed recovery is enough. Preserve interruption when catching causes at an ingress, worker, or stream boundary.
