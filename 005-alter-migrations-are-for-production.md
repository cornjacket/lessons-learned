# Lesson 005 — ALTER Migrations Are for Production, Not Pre-Production

**Category:** architecture
**Source project:** cornjacket-platform
**Date:** 2026-02-10

## Context

During Phase 1 development, a migration was created to rename a column (`ALTER TABLE RENAME COLUMN timestamp TO event_time`) and add a new column (`ALTER TABLE ADD COLUMN ingested_at`). This was done as migration 003, following migrations 001 and 002 which created the tables.

The ALTER migration caused problems in integration tests because it isn't idempotent — running it twice fails with "column does not exist." The initial fix attempted to suppress the error. A better fix was identified: since there is no production deployment and no data to preserve, the ALTER migration shouldn't exist at all. The column should have been named `event_time` in the original CREATE TABLE.

## Lesson

**ALTER migrations exist to transform live data in-place. If there is no live data, rewrite the CREATE migration instead.**

ALTER makes sense when:
- A production database has rows that must be preserved
- Other systems depend on the current schema and need a coordinated migration
- Rollback must be possible without data loss

ALTER does NOT make sense when:
- The project is in local development with no deployment
- The database is routinely dropped and recreated
- There are no external consumers of the schema

In pre-production, the migration sequence should always represent the **final desired schema**, not the history of how you arrived at it. Schema history only matters when there's live data bound to each historical version.

### When to switch from "rewrite" to "ALTER"

The moment a production deployment exists with real data. From that point forward, every schema change must be an additive migration (ALTER, not rewrite). The migration sequence becomes an append-only log. Before that point, it should be freely rewritable.

## Decision Guidance

When changing a database schema during development:

1. **Pre-production (no deployment):** Edit the original CREATE migration to reflect the final schema. Delete any ALTER migrations that only exist to evolve the dev schema. Keep the migration sequence clean.
2. **Post-production (live data exists):** Always add a new ALTER migration. Never modify existing migrations. The migration sequence is now an immutable history.
3. **The boundary is the first production deployment**, not "when it feels production-ready." Before that tag, migrations are mutable. After it, they're append-only.

## Related Lessons

- [004 — Integration Tests Should Own Their Schema Lifecycle](004-integration-tests-own-schema-lifecycle.md): The testing issue that surfaced this migration problem.
