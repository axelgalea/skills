# Add an RPC feature

Read the existing feature nearest to the requested behavior. Follow its
package and naming conventions. These steps describe the dependencies between
changes, not a requirement to create every file for every RPC.

1. Define the contract in domain. Add the input and output schemas, branded IDs, and failures the caller can act on. Add unique procedure tags. Apply the existing middleware. Merge a new group into `AppRpcs` and export new domain namespaces from the package barrel. See [rpc.md](rpc.md).
2. Implement the service or repository on the server. Keep persistence errors there. Decode database values there. Pass the authenticated user to operations that need ownership checks. Use a transaction when the operation's writes must commit together.
3. Add thin handlers with `Rpcs.toLayer`. Acquire long-lived dependencies while constructing the layer. Read middleware-provided identity inside the handler. Pass declared failures through and map other failures at this boundary.
4. Register the handler layer with the server's route layer. Reuse the existing database, auth, serializer, and transport layers. A new procedure in an existing group usually needs no new server entry point.
5. Add module-level query or mutation atoms in the web feature. Use the existing `ApiClient.Service`. Pick reactivity keys for every query affected by the mutation. Use `Atom.family` for parameterized queries. See [web.md](web.md).
6. Compose the component and route. Render loading, success, and relevant failure states. Keep RPC calls and state ownership in the atoms. Do not add another state library.
7. Verify the behavior with the real handler and repository. Test a declared failure as well as the successful operation. Check auth and ownership when the change affects them. Test mutation invalidation if that is part of the change. See [testing.md](testing.md).

Run the contract type check across domain, server, and web. Run tests from the
affected package directories. If a public schema changed, check existing
clients and persisted data before assuming a simultaneous deployment.

Do not add a stream, generator, shared package, migration, or new auth mechanism
unless the requested feature needs it. If a generator already exists, inspect
its output and run the same checks after editing the generated files.
