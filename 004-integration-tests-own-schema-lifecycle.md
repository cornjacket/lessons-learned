# Lesson 004 — Integration Tests Should Own Their Schema Lifecycle

**Category:** architecture
**Source project:** cornjacket-platform
**Date:** 2026-02-10

## Context

When implementing integration tests against a real Postgres database, the initial approach had `TestMain` re-run the project's migration files to ensure the schema existed. This seemed reasonable — if the schema is missing, the tests should create it.

In practice, the migrations had already been applied (via `make migrate` during development). Some migrations were idempotent (`CREATE TABLE IF NOT EXISTS`), but one was not (`ALTER TABLE RENAME COLUMN`). The failing migration was swallowed with a log message: "skipping — likely already applied."

This created a false safety net. `TestMain` appeared to manage the schema but was actually a no-op with error suppression. If someone ran the tests against a truly empty database, half the migrations would apply and the rename would succeed — but only by accident of ordering, not by design.

## Lesson

**Integration tests should own the full database schema lifecycle: drop everything, then migrate from scratch.**

The pattern:

```
TestMain:
    1. Drop all tables (clean slate)
    2. Run all migrations in order (fresh schema)
    3. m.Run() — execute tests

Each test:
    4. Truncate tables (clean rows, keep schema)
    5. Run test logic
```

This separates two concerns:
- **Schema lifecycle** (TestMain) — happens once per test run, ensures migrations work end-to-end
- **Data isolation** (each test) — truncate rows between tests so they don't interfere

### Cleanup-at-start, not cleanup-at-end

Each test truncates its tables **as its first action**, not in a `defer` or `t.Cleanup()` at the end. This matters because:

- If Test A panics or calls `t.FailNow()`, deferred cleanup may not execute (depending on how the failure propagates). Test B would inherit dirty rows and fail for the wrong reason.
- Cleanup-at-start is self-healing: no matter what happened to the previous test, the current test always begins with empty tables.
- End-of-test cleanup has a second problem — the last test in the suite leaves dirty data behind. If a developer then manually queries the database to inspect state, they see stale test data that looks like real data.

The cost of cleanup-at-start is negligible (one `TRUNCATE ... CASCADE` per test) and eliminates an entire class of flaky test failures caused by ordering dependencies.

### Why not just "make migrations idempotent"?

Some DDL is naturally idempotent (`CREATE TABLE IF NOT EXISTS`). Some is not (`ALTER TABLE RENAME COLUMN`, `ALTER TABLE ADD COLUMN` without guards). You can wrap non-idempotent statements in `DO $$ BEGIN ... EXCEPTION WHEN ... END $$` blocks, but that adds complexity to production migration files purely for test convenience. Migrations are designed to run once in sequence, not to be replayed — fighting that design costs more than owning the lifecycle.

### Why not skip migrations and require `make migrate`?

This works but introduces a hidden dependency: the test suite assumes someone else set up the schema. If the schema is stale (someone pulled new migrations but didn't re-run `make migrate`), tests fail with confusing errors about missing columns rather than a clear "migration failed" message. Dropping and recreating makes the test suite self-contained with respect to schema.

## Decision Guidance

When writing integration tests against a real database:

1. **`TestMain` drops all tables, then runs migrations.** This guarantees every test run starts from a known schema state and exercises the full migration sequence.
2. **Individual tests truncate tables at the start, not the end.** Cleanup-at-start is self-healing — a prior test's failure can't leave dirty state for the next test. `defer` or `t.Cleanup()` teardown is skippable; truncation as the first line of a test is not.
3. **Don't suppress migration errors.** If a migration fails against a blank schema, that's a real bug. Fatal immediately.
4. **Accept the trade-off:** manual test data inserted via `psql` during development will be lost when integration tests run. Integration tests own the database.

## Related Lessons

- ai-builder-lessons [005 — Abstract Test Infrastructure for Local vs CI]: The `testutil` abstraction layer where the drop/migrate logic lives.
- [003 — %w Only Works in fmt.Errorf](003-percent-w-only-in-fmt-errorf.md): Discovered in the same test helper code.
