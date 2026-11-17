# Database: running statements with Effect

All database work goes through the `Database` Effect service. It owns the connection pool and releases it when its layer is torn down, so code gets its client from the service by `yield* Database` and never creates a pool or imports a `db` instance.

The service exposes two ways to run work:

- `runQuery` runs a single statement and returns `Effect<A, DatabaseError>`.
- `runTransaction` runs one or more statements atomically and returns `Effect<A, E | DatabaseError>`, where `E` is the callback's own error type.

## `runQuery`: one statement

Drizzle builders are `PromiseLike`, so the callback returns the builder directly:

```ts
const { runQuery } = yield* Database;

const [user] = yield* runQuery((db) =>
  db.select().from(usersTable).where(eq(usersTable.id, id)).limit(1)
);
```

Run a single read with `runQuery`. A transaction is for grouping statements that must succeed or fail together.

## `runTransaction`: atomic groups

The callback receives the transaction handle and returns an Effect. **Any failure rolls the whole transaction back**: a query error, a typed domain error or a defect.

```ts
const { runTransaction } = yield* Database;

return yield* runTransaction((tx) =>
  Effect.gen(function* () {
    const user = yield* userRepo.create({ data }, tx);
    yield* sessionRepo.create({ ...session, userId: user.id }, tx);
    return user;
  })
);
```

- Pass `tx` to every repo and query call inside the callback, reads included, so they see the transaction's uncommitted writes.
- To run a raw Drizzle statement inside the callback, lift it with `Effect.tryPromise` and map the failure to `DatabaseError`:

```ts
yield* runTransaction((tx) =>
  Effect.tryPromise({
    try: () => tx.insert(usersTable).values(data).returning(),
    catch: (cause) => new DatabaseError({ operation: "query", cause }),
  })
);
```

## Errors

Driver failures arrive as a typed `DatabaseError`, so database code has no `try/catch`. The `operation` field says what failed:

- `"query"`: a statement failed.
- `"transaction"`: the transaction itself failed (begin or commit).

The error type flows up through repos and queries unchanged, so a service can react to a specific failure (a unique violation it turns into a domain error). At the end of each service method it becomes a defect with `Effect.catchTag("DatabaseError", Effect.die)`, so hosts see domain errors only (see `errors.md`).

## Layers

Repo and query services each depend on `Database`. Provide them through one repo layer and one query layer, both built on the same `Database` layer, so the whole app shares a single pool. The app layer composes all of them.
