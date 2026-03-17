# Data Patterns — Event Sourcing: Gate, Schema, and Event Store

Event sourcing pattern for Go Gin APIs when full audit trail and temporal queries are required.

---

## Gate — all must be true

- [ ] Full audit trail of every state change is a business or compliance requirement (not "nice to have")
- [ ] You need temporal queries: "what was the state at time T?"
- [ ] Multiple different projections of the same data are needed now (not speculatively)
- [ ] You accept: event store ops, projection rebuilds, eventual consistency, snapshot management
- [ ] Team has experience operating event-sourced systems

**Simpler alternative first:** An `audit_log` table covers 95% of audit requirements:

```sql
CREATE TABLE audit_log (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_id   UUID        NOT NULL,
    entity_type TEXT        NOT NULL,
    action      TEXT        NOT NULL,  -- 'created', 'updated', 'deleted'
    actor_id    UUID,
    diff        JSONB,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX ON audit_log(entity_id, created_at DESC);
```

---

## Schema

```sql
CREATE TABLE events (
    id             BIGSERIAL PRIMARY KEY,
    aggregate_id   UUID        NOT NULL,
    aggregate_type TEXT        NOT NULL,
    event_type     TEXT        NOT NULL,
    payload        JSONB       NOT NULL,
    version        INT         NOT NULL,
    occurred_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (aggregate_id, version)
);

CREATE TABLE snapshots (
    aggregate_id UUID PRIMARY KEY,
    version      INT  NOT NULL,
    state        JSONB NOT NULL
);
```

---

## Event Store

```go
type Event struct {
    ID            int64           `db:"id"`
    AggregateID   string          `db:"aggregate_id"`
    AggregateType string          `db:"aggregate_type"`
    EventType     string          `db:"event_type"`
    Payload       json.RawMessage `db:"payload"`
    Version       int             `db:"version"`
    OccurredAt    time.Time       `db:"occurred_at"`
}

type EventStore struct{ db *sqlx.DB }

// Append inserts events with optimistic concurrency via expectedVersion.
func (s *EventStore) Append(ctx context.Context, aggregateID string, events []Event, expectedVersion int) error {
    tx, err := s.db.BeginTxx(ctx, nil)
    if err != nil {
        return fmt.Errorf("begin tx: %w", err)
    }
    defer tx.Rollback()

    var currentVersion int
    err = tx.QueryRowContext(ctx,
        `SELECT COALESCE(MAX(version), 0) FROM events WHERE aggregate_id = $1`,
        aggregateID,
    ).Scan(&currentVersion)
    if err != nil {
        return fmt.Errorf("read version: %w", err)
    }
    if currentVersion != expectedVersion {
        return fmt.Errorf("optimistic lock: expected version %d, got %d", expectedVersion, currentVersion)
    }

    for i, e := range events {
        _, err = tx.ExecContext(ctx,
            `INSERT INTO events (aggregate_id, aggregate_type, event_type, payload, version, occurred_at)
             VALUES ($1, $2, $3, $4, $5, NOW())`,
            aggregateID, e.AggregateType, e.EventType, e.Payload, expectedVersion+i+1,
        )
        if err != nil {
            return fmt.Errorf("insert event: %w", err)
        }
    }
    return tx.Commit()
}

// Load retrieves all events for an aggregate, ordered by version.
func (s *EventStore) Load(ctx context.Context, aggregateID string) ([]Event, error) {
    var events []Event
    err := s.db.SelectContext(ctx, &events,
        `SELECT * FROM events WHERE aggregate_id = $1 ORDER BY version ASC`,
        aggregateID,
    )
    if err != nil {
        return nil, fmt.Errorf("load events: %w", err)
    }
    return events, nil
}
```

---

## See Also

- `data-patterns-event-sourcing-aggregate.md` — aggregate reconstruction, snapshot optimization, cost summary
