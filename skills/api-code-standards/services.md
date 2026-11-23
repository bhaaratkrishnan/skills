# Services: use cases

A service is the only layer a host calls. Each method is one **use case**, named for what the caller wants done (`inviteUser`, `listOrganizations`, `expireStaleInvitations`), and it owns everything that use case needs: authorization, business rules, transactions, combining several reads, side effects and the domain errors it can fail with.

## Rules

1. One service per domain, in `src/services/<domain>.service.ts`. Its methods are use cases, not table operations.
2. A service reaches data through repos and queries only; it holds no Drizzle builder. It may call another service, and the calls form no cycle.
3. A use case that reads and then writes based on what it read — or writes more than once — runs inside one `runTransaction`, and passes `tx` to every repo and query call in it, reads included.
4. Direct side effects outside the database (mail, storage, queues) run after the transaction commits, so a rollback never leaves a sent email behind. Their possible loss is a named decision, never an accident: each side effect is either **tolerable** (the caller can recover: resend, regenerate) or **guaranteed**. Tolerable is the default; a guarantee is declared at the use case, not improvised per call site. **Guaranteed** has a hard definition: an intent record is committed atomically with the data change, and a crash-recoverable dispatcher claims it and dispatches it — retryable failures are retried without discarding the record, permanent failures follow a declared escalation policy, and handlers tolerate duplicates: assume at-least-once delivery and make no exactly-once claim, with an idempotency key or downstream deduplication where a careful handler is not enough. A process can die between the commit and any post-commit call, so a mechanism that lives entirely after the commit (a queue that must first be enqueued to, a retrying mailer that must first be called) still loses the effect; only the atomic intent record closes that window. A guarantee backed by anything less is tolerable wearing a guarantee's name.
5. Authorization runs in the service, which is its **only authority**: membership and role are read fresh from the actor's identity and the input's scope at the moment of the decision — never from cached claims, and never only from a host guard. Host guards may reject early, but no request is correct because a guard allowed it and none is safe because one is missing; the authorization travels with the use case, so every host — HTTP, worker or script — enforces the same rules. The credential-to-actor seam is `schemas.md` #6.
6. `find*` results that the use case requires are turned into domain errors here: a `null` becomes `OrganizationNotFound`, never a `null` returned to the host.
7. Each method is written with `Effect.fn("<Service>.<method>")`, which names its tracing span, and ends with `Effect.catchTag("DatabaseError", Effect.die)` so its error channel holds domain errors only (see `errors.md`).
8. Inputs are already-decoded schema types; outputs are rows, query results or schema types. The service validates business rules, not shapes.
9. The shape is inferred from `make`, as for repos and queries; write no separate interface.
10. A use case that spans domains lives in the service that owns its **root**: the record whose existence is the point of the use case. That service runs the single transaction and may call any domain's repo inside it. Other domains contribute rules through their home — a pure rule function, never a copy pasted into the caller — and their records are reached through their repo, which the root service may call inside its transaction. Never nest a `runTransaction` inside another; a cross-domain write not needed atomically is a separate use case called after the commit. An invariant expressed in more than one file has been copied — move it to the domain it is about. Name the service file for the use case's noun, not the tables it touches.
11. A use case reads time through Effect's `Clock`, never `new Date()`, so expiry and staleness logic runs on injected time in tests and real time in production. Decision time and record timestamps never mix: `createdAt`/`updatedAt` are the database's record of history — audit and display data — and a use case never reads them to decide anything. A decision that needs a time compares `Clock` against a value the application itself wrote (an `expiresAt` set from `Clock`, a deadline in the input).
12. A transaction makes a write atomic; it does not make it correct. A check-then-act invariant — count the members, then insert — breaks under two concurrent requests running the same check. An invariant that must hold under concurrency is enforced by the database, not by the transaction: a unique or check constraint, a conditional write (`UPDATE ... WHERE seats > used`), a lock, or a serializable transaction — chosen at the invariant, and the choice is part of the use case's design, not an accident of what Drizzle emitted. An invariant with none of these is advisory, and the use case's author has decided it may be violated.
13. The scope reaches the data, not just the check. In a scoped use case, every read and write carries the input's scope: either the repo/query method takes the scope as a parameter and the statement filters by it (`WHERE attempt_id = ? AND organization_id = ?`), or a row fetched by a global id is verified against the scope before use. A scoped use case calling an unscoped method is a review failure — an authorization check that passes while the fetch ignores the scope is a cross-tenant leak.

## Example

```ts
// src/services/invitation.service.ts
const makeInvitationService = Effect.gen(function* () {
  const { runTransaction } = yield* Database;
  const organizationRepo = yield* OrganizationRepo;
  const invitationRepo = yield* InvitationRepo;
  const memberStats = yield* OrganizationMemberStatsQuery;
  const mailer = yield* Mailer;

  return {
    inviteUser: Effect.fn("InvitationService.inviteUser")(
      function* (input: InviteUserInput) {
        const actor = yield* CurrentActor;

        const invitation = yield* runTransaction((tx) =>
          Effect.gen(function* () {
            const organization = yield* organizationRepo.findOrganizationById(input.organizationId, tx);
            if (!organization) return yield* new OrganizationNotFound({ organizationId: input.organizationId });

            const members = yield* memberStats.countOrganizationMembersByOrganizationId(organization.id, tx);
            if (members >= organization.seats) return yield* new SeatLimitReached({ seats: organization.seats });

            return yield* invitationRepo.create(
              { data: { email: input.email, organizationId: organization.id, invitedByUserId: actor.userId } },
              tx,
            );
          }),
        );

        yield* mailer.sendInvitation(invitation);
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

The inferred type of `inviteUser` is the use case's contract: `(input: InviteUserInput) => Effect<Invitation, OrganizationNotFound | SeatLimitReached, CurrentActor>`. It names every failure a host must handle and the context it must provide.

## Paged lists

A paged list use case runs the list method and its count (see `data-access.md`, Pagination) and returns `{ items, total }`.
