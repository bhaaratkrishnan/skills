# Data access: repos and queries

Two layers, split by **what a function returns**, never by how many tables it touches:

- **Repo**: owns one table. It returns that table's **row** (the full record) or nothing, and it holds every write.
- **Query**: read-only. It returns anything else: a projection, a join, an aggregate, a nested result.

## Where does this code go?

Run this for every new data-access function:

1. **Writes** (insert, update, delete) go in the repo of the table being written. A write spanning several tables is a service that calls each repo inside one transaction.
2. **Reads returning a plain row** of one table go in that table's repo, however complex the filter.
3. **Reads returning anything else** go in a query. Find the **root entity** by asking *"one row per what?"* (or *"the what?"* for a single result). The answer names the file: `<root-entity>.query.ts`. The filter, the joins and any nested data never decide placement.
   - Users of an organization → one row per user → `user.query.ts`
   - An org with its members nested → one row per org → `organization.query.ts`
   - Members joined with their user details → one row per membership → `organization-member.query.ts`
4. **Results with no root entity** (scalars, totals, summaries) go in `<entity>.stats.query.ts`, named by **what is counted or aggregated**: a count of students goes in `organization-student.stats.query.ts`. A screen that shows several stats (a dashboard) is a service calling these methods; screen names never name a query file.

Done when the function sits in the file this procedure names.

## Repos — `src/repos/<table>.repo.ts`

1. A repo owns exactly one table and exports its row type, inferred from the schema: `export type User = typeof usersTable.$inferSelect`. Other files import the row type from the repo.
2. Every method returns that row type (`User`, `User | null`, `User[]`) or nothing, and declares its return type. Writes that hand back the record use `.returning()`.
3. Every write to the table goes through its repo.

## Queries — `src/queries/<root-entity>.query.ts`

1. Queries are read-only.
2. Each method runs **one SQL statement**. Combining the results of separate statements belongs in a service.
3. Result types are **inferred** from the statement, so the schema stays the single source of truth and left-joined columns come out nullable. Export a named result type only when a consumer must write it out, and derive it from the method rather than restating the fields:

   ```ts
   export type UserWithMembership = Effect.Success<
     ReturnType<Context.Service.Shape<typeof UserQuery>["findUsersWithMembershipByOrganizationId"]>
   >[number];
   ```

   The derivation above is the default, not a mandate — when it fights back (a deep Effect type change, a result the statement maps), a hand-written named type is allowed. The rule it serves is the one that holds: a result type is derived from the method's actual output, never restated field-by-field from the schema.

4. Write the selection inline in the statement. Extract it to a named `const` only when two or more queries share it; it then lives in the query file of the entity its columns describe, and the other files import it.

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

1. Every method takes an optional trailing `tx` parameter, reads included, so a transaction can read its own uncommitted writes. `tx` is always the last positional parameter, never a field of an object argument. The whole statement (CTEs and subqueries included) is built from the resolved client `tx ?? db`, so every part runs on the same connection.
2. Each repo and query is an Effect service whose shape is inferred from `make`. Write no separate interface for it:

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

3. Timestamps (`createdAt`, `updatedAt`) are set inside the statement, never passed in by the caller — except in a use case whose purpose is to carry history (an import, a backfill, a migration), which passes them explicitly and says so in the method name (`importUsersWithHistory`). If a table can be deactivated (a soft delete), its reads exclude inactive rows by default, and a method that includes them says so in its name (`findUsersIncludingDeactivated`).

## Method naming

Methods are named `find<Result>By<Filter>`. A missing match is data, not a failure: one result returns `null`, many return `[]`. What a missing record *means* — an `OrganizationNotFound`, a create-it, an ignore — is the use case's judgment (see `services.md` #6), never the data layer's.

`By<Filter>` names only the **required** keys. Optional filters and pagination go in the params object, so one method serves every combination:

- `findUsersByOrganizationId({ organizationId, role?, status? })`, one method in place of `findUsersByOrgAndRole`, `findUsersByOrgAndRoleAndStatus`, …
- `findOrganizations({ search?, status?, page?, limit? })` when nothing is required.

Writes follow the same discipline: `create<X>` inserts one row and hands it back via `.returning()`; `update<X>By<Filter>` and `delete<X>By<Filter>` name their required keys exactly as `find` does, with optional filters in the params object on the same terms. The payload arrives as `{ data }` — one parameter object, with the scope and filters beside it and `tx` last.

## Pagination

A paged list is two methods: the list (`LIMIT`/`OFFSET` in the statement) and its count, in `<entity>.stats.query.ts`. Both take the same filter params and build their `WHERE` from one exported filter builder, kept beside the list method, so the page and the total always match. The service runs both and returns `{ items, total }`. This pair is the default, not the only shape: a single statement carrying both (a `COUNT(*) OVER ()` window) is allowed, and a list with no total is allowed when the screen shows none — whichever shape is used, where a page and a total both exist they match by construction.

A paged list declares a **total order** — a sort column plus a unique tiebreaker — so its order is deterministic. That does not make its pages stable: under concurrent writes, offset pages shift, and a row can repeat or vanish between pages. Where page stability under concurrent writes matters, use **keyset pagination** — the cursor is the last row's sort values (`WHERE (createdAt, id) > (…)`), anchored to the data rather than the position. Its `limit` has a default and a maximum; a request above the maximum is a decode failure, not a query.

## Many ids

A service that needs records for many ids calls one `find<Results>ByIds(ids)` method (`WHERE id IN (...)`), added to the repo or query that owns the result, so the statement count stays fixed however many ids there are.
