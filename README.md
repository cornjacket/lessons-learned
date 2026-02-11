# Lessons Learned

Personal technical insights — patterns, idioms, and "now I get it" moments worth preserving.

Not project-specific. Not process-specific. Just things I learned that I don't want to re-derive.

## Index

- [001-go-function-field-mocks.md](001-go-function-field-mocks.md) — Why the indirection in hand-written Go mocks buys transparency
- [002-interaction-testing-vs-state-testing.md](002-interaction-testing-vs-state-testing.md) — State tests check results; interaction tests check collaborator calls
- [003-percent-w-only-in-fmt-errorf.md](003-percent-w-only-in-fmt-errorf.md) — `%w` is exclusive to `fmt.Errorf`; use `%v` everywhere else
- [004-integration-tests-own-schema-lifecycle.md](004-integration-tests-own-schema-lifecycle.md) — Drop all tables, migrate from scratch, truncate at start of each test
- [005-alter-migrations-are-for-production.md](005-alter-migrations-are-for-production.md) — Rewrite CREATE migrations pre-production; ALTER only when live data exists
- [006-inject-outputs-not-inputs.md](006-inject-outputs-not-inputs.md) — Inject output dependencies as interfaces; leave input infrastructure internal
- [007-sentinel-event-pattern-for-negative-assertions.md](007-sentinel-event-pattern-for-negative-assertions.md) — Use ordering guarantees as synchronization instead of timeouts for negative assertions
