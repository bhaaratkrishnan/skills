---
name: api-code-standards
description: Code standards for TypeScript API packages built on Effect and Drizzle. Use when writing, reviewing or refactoring API code, deciding where a database read or write belongs (repo or query), or running statements and transactions.
---

# API code standards

Each topic below is a standalone reference. Read the topic for every layer the change touches, then apply **every** rule in it. Done means each new or changed function has been checked against the topic's rules, not just the ones that looked relevant.

When reviewing, cite violations by topic and rule number (`data-access #8`) so the fix is traceable to the standard.

| Topic | Read when the change touches |
|---|---|
| [`data-access.md`](data-access.md) | Database reads or writes: repos, queries, method naming, pagination, or where a new one belongs |
| [`database.md`](database.md) | Running statements (`runQuery`, `runTransaction`), transactions, `DatabaseError` handling, or layer wiring |

Layer directories below are the defaults (`src/repos`, `src/queries`). A project that names them differently keeps its names; the rules stay the same.
