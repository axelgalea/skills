# SQL models

Read this when you define a record stored in a SQL table, write a repository,
or return stored records through RPC or HTTP.

Define each SQL table row with `Model.Class` from `effect/schema`. One field
declaration produces six schemas, and `SqlModel` and `SqlSchema` from
`effect/sql` read and write rows with them. This is the one place where the
skill uses a class for data. Records that are not stored in SQL keep
`Schema.Struct` plus a same-name `interface`, as described in
[schema-errors.md](schema-errors.md).

`effect/schema/Model` imports nothing from `effect/sql`. A shared domain package
can own the model, and only server code imports `effect/sql`. Effect marks
`Model` and `SqlModel` unstable, so check `repos/effect/packages/effect/src/schema/Model.ts`
after an upgrade.

## Define the model

```ts
// users/user.ts
import { Schema } from "effect"
import { Model } from "effect/schema"

export const UserId = Schema.String.pipe(Schema.brand("UserId"))
export type UserId = typeof UserId.Type

export class User extends Model.Class<User>("User")({
  id: Model.UuidV4Insert(UserId),
  email: Schema.String,
  passwordHash: Model.Sensitive(Schema.String),
  bio: Model.FieldOption(Schema.String),
  admin: Model.BooleanSqlite,
  createdAt: Model.DateTimeInsert,
  updatedAt: Model.DateTimeUpdate,
}) {}
```

The class is the `select` variant. Its static properties are the other five.

| Schema | Use for | Fields of `User` |
| --- | --- | --- |
| `User` | rows read from the database | all seven |
| `User.insert` | insert input | all seven |
| `User.update` | update input | all except `createdAt` |
| `User.json` | records an API returns | all except `passwordHash` |
| `User.jsonCreate` | API create input | `email`, `bio`, `admin` |
| `User.jsonUpdate` | API update input | `email`, `bio`, `admin` |

Column names are the field names. Set `transformQueryNames` and
`transformResultNames` on the SQL client if the table uses snake_case columns.

## Field helpers

| Helper | Column behavior |
| --- | --- |
| `Model.UuidV4Insert(Id)`, `Model.UuidV7Insert(Id)` | The app generates the id on insert. Clients cannot send it in `jsonCreate` or `jsonUpdate`. |
| `Model.Sensitive(schema)` | Present in `select`, `insert`, and `update`. Absent from every JSON variant. |
| `Model.FieldOption(schema)` | A nullable column decoded as `Option`. JSON omits the key for `Option.none()`. |
| `Model.BooleanSqlite` | Stored as `0` or `1`, exposed as `boolean` in JSON. |
| `Model.DateTimeInsert` | Set to the current time on insert. Absent from `update` and the JSON inputs. Stored as a string. |
| `Model.DateTimeUpdate` | Set to the current time on insert and on every update. Stored as a string. |
| `Model.DateTimeInsertFromDate`, `Model.DateTimeUpdateFromDate` | The same, for drivers that return `Date` objects. |
| `Model.JsonFromString(schema)` | A JSON document stored in a text column. |
| `Model.GeneratedByDb(schema)` | A column the database fills. Present only in `select` and `json`. |
| `Model.FieldExcept([...])`, `Model.FieldOnly([...])` | Pipe a schema through these to drop it from, or keep it in, the listed variants. |
| `Model.Field({ select, insert, ... })` | Name the schema for each variant yourself. |

Do not use `Model.GeneratedByDb` for a primary key that you pass to
`SqlModel.makeRepository`. The repository updates rows by id, so the id must
be in the `update` variant, and `makeRepository` rejects the model at compile
time. For an autoincrement key, declare the variants yourself:

```ts
export class Todo extends Model.Class<Todo>("Todo")({
  id: Model.Field({ select: TodoId, update: TodoId, json: TodoId }),
  title: Schema.NonEmptyString,
}) {}
```

## Build insert and update values

- Build inserts with `User.insert.makeEffect({ ... })`. It generates the id and
  the timestamps with the Effect clock, so `TestClock` controls them in tests.
  `User.insert.make` reads the system clock instead.
- `update` replaces the whole row. `makeRepository` writes every field in the
  `update` variant, so pass every field, not only the changed ones.
- Do not spread a selected row into `User.update.makeEffect`. The timestamp
  fields accept only `Model.Override(value)`, so the spread fails to compile.
  Leave them out to get the current time, or wrap a value in `Model.Override`
  to set it.

## Repository

`SqlModel.makeRepository` builds `insert`, `insertVoid`, `update`, `updateVoid`,
`findById`, and `delete` from the model. It needs `SqlClient`. Use `SqlSchema`
for any query it does not cover. Wrap both in a repository service with the
module surface from [services-layers.md](services-layers.md), and map their
errors into the repository's own errors.

