# Data access: repos and queries

Two layers, split by **what a function returns**, never by how many tables it touches:

- **Repo**: owns one table. It returns that table's **row** (the full record) or nothing, and it holds every write.
- **Query**: read-only. It returns anything else: a projection, a join, an aggregate, a nested result.

## Where does this code go?

Run this for every new data-access function:

1. **Writes (insert, update, delete) go in the repo of the table being written.** A write spanning several tables is a service that calls each repo inside one transaction.
2. **Reads returning a plain row of one table go in that table's repo, however complex the filter.**
3. **Reads returning anything else go in a query file named for the root entity: `<root-entity>.query.ts`.** Find the root by asking *"one row per what?"* (or *"the what?"* for a single result). The filter, the joins and any nested data never decide placement.
   - Users of an organization → one row per user → `user.query.ts`
   - An org with its members nested → one row per org → `organization.query.ts`
   - Members joined with their user details → one row per membership → `organization-member.query.ts`
4. **Results with no root entity (scalars, totals, summaries) go in `<entity>.stats.query.ts`, named by what is counted or aggregated.** A count of students goes in `organization-student.stats.query.ts`. A screen that shows several stats (a dashboard) is a service calling these methods; screen names never name a query file.

Done when the function sits in the file this procedure names.

| Function | Wrong | Right | Why |
|---|---|---|---|
| Full `User` rows of one org, filtered by a join on membership | `user.query.ts` | `user.repo.ts` | Returns the plain row (#2); the join is only a filter |
| Users with their membership `role` column | `user.repo.ts` | `user.query.ts` | Returns a projection, not the row (#3) |
| Members of an org, each with user email | `organization.query.ts` | `organization-member.query.ts` | One row per membership, not per org (#3) |
| Count of an org's members | `organization.query.ts` | `organization-member.stats.query.ts` | No root entity; named for what is counted (#4) |
| Admin dashboard totals in one method | `dashboard.stats.query.ts` | `bff/admin.service.ts` calling stats queries | Screens never name a query file (#4) |
| Insert a user and its first membership | `user.repo.ts` writing both tables | A service calling both repos in one transaction | Each write goes in its own table's repo (#1) |

## Repos — `src/repos/<table>.repo.ts`

5. **A repo owns exactly one table and exports its row type, inferred from the schema:** `export type User = typeof usersTable.$inferSelect`. Other files import the row type from the repo.
6. **Every repo method returns that row type (`User`, `User | null`, `User[]`) or nothing, and declares its return type.** Writes that hand back the record use `.returning()`.
7. **Every write to the table goes through its repo.**

## Queries — `src/queries/<root-entity>.query.ts`

8. **Queries are read-only.**
9. **Each query method runs one SQL statement.** Combining the results of separate statements belongs in a service.
10. **Query result types are inferred from the statement, never restated field-by-field from the schema.** This keeps the schema the single source of truth and makes left-joined columns come out nullable. Export a named result type only when a consumer must write it out, and derive it from the method:

    ```ts
    export type UserWithMembership = Effect.Success<
      ReturnType<Context.Service.Shape<typeof UserQuery>["findUsersWithMembershipByOrganizationId"]>
    >[number];
    ```

    The derivation above is the default, not a mandate. When it fights back (a deep Effect type change, a result the statement maps), a hand-written named type is allowed — as long as it describes the method's actual output.

11. **Write the selection inline in the statement; extract it to a named `const` only when two or more queries share it.** A shared selection lives in the query file of the entity its columns describe, and the other files import it.

```ts
// src/queries/user.query.ts
findUsersWithMembershipByOrganizationId: (
  { organizationId }: { organizationId: string },
  tx?: DbTxType,
) =>
  runQuery((db) =>
    (tx ?? db)
      .select({
        id: usersTable.id,
        email: usersTable.email,
        role: organizationMembersTable.role,
      })
      .from(usersTable)
      .innerJoin(organizationMembersTable, eq(organizationMembersTable.userId, usersTable.id))
      .where(eq(organizationMembersTable.organizationId, organizationId)),
  ),
```

## Both layers

12. **Every method takes an optional trailing `tx` parameter, reads included, and builds its whole statement from `tx ?? db`.**
    - A transaction can then read its own uncommitted writes.
    - `tx` is always the last positional parameter, never a field of an object argument.
    - CTEs and subqueries are built from the same resolved client, so every part runs on the same connection.
    - The one exception is a locking read (#18), whose `tx` is required.
13. **Each repo and query is an Effect service whose shape is inferred from `make`; write no separate interface:**

    ```ts
    const makeUserRepo = Effect.gen(function* () {
      const { runQuery } = yield* Database;
      return {
        findUserById: (id: string, tx?: DbTxType): Effect.Effect<User | null, DatabaseError> =>
          runQuery((db) => (tx ?? db).select().from(usersTable).where(eq(usersTable.id, id)).limit(1))
            .pipe(Effect.map(([user]) => user ?? null)),
      };
    });

    export class UserRepo extends Context.Service<UserRepo>()("@app/repos/UserRepo", {
      make: makeUserRepo,
    }) {
      static readonly layer = Layer.effect(UserRepo, UserRepo.make);
    }
    ```

14. **Timestamps (`createdAt`, `updatedAt`) are set inside the statement, and soft-deleted rows are excluded by default; a method that breaks either default says so in its name.**
    - A use case whose purpose is to carry history (an import, a backfill, a migration) passes timestamps explicitly: `importUsersWithHistory`.
    - A read that includes deactivated rows: `findUsersIncludingDeactivated`.

## Method naming

15. **Reads are named `find<Result>By<Filter>`, and a missing match is data, not a failure: one result returns `null`, many return `[]`.** What a missing record *means* — an `OrganizationNotFound`, a create-it, an ignore — is the use case's judgment (`services.md` #6), never the data layer's.
16. **`By<Filter>` names only the required keys; optional filters and pagination go in the params object, so one method serves every combination.**
    - `findUsersByOrganizationId({ organizationId, role?, status? })`, one method in place of `findUsersByOrgAndRole`, `findUsersByOrgAndRoleAndStatus`, …
    - `findOrganizations({ search?, status?, page?, limit? })` when nothing is required.
17. **Writes are `create<X>`, `update<X>By<Filter>` and `delete<X>By<Filter>`, with the payload as `{ data }` and `tx` last.**
    - `create<X>` inserts one row and hands it back via `.returning()`.
    - `update` and `delete` name their required keys exactly as `find` does, with optional filters in the params object on the same terms.
    - The payload arrives as `{ data }` — one parameter object, with the scope and filters beside it.
18. **A read that takes a row lock ends in `ForUpdate` (`findOrganizationByIdForUpdate`), runs `SELECT ... FOR UPDATE`, and takes a required `tx`.** A lock outside a transaction is released as soon as the statement ends, so the signature makes the misuse impossible. Use it to serialize check-then-act use cases on one parent row (`services.md` #12).

## Pagination

19. **A paged list is the list method (`LIMIT`/`OFFSET` in the statement) plus its count in `<entity>.stats.query.ts`, both building their `WHERE` from one exported filter builder.**
    - The filter builder lives beside the list method, so the page and the total always match. The service runs both and returns `{ items, total }`.
    - This pair is the default, not the only shape: a single statement carrying both (a `COUNT(*) OVER ()` window) is allowed, and so is a list with no total when the screen shows none. Whatever the shape, where a page and a total both exist they match by construction.
20. **A paged list declares a total order — a sort column plus a unique tiebreaker — and uses keyset pagination where page stability under concurrent writes matters.**
    - A total order makes the order deterministic; it does not make pages stable. Under concurrent writes, offset pages shift, and a row can repeat or vanish between pages.
    - Keyset pagination uses the last row's sort values as the cursor (`WHERE (createdAt, id) > (…)`), anchored to the data rather than the position.
21. **`limit` has a default and a maximum; a request above the maximum is a decode failure, not a query.**

## Many ids

22. **A service that needs records for many ids calls one `find<Results>ByIds(ids)` method (`WHERE id IN (...)`)**, added to the repo or query that owns the result, so the statement count stays fixed however many ids there are.
