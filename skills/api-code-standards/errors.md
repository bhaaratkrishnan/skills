# Errors

Errors split by who can act on them:

- **Domain errors** are failures a caller can respond to: a missing record, a full seat count, a forbidden action. They are typed and form part of a service method's contract.
- **Defects** are failures no caller can fix: a database outage, a bug. They travel as defects, are logged at the host and answered with a generic server error.

## Rules

1. **Each domain error is its own `Schema.TaggedError` class in `src/errors/<domain>.errors.ts`, named for the failure in the domain's words, with a stable `code`, the fields a caller needs, and `status` and `message` getters:**

   ```ts
   export class OrganizationNotFound extends Schema.TaggedError<OrganizationNotFound>()("OrganizationNotFound", {
     code: Schema.tag("ORGANIZATION_NOT_FOUND"),
     organizationId: Schema.String,
   }) {
     get status() { return 404 as const; }
     get message() { return `Organization ${this.organizationId} was not found`; }
   }

   export class SeatLimitReached extends Schema.TaggedError<SeatLimitReached>()("SeatLimitReached", {
     code: Schema.tag("SEAT_LIMIT_REACHED"),
     seats: Schema.Number,
   }) {
     get status() { return 409 as const; }
     get message() { return `All ${this.seats} seats are taken`; }
   }
   ```

   A service constructs it with its fields only (`new SeatLimitReached({ seats })`); `Schema.tag` fills `code`, and every raise of the error carries the same status and message.
2. **`_tag` is the error's identity inside Effect (`catchTag`, narrowing); `code` is its identity on the wire, in `SCREAMING_SNAKE_CASE` from the tag, and never changes once shipped** — clients depend on it.
3. **`status` is the HTTP status the failure means, returned as a literal (`404 as const`).** It is protocol metadata: a host on any HTTP framework uses it as is, and a host with no HTTP (a worker, a test) ignores it.
4. **`message` is one developer-facing English sentence built from the fields.** It appears in logs and traces and is the default text in a response; a client picks what to show its users from `code`.
5. **One class per distinct failure, so each code maps to one response and a method's error union lists every failure by name.** A generic error class with a `code` string erases that union.
6. **Authorization failures are the deliberate exception to #5: one `Unauthorized` and one `Forbidden({ action })` for the whole package, in `src/errors/auth.errors.ts`.**
   - `Unauthorized` (no actor) is raised by the Auth seam before any use case runs, so it is in no use case's inferred union (`schemas.md` #6).
   - `Forbidden` (an actor without authority) is raised by use cases and appears in their unions, like any domain error.
   - A client responds to every refusal the same way, and splitting `Forbidden` by reason (not a member, wrong role, wrong org) would tell an attacker which check failed. `action` names the use case for logs; it never says why.

   ```ts
   export class Unauthorized extends Schema.TaggedError<Unauthorized>()("Unauthorized", {
     code: Schema.tag("UNAUTHORIZED"),
   }) {
     get status() { return 401 as const; }
     get message() { return "No valid credential was provided"; }
   }

   export class Forbidden extends Schema.TaggedError<Forbidden>()("Forbidden", {
     code: Schema.tag("FORBIDDEN"),
     action: Schema.String,
   }) {
     get status() { return 403 as const; }
     get message() { return `Not allowed to ${this.action}`; }
   }
   ```

7. **Services fail with `return yield* new SeatLimitReached({ seats })`; nothing in the package throws.**
8. **`DatabaseError` stays typed in repos and queries, and every service method ends with `Effect.catchTag("DatabaseError", Effect.die)`,** so a service can still react to one (a unique violation it turns into `EmailAlreadyInvited`) and only domain errors reach the host.
9. **The package exports every domain error and their union, kept in `src/errors/index.ts`, including the package-level `Unauthorized` and `Forbidden`:**

   ```ts
   export type DomainError =
     | Unauthorized
     | Forbidden
     | OrganizationNotFound
     | SeatLimitReached
     | InvitationExpired;
   ```

## At the host

10. **One generic handler maps every domain error, and the host declares no error names of its own** — that is the condition under which adding an error needs no change at the host:

    ```ts
    const toErrorResponse = (error: DomainError) => ({
      status: error.status,
      body: { ...error, message: error.message }, // { _tag, code, ...fields, message }
    });
    ```

    The getters are left out of JSON, so the body adds `message` explicitly.
11. **Where a framework requires per-route error documentation (OpenAPI), it is derived from the method's inferred error union, never written by hand.** A route may document a tag the union contains, and nothing else.
12. **Defects are caught once by the host's runtime wrapper, logged with their cause, and answered with a generic server error.**
