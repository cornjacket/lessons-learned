# Lessons Learned

> ## ⚠️ Moved — this repository is archived
>
> **All eight lessons now live in my second brain**, as individual notes under
> `vault/resources/`, each tagged `golang`. Migrated 2026-09-24.
>
> **Do not add lessons here.** This repo is kept only as the historical record; the authored
> dates are the part worth preserving.
>
> The premise below — "things I learned that I don't want to re-derive" — was right, and the
> second brain is a better instrument for it: the notes are embedded and semantically searchable,
> tag-linted against a controlled vocabulary, and size-gated. A folder of numbered Markdown files
> cannot do any of that, and keeping both would leave a stale duplicate, which is worse than none.
>
> All eight were kept, including the Go-specific ones (`%w` in `fmt.Errorf`, `//go:embed` for
> migrations), even though Go is not in current use. A lesson that is cheap to keep and expensive
> to rediscover earns its place while dormant. The tag is `golang` rather than `go`, because `go`
> is a common English word and would pollute lexical search.
>
> Sibling repo `ai-builder-lessons` was archived the same day for the same reason.
>
> ---

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
- [008-go-embed-for-sql-migrations.md](008-go-embed-for-sql-migrations.md) — `//go:embed` compiles SQL migrations into the binary for distroless containers
