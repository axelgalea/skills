---
name: effect-authorization
description: Role-based authorization for Effect v4 applications. Roles, resources and CRUD operations are Schema literals in code, the database stores which role has which permission and which role a user holds, and every other rule is a composable policy function in code. Use when adding or changing roles, permissions, access checks, policies, or 401 and 403 errors in a project that uses `effect`. Use with the effect skill.
---

# Effect authorization

Use this skill with the `effect` skill from this repository. That skill owns
Schema, `Model.Class` and SQL conventions. This skill owns how access is
modeled and checked.

The design splits access into two parts:

- The database stores grants. A grant gives a role one CRUD operation on one
  resource type, such as `"task:read"`. Administrators can change grants
  without a deploy.
- Code holds every other rule as a policy. A policy is a typed function, so a
  rule such as "a member of the assigned team can change the status" is
  checked by TypeScript and tested without a database.

Do not store conditions, deny rows, or action-specific permissions such as
`"task:assign"` in the database. Write those rules as policies.

## Model

Put the model in the shared domain package, in `authorization.ts`, exported as
a namespace. The server and every frontend import it.

```ts
import { Effect, Option, Schema } from "effect"
import { Model } from "effect/schema"
import { UserId } from "./users.ts"
import { WorkspaceId } from "./workspaces.ts"

// The administrator passes every `permission` policy, so the database stores
// no grants for it.
export const Role = Schema.Literals(["administrator", "manager", "member"])
export type Role = typeof Role.Type

export const Resource = Schema.Literals(["workspace", "project", "task", "user"])
export type Resource = typeof Resource.Type

export const Operation = Schema.Literals(["create", "read", "update", "delete"])
export type Operation = typeof Operation.Type

// "task:read", "project:delete", and every other pair of the two lists.
export const Permission = Schema.TemplateLiteral([Resource, ":", Operation])
export type Permission = typeof Permission.Type

// A user holds one role in each workspace they belong to.
export class RoleAssignment extends Model.Class<RoleAssignment>("RoleAssignment")({
  userId: UserId,
  workspaceId: WorkspaceId,
  role: Role,
  createdAt: Model.DateTimeInsertFromDate,
}) {}

// A row grants the permission to every user who holds the role.
export class RolePermission extends Model.Class<RolePermission>("RolePermission")({
  role: Role,
  permission: Permission,
}) {}
```

Derive `Permission` with `Schema.TemplateLiteral`. Do not list every pair by
hand. Adding a resource or an operation adds its permissions, and decoding a
row rejects a string outside the pairs.

If users belong to one tenant only, drop the tenant id from `RoleAssignment`
or put `role` on the user. Keep the same `Role` schema in both places.

## Policies

An actor is a signed-in user in one tenant, with their role and the
permissions of that role. `Option.none()` is a guest.

```ts
export interface Actor {
  readonly userId: UserId
  readonly role: Role
  readonly permissions: ReadonlySet<Permission>
}

export type Policy = (actor: Option.Option<Actor>) => boolean

export const isSignedIn: Policy = Option.isSome
export const isGuest: Policy = Option.isNone

export const role =
  (role: Role): Policy =>
  (actor) =>
    Option.exists(actor, (actor) => actor.role === role)

export const permission =
  (permission: Permission): Policy =>
  (actor) =>
    Option.exists(
      actor,
      (actor) => actor.role === "administrator" || actor.permissions.has(permission),
    )

// Passes when the actor is one of the users, such as the members of a team.
export const userIn =
  (userIds: ReadonlyArray<UserId>): Policy =>
  (actor) =>
    Option.exists(actor, (actor) => userIds.includes(actor.userId))

export const any =
  (...policies: ReadonlyArray<Policy>): Policy =>
  (actor) =>
    policies.some((policy) => policy(actor))

export const all =
  (...policies: ReadonlyArray<Policy>): Policy =>
  (actor) =>
    policies.every((policy) => policy(actor))

export const not =
  (policy: Policy): Policy =>
  (actor) =>
    !policy(actor)
```

A policy reads only the actor. Pass any other data it needs as an argument of
a function that returns the policy, such as `userIn(memberIds)`. Keep
policies pure. The caller loads the data first.

