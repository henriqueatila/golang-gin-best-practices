# Data Patterns — Saga Pattern (Orchestration)

Distributed transaction pattern for business transactions spanning multiple services.

---

## Gate — all must be true

- [ ] A business transaction spans multiple services or databases
- [ ] Each step can fail independently and may have already partially committed
- [ ] You need compensating actions (rollbacks that can't use a single DB transaction)
- [ ] A single `BEGIN / COMMIT` across one database is NOT sufficient

**Simpler alternative first:** If all data is in one PostgreSQL database, use a database transaction. If two services are involved, call them sequentially with explicit compensation:

```go
func (s *CheckoutService) Checkout(ctx context.Context, orderID string) error {
    if err := s.inventoryClient.Reserve(ctx, orderID); err != nil {
        return fmt.Errorf("reserve inventory: %w", err)
    }
    if err := s.paymentClient.Charge(ctx, orderID); err != nil {
        _ = s.inventoryClient.Release(ctx, orderID) // compensate
        return fmt.Errorf("charge payment: %w", err)
    }
    return nil
}
```

---

## Orchestrator Saga

Use when you have 3+ steps or need durable state across retries.

```go
type SagaStep struct {
    Name       string
    Execute    func(ctx context.Context, sagaID string) error
    Compensate func(ctx context.Context, sagaID string) error
}

type SagaOrchestrator struct {
    steps []SagaStep
    store SagaStateStore
}

type SagaStateStore interface {
    SaveProgress(ctx context.Context, sagaID string, completedStep int) error
    LoadProgress(ctx context.Context, sagaID string) (int, error)
    MarkFailed(ctx context.Context, sagaID string, reason string) error
    MarkCompleted(ctx context.Context, sagaID string) error
}

func (o *SagaOrchestrator) Run(ctx context.Context, sagaID string) error {
    lastCompleted, _ := o.store.LoadProgress(ctx, sagaID)
    completed := make([]int, 0, len(o.steps))

    for i, step := range o.steps {
        if i < lastCompleted {
            completed = append(completed, i)
            continue
        }
        slog.InfoContext(ctx, "saga step executing", "saga_id", sagaID, "step", step.Name)
        if err := step.Execute(ctx, sagaID); err != nil {
            slog.ErrorContext(ctx, "saga step failed", "saga_id", sagaID, "step", step.Name, "err", err)
            _ = o.store.MarkFailed(ctx, sagaID, err.Error())
            for j := len(completed) - 1; j >= 0; j-- {
                si := completed[j]
                if o.steps[si].Compensate != nil {
                    if cerr := o.steps[si].Compensate(ctx, sagaID); cerr != nil {
                        slog.ErrorContext(ctx, "compensation failed", "step", o.steps[si].Name, "err", cerr)
                    }
                }
            }
            return fmt.Errorf("saga %s failed at step %s: %w", sagaID, step.Name, err)
        }
        completed = append(completed, i)
        _ = o.store.SaveProgress(ctx, sagaID, i+1)
    }
    return o.store.MarkCompleted(ctx, sagaID)
}
```

**PostgreSQL state store schema:**

```sql
CREATE TABLE saga_state (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    saga_type       TEXT        NOT NULL,
    status          TEXT        NOT NULL DEFAULT 'running',
    completed_steps INT         NOT NULL DEFAULT 0,
    failure_reason  TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

**Complexity: HIGH (4/5)** — use sequential calls with manual compensation (2 services) or a DB transaction (single DB) instead.

---

## See Also

- `data-patterns-outbox.md` — Transactional Outbox pattern (atomic DB + message publish)
