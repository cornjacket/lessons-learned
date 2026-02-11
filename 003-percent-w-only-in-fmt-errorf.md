# Lesson 003 — `%w` Only Works in `fmt.Errorf`

**Category:** tooling
**Source project:** cornjacket-platform
**Date:** 2026-02-10

## Context

During code review of a test helper function, `t.Fatalf("failed to read dir: %w", err)` was used instead of `%v`. The code ran correctly — `%w` falls back to `%v` behavior outside `fmt.Errorf`, so the error prints fine and nothing breaks at runtime.

The issue was initially called a "bug," which was inaccurate. It's a linter finding: `go vet` and `staticcheck` flag `%w` outside `fmt.Errorf` as a warning. This means it would fail CI lint checks (Phase 3) despite being functionally correct.

## Lesson

**`%w` is a format verb exclusive to `fmt.Errorf`.** It enables error wrapping for `errors.Is`/`errors.As` unwrapping. In any other formatting context — `t.Fatalf`, `fmt.Sprintf`, `log.Printf`, `fmt.Fprintf` — `%w` has no wrapping effect and is semantically wrong.

| Context | Verb | Correct? |
|---------|------|----------|
| `fmt.Errorf("failed: %w", err)` | `%w` | Yes — wraps for `errors.Is`/`errors.As` |
| `t.Fatalf("failed: %w", err)` | `%w` | No — prints correctly but triggers lint warning |
| `t.Fatalf("failed: %v", err)` | `%v` | Yes |
| `fmt.Sprintf("failed: %w", err)` | `%w` | No — same issue |
| `log.Printf("failed: %w", err)` | `%w` | No — same issue |

This is easy to get wrong because:
- The code compiles and runs correctly
- The output looks identical (`%w` falls back to `%v` rendering)
- The mistake is only caught by static analysis tools

## Decision Guidance

When formatting errors outside `fmt.Errorf`, always use `%v`. Reserve `%w` exclusively for `fmt.Errorf` where you intend to wrap the error for programmatic unwrapping. If your CI pipeline includes `go vet` or `staticcheck` (which it should), this will be caught automatically — but it's better to get it right in the first place than to discover it as a lint failure.

## Related Lessons

- ai-builder-lessons [006 — Explicit Milestone Tags in Checklists]: Both lessons illustrate things that "work fine now" but break under CI scrutiny.
