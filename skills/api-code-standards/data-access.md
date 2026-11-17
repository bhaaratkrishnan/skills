# Data access: shapes, repos, queries

Three layers, split by **what a function returns**, never by how many tables it touches:

- **Row**: one table's full record. Repos return rows.
- **Shape**: any other result: a projection, a join, an aggregate, a nested result. Queries return shapes.
- **Selection**: the Drizzle column object that produces a shape. Shapes are inferred from selections.

## Where does this code go?

Run this for every new data-access function:

1. **Writes** (insert, update, delete) go in the repo of the table being written. A write spanning several tables is a service that calls each repo inside one transaction.
2. **Reads returning a plain row** of one table go in that table's repo, however complex the filter.
3. **Reads returning anything else** go in a query. Find the **root entity** by asking *"one row per what?"* (or *"the what?"* for a single result). The answer names the file: `<root-entity>.query.ts`. The filter, the joins and any nested data never decide placement.
   - `getUsersByOrg` → one row per user → `user.query.ts`
   - An org with its members nested → one row per org → `organization.query.ts`
   - Members joined with their user details → one row per membership → `organization-member.query.ts`
4. **Results with no root entity** (scalars, totals, summaries) go in `<feature>.stats.query.ts`. This is the only feature-named query file; anything with a root entity takes rule 3.

Done when the function sits in the file this procedure names, and the shape it returns lives in that root entity's shape file.

## Shapes — `src/shapes/<entity>.shape.ts`

1. A shape file holds the entity's row type and every named shape and selection for it. It imports only the database schema and other shape files, so shapes can never form an import cycle with queries.
2. The bare entity name is the row: `User = typeof usersTable.$inferSelect`. Every other shape carries a qualified name: `UserListItem`, `UserSummary`, `UserWithMembership`.
3. Shape types are inferred from their selection with `InferShape`, so the schema is the single source of truth. Joined fields come from the joined table's columns.
4. A shape lives in its root entity's file; other files import it. When a caller needs a different form, add a new named shape to the owner's file.
5. `InferShape` is defined once, in `src/shapes/infer.ts`, wrapping Drizzle's `SelectResultFields`. That import comes from an internal Drizzle path, so when Drizzle moves it, `infer.ts` is the one edit.

```ts
// src/shapes/infer.ts
import type { SelectResultFields } from "drizzle-orm/query-builders/select.types";
export type InferShape<S> = SelectResultFields<S>;
```

```ts
// src/shapes/user.shape.ts
import { usersTable, organizationMembersTable } from "<schema package>";
import type { InferShape } from "./infer";

export type User = typeof usersTable.$inferSelect;

export const userListItemSelection = {
  id: usersTable.id,
  name: usersTable.name,
  email: usersTable.email,
};
export type UserListItem = InferShape<typeof userListItemSelection>;

export const userWithMembershipSelection = {
  ...userListItemSelection,
  role: organizationMembersTable.role,
};
export type UserWithMembership = InferShape<typeof userWithMembershipSelection>;
```

## Repos — `src/repos/<table>.repo.ts`

1. A repo owns exactly one table. Its methods return that table's row type, imported from the table's shape file, or nothing.
2. Every write to the table goes through its repo.
3. Every method takes an optional trailing `tx` parameter, reads included, so a transaction can read its own uncommitted writes. `tx` is always the last positional parameter, never a field of an object argument.

## Queries — `src/queries/<root-entity>.query.ts`

1. Queries are read-only.
2. Each method runs **one SQL statement** and returns a shape from the root entity's shape file.
3. Reuse across queries happens at the selection level: import selections from shape files and compose them into your own statement. Combining the results of separate fetches belongs in a service.
4. Every method takes an optional trailing `tx` parameter, same as repos, and builds its whole statement (CTEs and subqueries included) from the resolved client `tx ?? db`, so every part runs on the same connection.

```ts
// src/queries/user.query.ts
getUsersByOrg: ({ organizationId }: { organizationId: string }, tx?: DbTxType) =>
  runQuery((db) =>
    (tx ?? db)
      .select(userWithMembershipSelection)
      .from(usersTable)
      .innerJoin(organizationMembersTable, eq(organizationMembersTable.userId, usersTable.id))
      .where(eq(organizationMembersTable.organizationId, organizationId)),
  ), // Effect<UserWithMembership[], DatabaseError>
```

## Method naming

Methods are named `<verb><Result>By<Filter>`, and the verb states the empty case:

- `find*` may come back empty: `null` for one, `[]` for many.
- `get*` fails with a typed not-found error when nothing matches.