## Errors and `authorize`

```ts
// The request has no valid session.
export class Unauthenticated extends Schema.TaggedError<Unauthenticated>()(
  "Authorization.Unauthenticated",
  {},
  { httpApiStatus: 401 },
) {}

// The session is valid, but the policy does not pass.
export class Forbidden extends Schema.TaggedError<Forbidden>()(
  "Authorization.Forbidden",
  {},
  { httpApiStatus: 403 },
) {}

// A guest that fails the policy gets `Unauthenticated`, so the client can ask
// them to sign in. A signed-in user gets `Forbidden`.
export const authorize = (
  policy: Policy,
  actor: Option.Option<Actor>,
): Effect.Effect<void, Unauthenticated | Forbidden> => {
  if (policy(actor)) return Effect.void
  return Option.isNone(actor) ? Effect.fail(new Unauthenticated()) : Effect.fail(new Forbidden())
}

export * as Authorization from "./authorization.ts"
```

## Resource policies

Put a rule that belongs to one resource in that resource's file, next to its
model and its other rules. Name the file's export `policy`.

```ts
// tasks.ts
// Who may take each action on a task whose team has `teamUserIds` as members.
// A member of the team can change the status without "task:update".
export const policy = (
  action: Action["_tag"],
  teamUserIds: ReadonlyArray<UserId>,
): Authorization.Policy => {
  const update = Authorization.permission("task:update")
  const policies: Record<Action["_tag"], Authorization.Policy> = {
    Assign: update,
    Edit: update,
    Comment: Authorization.permission("task:read"),
    ChangeStatus: Authorization.any(update, Authorization.userIn(teamUserIds)),
  }
  return policies[action]
}
```

The `Record` over the action tags makes TypeScript reject a new action that
has no policy.

## Where a policy lives

Every app can build policies from the same helpers:

- The server enforces a policy and a frontend shows or hides a control with
  it: put it in the resource's domain file, so both use one rule.
- Only the server needs it, such as an admin export: put it in the server.
- Only the UI needs it: put it in the frontend. The server still enforces its
  own check, because a client can call the API without the UI.

## Server

The server builds the actor for each request:

1. Read the session. No session means `Option.none()`.
2. Load the user's `RoleAssignment` for the tenant that owns the target. If
   the session is valid but the user has no role there, fail with `Forbidden`
   yourself. Passing `Option.none()` would make `authorize` answer
   `Unauthenticated` to a signed-in user.
3. Load the grants of that role and build `permissions` as a `Set`.
4. Load what the policy needs, such as team members, then run
   `yield* Authorization.authorize(Tasks.policy(action._tag, memberIds), actor)`.

The tables copy the literals in CHECK constraints:

```sql
CREATE TABLE role_assignments (
  user_id uuid NOT NULL REFERENCES users (id) ON DELETE CASCADE,
  workspace_id uuid NOT NULL REFERENCES workspaces (id),
  role text NOT NULL CHECK (role IN ('administrator', 'manager', 'member')),
  created_at timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (user_id, workspace_id)
);

CREATE TABLE role_permissions (
  role text NOT NULL CHECK (role IN ('administrator', 'manager', 'member')),
  permission text NOT NULL CHECK (
    permission ~ '^(workspace|project|task|user):(create|read|update|delete)$'
  ),
  PRIMARY KEY (role, permission)
);
```

A change to a literal list needs a new migration that updates the CHECK
constraint. Seed the starting grants in a migration. Do not seed grants for
the administrator.

## Tests

Test policies as plain function calls in the test file of the module that
defines them. Build actors inline:

```ts
const actor = (role: Authorization.Role, permissions: ReadonlyArray<Authorization.Permission>) =>
  Option.some<Authorization.Actor>({ userId, role, permissions: new Set(permissions) })

assert.isTrue(Tasks.policy("ChangeStatus", [userId])(actor("member", ["task:read"])))
assert.isFalse(Tasks.policy("Assign", [userId])(actor("member", ["task:read"])))
```

Test `authorize` once for each error with `Effect.flip`. Test the grant tables
against the real database through the migrations, not with mocks.
