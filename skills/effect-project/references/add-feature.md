# Add a feature

Read the existing feature nearest to the requested behavior. Follow its
package and naming conventions. These steps describe the dependencies between
changes, not a requirement to create every file for every feature.

## Place the contract and implementation

Define the feature's data and errors using the Effect skill's
[schema guidance](../../effect/references/schema-errors.md). In a full-stack
app, shared contracts belong in domain. In a CLI or server-only app, keep them
local until another consumer needs them.

Implement business rules and persistence with the Effect skill's
[service conventions](../../effect/references/services-layers.md). Keep the
implementation in the app that owns the resource. Add a migration if the
persisted representation changes.

## Connect the entry points

For a CLI, connect the command to its service and compose the required layers.
For a server, connect the handler to the service and register it with the
existing route layer.

For a full-stack RPC feature, make the following edits:

1. Define the RPCs in domain. Add a new group to `AppRpcs` and export its namespace from the package barrel. Use [Effect RPC](../../effect/references/rpc.md) for the API.
2. Implement the service or repository in the server feature folder. Put thin handlers beside it.
3. Register the handler layer with the existing route layer. Reuse the database, auth, serializer, and transport composition. See [app-wiring.md](app-wiring.md).
4. Add query and mutation atoms in the web feature folder. Reuse `ApiClient.Service`. Use [RPC client atoms](../../effect/references/rpc-atoms.md) for invocation shapes and invalidation.
5. Compose the component and route. Render the feature's loading, success, and relevant failure states.

A new procedure in an existing group usually needs no new listener or client
service. Add files only when the feature needs them.

## Check the change

Follow [verification.md](verification.md). Run the contract type check across
all affected packages, then tests from the package directories. If a public
schema changed, check existing clients and persisted data before assuming a
simultaneous deployment.

If a generator already exists, inspect its output and run the same checks after
editing the generated files. Do not add a generator or shared package just to
complete one feature.
