---
name: api-code-standards
description: Code standards for self-contained TypeScript API packages built on Effect and Drizzle. Use when writing, reviewing or refactoring API code, adding a use case or service, deciding where a database read or write belongs (repo or query), defining errors or input schemas, or running statements and transactions.
---

# API code standards

The API package holds every use case and stays independent of any HTTP framework; hosts call its services. Each topic below is a standalone reference. Read the topic for every layer the change touches, then apply **every** rule in it. Done means each new or changed function has been checked against the topic's rules, not just the ones that looked relevant.

When reviewing, cite violations by topic and rule number (`services #7`) so the fix is traceable to the standard.

| Topic | Read when the change touches |
|---|---|
| [`package.md`](package.md) | The package boundary, its exports, layer composition, config, external adapters, hosts or tests |
| [`services.md`](services.md) | A use case: orchestration, transactions, authorization, side effects |
| [`errors.md`](errors.md) | Domain errors, error codes, `DatabaseError` handling, or mapping errors at a host |
| [`schemas.md`](schemas.md) | Use case inputs and outputs, or the request context (`CurrentActor`) |
| [`data-access.md`](data-access.md) | Database reads or writes: repos, queries, method naming, pagination, or where a new one belongs |
| [`database.md`](database.md) | Running statements (`runQuery`, `runTransaction`), transactions, `DatabaseError`, or repo and query layer wiring |

Layer directories below are the defaults (`src/services`, `src/repos`, `src/queries`, `src/errors`, `src/schemas`, `src/context`). A project that names them differently keeps its names; the rules stay the same.
