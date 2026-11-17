# Schemas and request context

## Input and output schemas

The package owns the contract of every use case, so a host reuses it instead of restating it.

1. Each use case input is an Effect `Schema` in `src/schemas/<domain>.schema.ts`, named `<UseCase>Input` (`InviteUserInput`), with its type exported beside it:

   ```ts
   export const InviteUserInput = Schema.Struct({
     email: Schema.String.check(Schema.isPattern(/^[^@\s]+@[^@\s]+$/)),
   });
   export type InviteUserInput = typeof InviteUserInput.Type;
   ```

2. A host decodes input once, at the edge, with the package's schema. A service method takes the decoded type and trusts its shape; it checks business rules only.
3. A host whose framework takes Standard Schema (oRPC, Hono validators) passes `Schema.toStandardSchemaV1(InviteUserInput)`, so the same schema validates every host.
4. An output schema is written only when a host must serialize or document the result (an OpenAPI response, a public API). Otherwise the output is the inferred row or query result type.

## Request context

Who is calling is part of the environment, not an argument.

5. The caller's identity is a context service with no implementation in the package; each host provides it per request:

   ```ts
   // src/context/current-actor.ts
   export class CurrentActor extends Context.Service<
     CurrentActor,
     { readonly userId: string; readonly organizationId: string }
   >()("@app/context/CurrentActor") {}
   ```

6. A use case that acts on behalf of someone reads `yield* CurrentActor`, so `CurrentActor` appears in its requirements and a host that forgets to provide it fails to compile.
7. A use case run by the system (a cron job, a migration) takes no `CurrentActor`; its requirements show that it runs unauthenticated.
