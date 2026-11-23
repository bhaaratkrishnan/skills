# Schemas and request context

## Input and output schemas

The package owns the contract of every use case, so a host reuses it instead of restating it.

1. Each use case input is an Effect `Schema` in `src/schemas/<domain>.schema.ts`, named `<UseCase>Input` (`InviteUserInput`), with its type exported beside it:

   ```ts
   export const InviteUserInput = Schema.Struct({
     organizationId: Schema.String,
     email: Schema.String.check(Schema.isPattern(/^[^@\s]+@[^@\s]+$/)),
   });
   export type InviteUserInput = typeof InviteUserInput.Type;
   ```

2. A host decodes input once, at the edge, with the package's schema. A service method takes the decoded type and trusts its shape; it checks business rules only. A failed decode is one uniform 400 on every route: `{ code: "INVALID_INPUT" }` plus the parser's field issues. Its shape is defined once at the host and never restated per route.
3. A host whose framework takes Standard Schema (oRPC, Hono validators) passes `Schema.toStandardSchemaV1(InviteUserInput)`, so the same schema validates every host.
4. An output schema is written only when a host must serialize or document the result (an OpenAPI response, a public API) — and a screen service always writes one, because a screen's shape is itself the contract (`bff.md` #5). Otherwise the output is the inferred row or query result type.

## Request context

Who is calling is part of the environment, not an argument.

5. The host provides a **credential**: an opaque string — a session token, a JWT — extracted from wherever its transport carried it. The credential is a context service with no implementation in the package, and the package sees only the string, never a cookie, header or request object:

   ```ts
   // src/context/credential.ts
   export class Credential extends Context.Service<Credential, string>()(
     "@app/context/Credential",
   ) {}
   ```

6. Identity is resolved in the package, behind one seam: an **Auth** service maps the credential to a `CurrentActor`, with one layer per credential format (a session-table lookup; a JWT parse plus a liveness read — a signed token proves who issued it, not that the account is still live). A credential that is unknown, expired or locked fails **silently and uniformly**: the Auth layer yields no actor and the request fails with the package's single `Unauthorized` domain error before the use case runs — never a distinction between failure causes, never a crash, and never a use case running without an actor. This rule travels with the package, so every host inherits it. `Unauthorized` is a package-level error, not a member of any use case's inferred union; the host maps it through the same generic handler as every other tag (`errors.md`). A new credential format is a new layer, nothing else.

   ```ts
   // src/context/current-actor.ts
   export class CurrentActor extends Context.Service<
     CurrentActor,
     { readonly userId: string; readonly globalRole?: string }
   >()("@app/context/CurrentActor") {}
   ```

7. The actor carries **identity only**. It does not carry an organization or a role in one: a role is a property of a membership, not of a user, so no fixed actor shape can state it. The organization a use case acts on is part of its **input**, never a field of the actor; the actor's authority over that scope is read fresh by the service at the moment of the decision (see `services.md` #5). Decisions read current data, never cached claims. A present `globalRole` is one of a vocabulary the package defines — exported constants or a union type; the Auth layer maps the host's credential into it, and a value the package does not recognize is a failed resolution, never an unrecognized pass.
8. A use case that acts on behalf of someone reads `yield* CurrentActor`, so `CurrentActor` appears in its requirements and a host that forgets to provide it fails to compile. An org-scoped use case whose requirements lack `CurrentActor` is a review failure.
9. A use case run by the system (a cron job, a migration) takes no `CurrentActor`; its requirements show that it runs unauthenticated.

## Evolution

The contract changes under one rule: **evolution is scoped to who compiles with you.**

The host generates the client-facing contract — the OpenAPI document, the typed client — from the package's schemas and services; that generated artifact is what co-compiling callers build against.

A caller that builds in this repo against the generated contract is migrated by the type checker: inputs, outputs and method names may change freely, and the compile is the migration. The compile migrates **shape**; it does not migrate **meaning** — a change that keeps the shape but changes what a field means compiles green and is reviewed like a new use case — and surfaces that deploy separately from the package sequence additively even if they co-compile, because type-checking at build time does not order a deploy. A caller that does not build with you (a mobile app, a third party, a public API) makes its surface **additive-only**: new optional fields, new methods, new codes — never edits to what such a caller already reads. Error codes stay stable regardless of who compiles with you: they outlive compiled code, living in logs, dashboards and support scripts. Additive-only does not mean forever: something such a surface has outgrown is **retired, never deleted outright** — marked deprecated and kept working, its removal announced, and removed only after observed usage reaches zero; removal without that lifecycle is a breaking edit by another name.
