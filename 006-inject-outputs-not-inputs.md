# Lesson 006 — Inject Outputs, Not Inputs

**Category:** architecture
**Source project:** cornjacket-platform
**Date:** 2026-02-10

## Context

While designing `Service()` entry points for a CQRS pipeline (Ingestion → EventHandler → Query), the question arose: what dependencies should each `Service()` function accept as interfaces vs. create internally?

The initial design gave every service a `*pgxpool.Pool` and had each service create its own repos and adapters internally. This worked for Ingestion and Query, but EventHandler's only use of the pool was to create a `PostgresStore` for writing projections — the exact output that feeds the Query service downstream. Injecting the pool meant EventHandler was creating its own output internally, making it impossible to mock for component testing.

## Lesson

**In a service pipeline, inject output dependencies as interfaces. Leave input infrastructure internal.**

- **Outputs** cross service boundaries — they feed downstream consumers. Injecting them as interfaces lets you mock the downstream side in component tests to verify what was produced.
- **Inputs** are infrastructure the service configures and consumes from (HTTP listeners, message bus consumers). These are tested by sending real traffic, not by mocking the input source.

### Applied to a CQRS pipeline

| Service | Input (internal) | Output (injected interface) |
|---------|------------------|-----------------------------|
| Ingestion | HTTP requests, DB (outbox, event store) | `EventSubmitter` — publishes to message bus |
| EventHandler | Message bus consumer | `ProjectionWriter` — writes projections |
| Query | HTTP requests, DB (projections) | HTTP responses to external clients (no interface needed) |

The DB pool is concrete where the service owns its internal DB wiring (Ingestion creating outbox/event store repos, Query creating projection reader). But when the DB interaction *is* the output (EventHandler writing projections), the pool disappears — replaced by the output interface.

### The test consequence

This pattern enables component testing — start a real service with real input infrastructure but mock its output:

```
Component test for Ingestion:
    Real DB + real HTTP server + mock EventSubmitter
    → POST events, assert EventSubmitter received them

Component test for EventHandler:
    Real Redpanda consumer + mock ProjectionWriter
    → Produce events, assert ProjectionWriter received projection writes
```

Without output injection, you can only test a service by inspecting the downstream database — which couples your test to both services.

## Decision Guidance

When designing service entry points in a pipeline:

1. **Identify the service's output** — what does it produce that another service consumes? That's the interface parameter.
2. **Leave inputs internal** — HTTP servers, message consumers, and DB repos that the service reads from are configured via config structs, not injected.
3. **DB pools are concrete when internal, absent when the DB interaction is the output.** If the only reason to pass a pool is to create the output adapter, inject the adapter interface instead.
4. **The last service in the pipeline has no output interface** — its output is external (HTTP responses, files, etc.) and doesn't need mocking.

## Related Lessons

- ai-builder-lessons [005 — Abstract Test Infrastructure for Local vs CI]: The `testutil` layer that provides real infrastructure for component test inputs.
- [004 — Integration Tests Should Own Their Schema Lifecycle](004-integration-tests-own-schema-lifecycle.md): Schema management for tests that use real databases.
