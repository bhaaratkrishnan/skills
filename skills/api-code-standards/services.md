# Services: use cases

A service is the only layer a host calls. Each method is one **use case**, named for what the caller wants done (`inviteUser`, `listOrganizations`, `expireStaleInvitations`), and it owns everything that use case needs: authorization, business rules, transactions, combining several reads, side effects and the domain errors it can fail with.

## Rules

1. **One service per domain, in `src/services/<domain>.service.ts`; its methods are use cases, not table operations.**
2. **A service reaches data through repos and queries only; it holds no Drizzle builder.** It may call another service, and the calls form no cycle.
3. **A use case that reads then writes based on what it read — or writes more than once — runs inside one `runTransaction`, and passes `tx` to every repo and query call in it, reads included.**
4. **Side effects outside the database (mail, storage, queues) run after the commit, and each is declared either tolerable or guaranteed.**
   - Running after the commit means a rollback never leaves a sent email behind.
   - **Tolerable** is the default: the caller can recover (resend, regenerate). When a tolerable effect fails, log the cause and still return the committed result — never a 500 after the write succeeded.
   - **Guaranteed** is declared at the use case, never improvised per call site, and has a hard definition: an intent record is committed atomically with the data change, and a crash-recoverable dispatcher claims and dispatches it. Retryable failures are retried without discarding the record; permanent failures follow a declared escalation policy.
   - Delivery is at-least-once; make no exactly-once claim. Handlers tolerate duplicates, with an idempotency key or downstream deduplication where a careful handler is not enough.
   - A process can die between the commit and any post-commit call, so a mechanism that lives entirely after the commit (a queue that must first be enqueued to, a retrying mailer that must first be called) still loses the effect. Only the atomic intent record closes that window; a guarantee backed by anything less is tolerable wearing a guarantee's name.
5. **The service is the only authority for authorization: it reads membership and role fresh, from the actor's identity and the input's scope, at the moment of the decision.**
   - Never from cached claims, and never only from a host guard.
   - Host guards may reject early, but no request is correct because a guard allowed it and none is safe because one is missing.
   - The authorization travels with the use case, so every host — HTTP, worker or script — enforces the same rules. The credential-to-actor seam is `schemas.md` #6.
6. **A `find*` result the use case requires becomes a domain error here:** a `null` becomes `OrganizationNotFound`, never a `null` returned to the host.
7. **Each method is written with `Effect.fn("<Service>.<method>")` and ends with `Effect.catchTag("DatabaseError", Effect.die)`.** The name becomes its tracing span; the catch leaves only domain errors in its error channel (see `errors.md`).
8. **Inputs are already-decoded schema types; the service validates business rules, not shapes.** Outputs are rows, query results or schema types.
9. **The service's shape is inferred from `make`, as for repos and queries; write no separate interface.**
10. **A use case that spans domains lives in the service that owns its root — the record whose existence is the point of the use case — and runs one transaction there.**
    - The root service may call any domain's repo inside its transaction.
    - Other domains contribute rules through their home — a pure rule function, never a copy pasted into the caller. An invariant expressed in more than one file has been copied: move it to the domain it is about.
    - Never nest a `runTransaction` inside another. A cross-domain write that need not be atomic is a separate use case called after the commit.
    - Name the service file for the use case's noun, not the tables it touches.
11. **A use case reads time through Effect's `Clock`, never `new Date()`, and never decides anything from `createdAt`/`updatedAt`.**
    - `Clock` lets expiry and staleness logic run on injected time in tests and real time in production.
    - `createdAt`/`updatedAt` are the database's record of history — audit and display data.
    - A decision that needs a time compares `Clock` against a value the application itself wrote (an `expiresAt` set from `Clock`, a deadline in the input).
12. **An invariant that must hold under concurrency is enforced by the database, not by the transaction.**
    - A transaction makes a write atomic; it does not make it correct. A check-then-act invariant — count the members, then insert — breaks under two concurrent requests running the same check.
    - Pick the mechanism at the invariant: a unique or check constraint, a conditional write (`UPDATE ... WHERE seats > used`), a row lock (a `find*ForUpdate` read, `data-access.md` #18), or a serializable transaction. The choice is part of the use case's design, not an accident of what Drizzle emitted.
    - An invariant with none of these is advisory: its author has decided it may be violated.
13. **In a scoped use case, every read and write carries the input's scope into the statement.**
    - Either the repo or query method takes the scope as a parameter and filters by it (`WHERE attempt_id = ? AND organization_id = ?`), or a row fetched by a global id is verified against the scope before use.
    - A scoped use case calling an unscoped method is a review failure: an authorization check that passes while the fetch ignores the scope is a cross-tenant leak.

## Example

```ts
// src/services/invitation.service.ts
const makeInvitationService = Effect.gen(function* () {
  const { runTransaction } = yield* Database;
  const organizationRepo = yield* OrganizationRepo;
  const memberRepo = yield* OrganizationMemberRepo;
  const invitationRepo = yield* InvitationRepo;
  const memberStats = yield* OrganizationMemberStatsQuery;
  const mailer = yield* Mailer;

  return {
    inviteUser: Effect.fn("InvitationService.inviteUser")(
      function* (input: InviteUserInput) {
        const actor = yield* CurrentActor;

        const invitation = yield* runTransaction((tx) =>
          Effect.gen(function* () {
            // The row lock serializes concurrent invites to this org, so the seat count below cannot race.
            const organization = yield* organizationRepo.findOrganizationByIdForUpdate(input.organizationId, tx);
            if (!organization) return yield* new OrganizationNotFound({ organizationId: input.organizationId });

            const membership = yield* memberRepo.findOrganizationMemberByUserAndOrganizationId(
              { userId: actor.userId, organizationId: organization.id },
              tx,
            );
            if (!membership || membership.role !== "admin") {
              return yield* new Forbidden({ action: "inviteUser" });
            }

            const members = yield* memberStats.countOrganizationMembersByOrganizationId(organization.id, tx);
            if (members >= organization.seats) return yield* new SeatLimitReached({ seats: organization.seats });

            return yield* invitationRepo.createInvitation(
              { data: { email: input.email, organizationId: organization.id, invitedByUserId: actor.userId } },
              tx,
            );
          }),
        );

        yield* mailer.sendInvitation(invitation).pipe(
          Effect.catchCause((cause) => Effect.logError("Failed to send invitation email", cause)),
        );
        return invitation;
      },
      Effect.catchTag("DatabaseError", Effect.die),
    ),
  };
});

export class InvitationService extends Context.Service<InvitationService>()(
  "@app/services/InvitationService",
  { make: makeInvitationService },
) {
  static readonly layer = Layer.effect(InvitationService, InvitationService.make);
}
```

The inferred type of `inviteUser` is the use case's contract: `(input: InviteUserInput) => Effect<Invitation, OrganizationNotFound | Forbidden | SeatLimitReached, CurrentActor>`. It names every failure a host must handle and the context it must provide.

## Paged lists

A paged list use case runs the list method and its count (see `data-access.md` #19) and returns `{ items, total }`.
