# Lesson 007 — Sentinel Event Pattern for Negative Assertions

**Category:** testing
**Source project:** cornjacket-platform
**Date:** 2026-02-10

## Context

While writing component tests for an event-driven CQRS system, one test needed to prove that an unknown event type was consumed but produced **no** output. The service reads from Redpanda, dispatches events to handlers by prefix, and writes projections. An event with no matching handler should be silently skipped.

The naive approach — "produce the event, wait 500ms, check that nothing happened" — is brittle in both directions:
- **Too short:** The consumer might not have polled yet, so the test passes falsely.
- **Too long:** Slows down the test suite for no benefit.

Timeouts assert on the absence of activity, which is fundamentally unreliable. You can never be sure the system has finished processing — only that it hasn't produced output *yet*.

## Lesson

**Use the system's own ordering guarantees as synchronization instead of arbitrary timeouts.**

Produce the unknown event, then immediately produce a known event (the "sentinel") that *will* trigger output. Since ordered message systems guarantee in-order delivery within a partition, when the mock receives the sentinel's output, you know the unknown event was already processed and skipped.

```
1. Produce "billing.charge"    → no handler registered
2. Produce "sensor.reading"    → handler writes "sensor_state" projection

Ordering guarantee: (1) is processed before (2).
When mock receives "sensor_state", "billing.charge" was already consumed and dropped.
```

Then assert:
- The mock received exactly 1 call (the sentinel's output)
- The channel is empty afterward (no extra writes from the unknown event)

### Why this is better than timeouts

| Approach | Speed | Reliability | Proves processing happened |
|----------|-------|-------------|---------------------------|
| Timeout (500ms) | Slow | Flaky — races in both directions | No |
| Sentinel event | Instant (wakes on channel) | Deterministic — ordering guarantee | Yes |

The sentinel approach is both faster and more reliable because it turns a negative assertion ("nothing happened") into a positive one ("exactly this one thing happened, and nothing else").

## Decision Guidance

Use the sentinel event pattern when:

1. **You need to prove something was consumed but produced no output** — the classic negative assertion in async systems.
2. **The system provides ordering guarantees** — Kafka partitions, database sequences, FIFO queues, single-threaded workers. The sentinel and the event under test must share the same ordering domain.
3. **The sentinel's output is observable** — a channel-based mock, a database write, an HTTP callback. You need something to block on.

Do NOT use this pattern when:
- Events may be processed out of order (e.g., different partitions, parallel workers with no ordering). Use a different synchronization mechanism.
- The "no output" assertion is the entire test. If there's no sentinel to produce, a short timeout with a clear comment is acceptable as a last resort.

## Related Lessons

- [006 — Inject Outputs, Not Inputs](006-inject-outputs-not-inputs.md): The output injection pattern that makes channel-based mocks possible in component tests.
- [004 — Integration Tests Should Own Their Schema Lifecycle](004-integration-tests-own-schema-lifecycle.md): Infrastructure setup patterns for tests that use real message buses.
