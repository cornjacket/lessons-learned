# Interaction Testing vs State Testing

## The Two Approaches

Every test ultimately asks one of two questions:

| Approach | Question | What You Assert |
|----------|----------|-----------------|
| **State testing** | Did we get the right result? | Return values, final state |
| **Interaction testing** | Did we talk to the right collaborators? | Which dependencies were called, with what arguments |

These aren't competing ideologies — they're tools for different situations.

## State Testing

```go
func TestAdd(t *testing.T) {
    result := calculator.Add(2, 3)
    assert.Equal(t, 5, result)
}
```

You call the function. You check the output. You don't care *how* it computed the answer — maybe it used a lookup table, maybe it looped, doesn't matter. The result is what matters.

**Use when:** The function computes something. It takes inputs and produces a meaningful output or transforms state you can inspect.

## Interaction Testing

```go
func TestProcessEntry_Success(t *testing.T) {
    var inserted, submitted, deleted bool

    eventStore := &mockEventStoreWriter{
        InsertFn: func(ctx context.Context, event *events.Envelope) error {
            inserted = true
            return nil
        },
    }
    submitter := &mockEventSubmitter{
        SubmitEventFn: func(ctx context.Context, event *events.Envelope) error {
            submitted = true
            return nil
        },
    }
    outbox := &mockOutboxReader{
        DeleteFn: func(ctx context.Context, outboxID string) error {
            deleted = true
            return nil
        },
    }

    p := &Processor{eventStore: eventStore, submitter: submitter, outbox: outbox}
    p.processEntry(context.Background(), slog.Default(), entry)

    assert.True(t, inserted, "should insert into event store")
    assert.True(t, submitted, "should submit to message bus")
    assert.True(t, deleted, "should delete from outbox")
}
```

You call the function. You check *who it talked to*. The booleans are call recorders — did it insert? did it submit? did it delete? The function itself returns nothing meaningful.

**Use when:** The function is orchestration — its job is to call the right things in the right order under the right conditions.

## Why `processEntry` Demands Interaction Testing

`processEntry` is pure glue logic. Its entire purpose is coordinating three side effects in sequence:

```
Insert into event store → Submit to message bus → Delete from outbox
```

There's no return value to assert. There's no state transformation to inspect. If you tried state testing, you'd be testing the mocks themselves — "did the mock store the value I told it to store?" That proves nothing.

The interesting behavior is in the *branching*:
- **Success path:** all three called, outbox entry deleted
- **Duplicate event (23505):** skip insert, still submit and delete
- **Submit error:** don't delete, increment retry
- **Max retries exceeded:** don't process at all

Each scenario is a different interaction pattern. The test verifies the right collaborators were called (and the wrong ones weren't).

## Capturing Arguments, Not Just Calls

Interaction testing gets more useful when you verify *what* was passed, not just *whether* something was called:

```go
submitter := &mockEventSubmitter{
    SubmitEventFn: func(ctx context.Context, event *events.Envelope) error {
        assert.Equal(t, "sensor.reading", event.EventType)  // verify argument
        assert.Equal(t, "device-001", event.AggregateID)
        return nil
    },
}
```

Now you're testing that the orchestrator passes the right data to the right collaborator — not just that it calls it.

## Negative Assertions Are Equally Important

The `t.Fatal` pattern in interaction tests verifies that something was *not* called:

```go
outbox := &mockOutboxReader{
    DeleteFn: func(ctx context.Context, outboxID string) error {
        t.Fatal("Delete should not be called when submit fails")
        return nil
    },
}
```

If the orchestrator incorrectly deletes the outbox entry after a submit failure, the test immediately fails with a clear message. This is the interaction testing equivalent of "assert not equal."

## When Each Approach Breaks Down

**State testing breaks down** when the function is pure orchestration with no meaningful return value. You end up testing mock internals.

**Interaction testing breaks down** when the function computes something internally. Testing that `Add()` calls `incrementByOne()` five times is testing implementation details — refactoring the algorithm would break the test even if the result is still correct.

## The Heuristic

Look at the function under test:
- **Does it compute and return something?** → State test
- **Does it coordinate calls to dependencies?** → Interaction test
- **Does it do both?** → State test the return value, interaction test the side effects

Most service-layer code in CQRS systems (handlers, processors, orchestrators) is orchestration — which is why interaction testing dominates in this codebase.