```ts
// users/user-repo.ts
import { Context, Effect, Layer, Option, Schema } from "effect"
import { SqlClient, SqlModel, SqlSchema } from "effect/sql"
import { User, UserId } from "./user.ts"

export class NotFound extends Schema.TaggedError<NotFound>()(
  "UserRepo.NotFound",
  { id: UserId },
) {}

export class PersistenceError extends Schema.TaggedError<PersistenceError>()(
  "UserRepo.PersistenceError",
  { operation: Schema.String, cause: Schema.Defect() },
) {}

export interface Interface {
  readonly create: (email: string, passwordHash: string) => Effect.Effect<User, PersistenceError>
  readonly findById: (id: UserId) => Effect.Effect<User, NotFound | PersistenceError>
  readonly findByEmail: (email: string) => Effect.Effect<Option.Option<User>, PersistenceError>
}

export class Service extends Context.Service<Service, Interface>()("@app/UserRepo") {}

export const layer = Layer.effect(
  Service,
  Effect.gen(function* () {
    const sql = yield* SqlClient.SqlClient
    const repo = yield* SqlModel.makeRepository(User, {
      tableName: "users",
      spanPrefix: "UserRepo",
      idColumn: "id",
    })

    const findByEmailQuery = SqlSchema.findOneOption({
      Request: Schema.String,
      Result: User,
      execute: (email) => sql`SELECT * FROM users WHERE email = ${email}`,
    })

    const create = Effect.fn("UserRepo.create")(
      function* (email: string, passwordHash: string) {
        const input = yield* User.insert.makeEffect({
          email,
          passwordHash,
          bio: Option.none(),
          admin: false,
        })
        return yield* repo.insert(input)
      },
      Effect.mapError((cause) => new PersistenceError({ operation: "UserRepo.create", cause })),
    )

    const findById = Effect.fn("UserRepo.findById")(
      function* (id: UserId) {
        return yield* repo.findById(id).pipe(
          Effect.catchTags({
            NoSuchElementError: () => new NotFound({ id }),
            SchemaError: (cause) => new PersistenceError({ operation: "UserRepo.findById", cause }),
            SqlError: (cause) => new PersistenceError({ operation: "UserRepo.findById", cause }),
          }),
        )
      },
    )

    const findByEmail = Effect.fn("UserRepo.findByEmail")(
      function* (email: string) {
        return yield* findByEmailQuery(email)
      },
      Effect.mapError((cause) => new PersistenceError({ operation: "UserRepo.findByEmail", cause })),
    )

    return Service.of({ create, findById, findByEmail })
  }),
)

export * as UserRepo from "./user-repo.ts"
```

- `findById` fails with `NoSuchElementError` when no row matches. Map it to the
  domain's not-found error.
- Every repository method can fail with `SchemaError` and `SqlError`. A unique
  constraint violation arrives as `SqlError`.
- Pass `softDeleteColumn` to make reads skip rows where that column is set. With
  it, `delete` sets the column to `CURRENT_TIMESTAMP` instead of deleting the row.
- `SqlSchema.findAll`, `findOne`, `findOneOption`, `findNonEmpty`, and
  `SqlSchema.void` encode the request with `Request` and decode rows with
  `Result`. Pass the model as `Result` so rows decode to model instances.
  `findOne` fails with `NoSuchElementError` on zero rows. `findOneOption`
  returns `Option.none()`.

## Batched lookups

`SqlModel.makeResolvers` builds request resolvers for `insert`, `insertVoid`,
`findById`, and `delete`. It needs `SqlClient` and `Scope`. Call
`SqlResolver.request(id, resolvers.findById)`. Concurrent `findById` requests
share one `WHERE id IN (...)` query. See [caching.md](caching.md) for when
batching helps.

## Contracts

Put the JSON variants in RPC and HTTP API contracts. Use `User.json` for
responses and `User.jsonCreate` or `User.jsonUpdate` for payloads.

```ts
Rpc.make("Users.Create", { payload: User.jsonCreate, success: User.json })
```

- A handler can return the `User` row as it comes from the repository. Encoding
  through `User.json` drops `passwordHash` and encodes `admin` as a boolean.
- Do not put `User` itself in a contract. It encodes the database row, so it
  sends `Sensitive` fields and database encodings such as `0` and `1` for
  `BooleanSqlite`.
- `RpcTest.makeClient` does not encode values, so a sensitive field is still
  visible in its results. Test that a field stays out of the response through
  the HTTP transport. See [rpc-testing.md](rpc-testing.md).

## Tests

Give each test a fresh `SqliteClient.layer({ filename: ":memory:" })` and run
the migrations on it. Test the real repository layer against that database.

```ts
// users/user-repo.test.ts
import { SqliteClient, SqliteMigrator } from "@effect/sql-sqlite-node"
import { expect, it } from "@effect/vitest"
import { DateTime, Effect, Layer } from "effect"
import { SqlClient } from "effect/sql"
import { TestClock } from "effect/testing"
import { UserRepo } from "./user-repo.ts"

const Migrations = SqliteMigrator.layer({
  loader: SqliteMigrator.fromRecord({
    "0001_create_users": Effect.gen(function* () {
      const sql = yield* SqlClient.SqlClient
      yield* sql`CREATE TABLE users (
        id TEXT PRIMARY KEY,
        email TEXT NOT NULL UNIQUE,
        passwordHash TEXT NOT NULL,
        bio TEXT,
        admin INTEGER NOT NULL,
        createdAt TEXT NOT NULL,
        updatedAt TEXT NOT NULL
      )`
    }),
  }),
})
const Db = Migrations.pipe(Layer.provideMerge(SqliteClient.layer({ filename: ":memory:" })))
const TestLive = UserRepo.layer.pipe(Layer.provideMerge(Db))

it.effect("creates a user and finds it by id", () =>
  Effect.gen(function* () {
    const users = yield* UserRepo.Service
    yield* TestClock.setTime(1_000)
    const created = yield* users.create("ada@example.com", "hash")
    expect(DateTime.toEpochMillis(created.createdAt)).toBe(1_000)
    expect(yield* users.findById(created.id)).toEqual(created)
  }).pipe(Effect.provide(TestLive)),
)
```

Use `TestClock.setTime` or `TestClock.adjust` to control `createdAt` and
`updatedAt`. A real application loads migrations from files with
`SqliteMigrator.fromFileSystem`.
