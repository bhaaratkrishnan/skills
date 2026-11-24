# BFF: screen-shaped reads per audience

A domain route returns records; a screen needs composed data in a shape that differs per audience — an admin dashboard's totals, an org portal's member list with stats. These reads are use cases too: they live in the package, never composed in a host handler.

## Rules

1. **A screen read lives in `src/services/bff/<audience>.service.ts` — one service per audience, one method per screen (`getDashboard`, `getStudentOverview`).** The file is named for the audience, the method for the screen; screens never appear in domain services, and screen names never name a query file.
2. **Every rule in `services.md` applies unchanged, and so do the tests in `package.md`:** the actor from context, typed domain errors, `Effect.fn` spans, mocked-repo tests, and one test per error tag — so every screen has a test where the wrong audience gets `Forbidden`.
3. **A screen service is read-only: it depends on read methods only — queries and repo `find*` methods.** No writes, no transactions, no side effects.
4. **Authorization is the audience assertion, checked in the screen service:** the actor's `globalRole` for platform staff, a fresh membership read for org-scoped audiences. The screen service is the authority (`services.md` #5); a host guard is fast-fail only.
5. **A screen method always has an output schema, written here** (`schemas.md` #4 names screens as its always-schema case). A screen's shape is a contract the host serializes and documents from, so a screen change is a schema change, not a host edit.
6. **The host route stays thin and is guarded once per audience: decode → provide context → call one method → map errors.** It holds no composition, no authorization, and imports no query or repo.
7. **A write never lives in a screen service; it is a domain use case, wherever its route path sits.**
   - A route that "does several jobs" is one use case in the domain service that owns its root record (`services.md` #10), run in one transaction and called from a thin route.
   - If a write seems audience-specific, the difference is either authorization — handled inside the use case, from the actor — or an extra step, which is a separate use case. It is never an audience folder.
