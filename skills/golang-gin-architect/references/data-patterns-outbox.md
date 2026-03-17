# Data Patterns — Transactional Outbox

Atomic DB + message delivery pattern. Companion to `data-patterns-saga.md`.

---

## Gate — all must be true

- [ ] You need to atomically update a database record AND publish a message/event
- [ ] "At-least-once" delivery is a requirement (losing messages is not acceptable)
- [ ] You cannot use a distributed transaction (2PC) or it is too expensive

---

## Schema

```sql
CREATE TABLE outbox (
    id           UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    topic        TEXT        NOT NULL,
    payload      JSONB       NOT NULL,
    status       TEXT        NOT NULL DEFAULT 'pending',
    created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    processed_at TIMESTAMPTZ
);
CREATE INDEX ON outbox(status, created_at) WHERE status = 'pending';
```

---

## Writer — Atomic DB Update + Outbox Insert

```go
func (r *Repository) CreateWithOutbox(ctx context.Context, cmd CreateOrderCommand) (string, error) {
    tx, err := r.db.BeginTxx(ctx, nil)
    if err != nil {
        return "", fmt.Errorf("begin tx: %w", err)
    }
    defer tx.Rollback()

    var id string
    err = tx.QueryRowContext(ctx,
        `INSERT INTO orders (user_id, status) VALUES ($1, 'pending') RETURNING id`,
        cmd.UserID,
    ).Scan(&id)
    if err != nil {
        return "", fmt.Errorf("insert order: %w", err)
    }

    payload, _ := json.Marshal(map[string]string{"order_id": id, "user_id": cmd.UserID})
    _, err = tx.ExecContext(ctx,
        `INSERT INTO outbox (topic, payload) VALUES ('order.created', $1)`, payload)
    if err != nil {
        return "", fmt.Errorf("insert outbox: %w", err)
    }
    return id, tx.Commit()
}
```

---

## Publisher — Poll and Dispatch

```go
type MessageBroker interface {
    Publish(ctx context.Context, topic string, payload json.RawMessage) error
}

type OutboxPublisher struct {
    db     *sqlx.DB
    broker MessageBroker
}

func (p *OutboxPublisher) Run(ctx context.Context, interval time.Duration) {
    ticker := time.NewTicker(interval)
    defer ticker.Stop()
    for {
        select {
        case <-ticker.C:
            if err := p.processBatch(ctx); err != nil {
                slog.ErrorContext(ctx, "outbox publish failed", "err", err)
            }
        case <-ctx.Done():
            return
        }
    }
}

type outboxRow struct {
    ID      string          `db:"id"`
    Topic   string          `db:"topic"`
    Payload json.RawMessage `db:"payload"`
}

func (p *OutboxPublisher) processBatch(ctx context.Context) error {
    var rows []outboxRow
    err := p.db.SelectContext(ctx, &rows,
        `SELECT id, topic, payload FROM outbox WHERE status = 'pending'
         ORDER BY created_at LIMIT 100 FOR UPDATE SKIP LOCKED`)
    if err != nil {
        return fmt.Errorf("select outbox: %w", err)
    }
    for _, row := range rows {
        if err := p.broker.Publish(ctx, row.Topic, row.Payload); err != nil {
            _, _ = p.db.ExecContext(ctx,
                `UPDATE outbox SET status = 'failed' WHERE id = $1`, row.ID)
            continue
        }
        _, _ = p.db.ExecContext(ctx,
            `UPDATE outbox SET status = 'sent', processed_at = NOW() WHERE id = $1`, row.ID)
    }
    return nil
}
```

**Idempotent consumers:** Each consumer must deduplicate by `outbox.id`. Store processed IDs in a `processed_messages(id UUID PRIMARY KEY, processed_at TIMESTAMPTZ)` table.

---

## Pattern Comparison

| Pattern | Complexity | When to Use | Simple Alternative |
|---|---|---|---|
| Saga (Orchestrator) | HIGH (4/5) | 3+ cross-service steps + compensations | Sequential calls with manual compensation |
| Transactional Outbox | MEDIUM (3/5) | Atomic DB + message publish required | Write DB then publish; retry on failure |
