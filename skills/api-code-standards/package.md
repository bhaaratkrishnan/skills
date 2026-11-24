# Package: the self-contained API

The API package holds every use case of the backend: data access, business rules, transactions and side effects. A **host** (an HTTP server, an RPC router, a worker, a script, a test) is a thin shell around it that decodes input, calls a service and maps the result to its own output. Any framework can be the host, so the package never depends on one.

## Layers inside the package

Dependencies point down only:

```
services   use cases: orchestration, transactions, authorization, domain errors
   ↓
repos / queries   data access (see data-access.md)
   ↓
Database   connection pool, runQuery, runTransaction (see database.md)
```

Beside them sit `errors/` (domain errors), `schemas/` (input and output contracts), `context/` (request context such as the current actor), `config/` and the adapters for external systems (mail, storage, queues).

## Boundary

1. **The package imports no HTTP or RPC framework, and nothing in it knows about headers, cookies or request objects.** The one piece of HTTP it carries is the `status` on each domain error (see `errors.md`).
2. **The public surface is the services, the composed `ApiLayer`, the domain errors, the schemas and the context services, plus the row and result types services return.** A type may cross the boundary; an implementation may not. Repos, queries and `Database` stay internal, so every caller goes through a use case.
3. **The package entry exports exactly that surface; the `exports` map in `package.json` lists no internal folder.**

## Composition

4. **Every service class carries `static readonly layer`, and the package exports one `ApiLayer` that provides every service on one pool — except Auth, which is provided per request.** A host provides one layer and gets every use case:

   ```ts
   // src/index.ts
   export const ApiLayer = Layer.mergeAll(UserService.layer, InvitationService.layer).pipe(
     Layer.provide(Layer.mergeAll(RepoLayer, QueryLayer, Mailer.layer)),
     Layer.provide(Database.layer),
   );
   ```

   Auth resolves `CurrentActor` from the per-request `Credential`, so it is provided by `provideRequestContext` (see below); merging it into `ApiLayer` would resolve one actor at startup, with no credential, for every request.

5. **Configuration is read through Effect `Config` inside the layer that needs it; code reads `process.env` nowhere.** `Database.layer` reads the connection string, `Mailer.layer` its credentials, and a host supplies config through a `ConfigProvider` when it needs to.
6. **Each external system (mail, object storage, payment, queue) is its own service with a real layer and, where tests need it, a `layerTest`; services call the adapter, never the vendor SDK.** An adapter's contract states what happens when its system is down — retry, a domain error, or a defect — matching the side effect's declared loss rule (`services.md` #4), never left as an accident.

## What a host does

A host builds one runtime from `ApiLayer` at startup, then for each call: decodes the input with the package's schema (a failure is the uniform 400 of `schemas.md` #2), provides the credential string from its transport (the package resolves the actor through its Auth seam — see `schemas.md` #6), runs the service method and maps each error tag to its own response.

```ts
// HTTP host (Hono)
const runtime = ManagedRuntime.make(ApiLayer);

app.post("/invitations", async (c) => {
  const decoded = Schema.decodeUnknownExit(InviteUserInput)(await c.req.json());
  if (Exit.isFailure(decoded)) return invalidInput(c, decoded.cause); // the one 400 shape, defined once at the host

  const exit = await runtime.runPromiseExit(
    InvitationService.use((s) => s.inviteUser(decoded.value)).pipe(
      provideRequestContext(c.req.header("authorization") ?? ""),
    ),
  );
  return toResponse(c, exit); // domain error → its status, code and message; defect → 500
});
```

`provideRequestContext` is exported by the package: it provides the credential service and resolves `CurrentActor` through the Auth layer, so a host hands over one string and nothing else.

```ts
// Script or worker host: same package, no HTTP
await ManagedRuntime.make(ApiLayer).runPromise(
  InvitationService.use((s) => s.expireStaleInvitations()),
);
```

## Tests

7. **A use case is tested against the service with its dependencies replaced by layers, and the actor provided directly as `CurrentActor`.**
   - `Layer.mock(UserRepo)({ findUserById: … })` covers the methods the test exercises; `layerTest` covers adapters.
   - Providing `CurrentActor` directly keeps the test about the use case; the credential-to-actor path is covered once, by the Auth layer's own tests (`schemas.md` #6).

   ```ts
   it.effect("rejects an invite when the seats are full", () =>
     Effect.gen(function* () {
       const service = yield* InvitationService;
       const error = yield* Effect.flip(service.inviteUser({ organizationId: "o1", email: "a@b.co" }));
       expect(error._tag).toBe("SeatLimitReached");
     }).pipe(
       Effect.provide(InvitationService.layer),
       Effect.provide(Layer.mergeAll(fakeRepos({ seats: 0, role: "admin" }), Mailer.layerTest)),
       Effect.provideService(CurrentActor, { userId: "u1" }),
     ),
   );
   ```

8. **Repos and queries are tested against a real, disposable database — a container or a reset schema.** A mock confirms the test author's assumption, not the statement: joins, left-join nullability and the list/count filter match are verified only by running the SQL. The mechanism is the repo's choice; the rule is not.
9. **Every error tag in a use case's inferred error union has a test that triggers it.** The inferred contract is the test oracle: `OrganizationNotFound | Forbidden | SeatLimitReached` means three tests minimum, and a union member with no test is a coverage hole visible in the type.
