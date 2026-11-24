# Database: running statements with Effect

All database work goes through the `Database` Effect service. It owns the connection pool and releases it when its layer is torn down.

The service exposes two ways to run work:

- `runQuery` runs a single statement and returns `Effect<A, DatabaseError>`.
- `runTransaction` runs one or more statements atomically and returns `Effect<A, E | DatabaseError>`, where `E` is the callback's own error type.

## Rules

1. **Code gets its client by `yield* Database`; it never creates a pool or imports a `db` instance.**
2. **A single statement runs with `runQuery`; a transaction is only for statements that must succeed or fail together.** Drizzle builders are `PromiseLike`, so the callback returns the builder directly:

   ```ts
   const { runQuery } = yield* Database;

   const [user] = yield* runQuery((db) =>
     db.select().from(usersTable).where(eq(usersTable.id, id)).limit(1)
   );
   ```

3. **Statements that must succeed or fail together run in one `runTransaction`, and any failure inside it rolls the whole transaction back** — a query error, a typed domain error or a defect. The callback receives the transaction handle and returns an Effect:

   ```ts
   const { runTransaction } = yield* Database;

   return yield* runTransaction((tx) =>
     Effect.gen(function* () {
       const user = yield* userRepo.createUser({ data }, tx);
       yield* sessionRepo.createSession({ data: { ...session, userId: user.id } }, tx);
       return user;
     })
   );
   ```

4. **Every repo and query call inside the callback receives `tx`, reads included,** so they see the transaction's uncommitted writes.
5. **A raw Drizzle statement inside a transaction is written only in a repo or query method, lifted with `Effect.tryPromise` and mapped to `DatabaseError`.** Services hold no Drizzle builder (`services.md` #2); they call the repo method, which runs on the `tx` it is given:

   ```ts
   // inside a repo method that received `tx`
   Effect.tryPromise({
     try: () => tx.insert(usersTable).values(data).returning(),
     catch: (cause) => new DatabaseError({ operation: "query", cause }),
   })
   ```

6. **Driver failures arrive as a typed `DatabaseError`, so database code has no `try/catch`.** The `operation` field says what failed:
   - `"query"`: a statement failed.
   - `"transaction"`: the transaction itself failed (begin or commit).
7. **`DatabaseError` flows up through repos and queries unchanged, and becomes a defect at the end of each service method** with `Effect.catchTag("DatabaseError", Effect.die)`. Until then a service can react to a specific failure (a unique violation it turns into a domain error); after it, hosts see domain errors only (see `errors.md`).
8. **Repo and query services are provided through one repo layer and one query layer, both built on the same `Database` layer,** so the whole app shares a single pool. The app layer composes all of them.
