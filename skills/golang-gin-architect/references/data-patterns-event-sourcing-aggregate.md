# Data Patterns — Event Sourcing: Aggregate Reconstruction and Snapshots

Companion to `data-patterns-event-store.md` (gate, schema, EventStore Append/Load).

---

## Aggregate Reconstruction

Reconstruct state by replaying events in version order.

```go
// OrderAggregate reconstructs state by replaying events.
type OrderAggregate struct {
    ID      string
    Status  string
    Version int
}

func (a *OrderAggregate) Apply(e Event) {
    switch e.EventType {
    case "OrderCreated":
        var p struct{ Status string }
        _ = json.Unmarshal(e.Payload, &p)
        a.ID = e.AggregateID
        a.Status = p.Status
    case "OrderShipped":
        a.Status = "shipped"
    case "OrderCancelled":
        a.Status = "cancelled"
    }
    a.Version = e.Version
}

func RebuildOrder(events []Event) OrderAggregate {
    var agg OrderAggregate
    for _, e := range events {
        agg.Apply(e)
    }
    return agg
}
```

---

## Snapshot Optimization

Add when replay of >500 events becomes slow:

```go
type Snapshot struct {
    AggregateID string          `db:"aggregate_id"`
    Version     int             `db:"version"`
    State       json.RawMessage `db:"state"`
}

func SaveSnapshot(ctx context.Context, db *sqlx.DB, s Snapshot) error {
    _, err := db.ExecContext(ctx,
        `INSERT INTO snapshots (aggregate_id, version, state)
         VALUES ($1, $2, $3)
         ON CONFLICT (aggregate_id) DO UPDATE SET version = $2, state = $3`,
        s.AggregateID, s.Version, s.State,
    )
    if err != nil {
        return fmt.Errorf("save snapshot: %w", err)
    }
    return nil
}
```

Load with snapshot: load the latest snapshot first, then replay only events after `snapshot.Version`.

---

## Cost Summary

**Complexity: VERY HIGH (5/5)**

| Trade-off | Detail |
|---|---|
| Benefit | Complete audit trail; rebuild any past state; temporal queries |
| Alternative | `audit_log` table in PostgreSQL (handles 90% of audit requirements) |
| You need it when | Legal/compliance to reproduce state at any past point in time AND complex state machines AND team has event store experience |
| You do NOT need it for | "an audit log" — that's a PostgreSQL table, not event sourcing |
