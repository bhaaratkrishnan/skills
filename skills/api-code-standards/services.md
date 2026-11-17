# Services: use cases

A service is the only layer a host calls. Each method is one **use case**, named for what the caller wants done (`inviteUser`, `listOrganizations`, `expireStaleInvitations`), and it owns everything that use case needs: authorization, business rules, transactions, combining several reads, side effects and the domain errors it can fail with.

## Rules

1. One service per domain, in `src/services/<domain>.service.ts`. Its methods are use cases, not table operations.
2. A service reaches data through repos and queries only; it holds no Drizzle builder. It may call another service, and the calls form no cycle.
3. A use case that writes more than once runs inside one `runTransaction`, and passes `tx` to every repo and query call in it, reads included.
4. Side effects outside the database (mail, storage, queues) run after the transaction commits, so a rollback never leaves a sent email behind.
5. Authorization runs in the service from `CurrentActor` (see `schemas.md`), so every host enforces the same rules.
6. `find*` results that the use case requires are turned into domain errors here: a `null` becomes `OrganizationNotFound`, never a `null` returned to the host.
7. Each method is written with `Effect.fn("<Service>.<method>")`, which names its tracing span, and ends with `Effect.catchTag("DatabaseError", Effect.die)` so its error channel holds domain errors only (see `errors.md`).
8. Inputs are already-decoded schema types; outputs are rows, query results or schema types. The service validates business rules, not shapes.
9. The shape is inferred from `make`, as for repos and queries; write no separate interface.

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
            const organization = yield* organizationRepo.findOrganizationById(actor.organizationId, tx);
            if (!organization) return yield* new OrganizationNotFound({ organizationId: actor.organizationId });

            const members = yield* memberStats.countOrganizationMembersByOrganizationId(organization.id, tx);
            if (members >= organization.seats) return yield* new SeatLimitReached({ seats: organization.seats });

            return yield* invitationRepo.create({ data: { email: input.email, organizationId: organization.id } }, tx);
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
