# Schemas and request context

## Input and output schemas

The package owns the contract of every use case, so a host reuses it instead of restating it.

1. **Each use case input is an Effect `Schema` in `src/schemas/<domain>.schema.ts`, named `<UseCase>Input`, with its type exported beside it:**

   ```ts
   export const InviteUserInput = Schema.Struct({
     organizationId: Schema.String,
     email: Schema.String.check(Schema.isPattern(/^[^@\s]+@[^@\s]+$/)),
   });
   export type InviteUserInput = typeof InviteUserInput.Type;
   ```

2. **A host decodes input once, at the edge, with the package's schema, and a failed decode is one uniform 400 on every route.**
   - The service takes the decoded type and trusts its shape; it checks business rules only.
   - The 400 body is `{ code: "INVALID_INPUT" }` plus the parser's field issues. Its shape is defined once at the host and never restated per route.
3. **A host whose framework takes Standard Schema (oRPC, Hono validators) passes `Schema.toStandardSchemaV1(InviteUserInput)`,** so the same schema validates every host.
4. **An output schema is written only when a host must serialize or document the result (an OpenAPI response, a public API) — and a screen service always writes one** (`bff.md` #5), because a screen's shape is itself the contract. Otherwise the output is the inferred row or query result type.

## Request context

Who is calling is part of the environment, not an argument.

5. **The host provides a credential — an opaque string, such as a session token or a JWT — and the package sees only that string, never a cookie, header or request object.** The credential is a context service with no implementation in the package:

   ```ts
   // src/context/credential.ts
   export class Credential extends Context.Service<Credential, string>()(
     "@app/context/Credential",
   ) {}
   ```

6. **Identity is resolved in the package behind one Auth seam, and every failed resolution is the same `Unauthorized` before the use case runs.**
   - The Auth service maps the credential to a `CurrentActor`, with one layer per credential format: a session-table lookup, or a JWT parse plus a liveness read (a signed token proves who issued it, not that the account is still live). A new credential format is a new layer, nothing else.
   - An unknown, expired or locked credential fails **silently and uniformly**: no distinction between causes, never a crash, and never a use case running without an actor. This travels with the package, so every host inherits it.
   - `Unauthorized` is package-level, in no use case's inferred union; the host maps it through the same generic handler as every other tag (`errors.md` #6, #10).
   - The Auth layers have their own tests (credential → actor, each failure cause → `Unauthorized`), so use case tests can provide `CurrentActor` directly (`package.md` #7).

   ```ts
   // src/context/current-actor.ts
   export class CurrentActor extends Context.Service<
     CurrentActor,
     { readonly userId: string; readonly globalRole?: string }
   >()("@app/context/CurrentActor") {}
   ```

7. **The actor carries identity only; the organization a use case acts on is part of its input, and the actor's authority over it is read fresh by the service** (`services.md` #5).
   - A role is a property of a membership, not of a user, so no fixed actor shape can state it. Decisions read current data, never cached claims.
   - A present `globalRole` is one of a vocabulary the package defines (exported constants or a union type). The Auth layer maps the host's credential into it, and a value the package does not recognize is a failed resolution, never an unrecognized pass.
8. **A use case that acts on behalf of someone reads `yield* CurrentActor`, so a host that forgets to provide it fails to compile.** An org-scoped use case whose requirements lack `CurrentActor` is a review failure.
9. **A use case run by the system (a cron job, a migration) takes no `CurrentActor`;** its requirements show that it runs unauthenticated.

## Evolution

The contract changes under one principle: **evolution is scoped to who compiles with you.** The host generates the client-facing contract — the OpenAPI document, the typed client — from the package's schemas and services; that generated artifact is what co-compiling callers build against.

10. **A caller that builds in this repo against the generated contract is migrated by the type checker:** inputs, outputs and method names may change freely, and the compile is the migration.
11. **A change that keeps the shape but changes a field's meaning is reviewed like a new use case.** The compile migrates shape, not meaning; such a change compiles green.
12. **A surface that deploys separately from the package changes additively, even if it co-compiles.** Type-checking at build time does not order a deploy.
13. **A caller that does not build with you (a mobile app, a third party, a public API) gets an additive-only surface:** new optional fields, new methods, new codes — never edits to what such a caller already reads.
14. **Error codes stay stable regardless of who compiles with you.** They outlive compiled code, living in logs, dashboards and support scripts.
15. **Something an additive-only surface has outgrown is retired, never deleted outright:** marked deprecated and kept working, its removal announced, and removed only after observed usage reaches zero. Removal without that lifecycle is a breaking edit by another name.
