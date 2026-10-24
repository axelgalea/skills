# Verify a project change

Use the Effect skill's [testing reference](../../effect/references/testing.md)
for test services, layer lifetime, configuration, and synchronization. Use
[RPC tests](../../effect/references/rpc-testing.md) for in-memory clients,
HTTP serialization, and substituting an AtomRpc client. This reference decides
which application boundaries the change needs to check.

Run the project's check command. Run package tests from the changed package's
directory. CI should run the same type, lint, and behavior checks.

## CLI or server-only changes

Check the entry point with its real service layers. Test the requested behavior
and failures the caller can act on. Run persistence changes against a fresh
database with the real migrations. Exercise an external transport when its
wiring or encoding changes.

## Full-stack changes

Type-check domain, server, and web together after a shared contract change.
This finds stale handler definitions and client call sites. It does not prove
that independently deployed clients accept the new wire format.

Test server handlers with their real repository and a fresh database. A fixed
identity test layer checks handler behavior, but it does not check authentication.
Use the real middleware for accepted and rejected credentials. Check a second
user's behavior when the change affects ownership.

Test web atoms with a controlled RPC implementation at the server boundary.
The web package must not import server implementation modules. Assert the
actual result after a mutation when invalidation is part of the change.

A route or serializer change needs a request through the real HTTP route and
serializer. Serve the route layer with an ephemeral test listener. Keep the
normal fixed-port listener out of that graph. A streaming change needs checks
for the items, declared failures, and any lifetime behavior it changes.

For a new web project, start development and exercise the example feature in
the browser. Confirm that its query loads and its mutation updates the view.
Report any boundary that the available environment prevented you from checking.
