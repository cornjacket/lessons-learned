# Function-Field Mocks in Go

## The Pattern

```go
type mockOutboxRepository struct {
    InsertFn func(ctx context.Context, event *events.Envelope) error
}

func (m *mockOutboxRepository) Insert(ctx context.Context, event *events.Envelope) error {
    return m.InsertFn(ctx, event)
}
```

## Why the Indirection Matters

The struct method (`Insert`) satisfies the interface. The function field (`InsertFn`) lets each test define the mock's behavior inline:

```go
func TestIngest_Success(t *testing.T) {
    mock := &mockOutboxRepository{
        InsertFn: func(ctx context.Context, event *events.Envelope) error {
            return nil // this test: outbox succeeds
        },
    }
    service := NewService(mock, slog.Default())
    // ... test the success path
}

func TestIngest_OutboxError(t *testing.T) {
    mock := &mockOutboxRepository{
        InsertFn: func(ctx context.Context, event *events.Envelope) error {
            return fmt.Errorf("connection refused") // this test: outbox fails
        },
    }
    service := NewService(mock, slog.Default())
    // ... test the error path
}
```

## What You'd Lose Without the Indirection

**Hardcoded return values inside the method** — the mock's behavior is hidden from the test reader. You'd have to navigate to the mock definition to understand what scenario the test is exercising.

**Shared mutable state** — you could add setter methods or exported fields that change the mock's behavior, but this creates coupling between tests and risks race conditions if the mock is used concurrently (e.g., in parallel subtests).

**Test init functions with side effects** — clumsy and fragile. The relationship between "what the test sets up" and "what the mock does" becomes indirect and hard to trace.

## The Key Insight

The function field makes mock behavior **local to the test that defines it**. The test reader sees the setup and the assertion in one place. No navigating to mock definitions, no shared state, no side effects. The indirection buys transparency.
