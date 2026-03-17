# Replication and HA — Go Application Resilience

## Connection Retry with Exponential Backoff

```go
// internal/db/wait.go
package db

import (
    "context"
    "fmt"
    "log/slog"
    "time"

    "github.com/jmoiron/sqlx"
    _ "github.com/lib/pq"
)

// WaitForConnection attempts to connect to dsn with exponential backoff.
// Returns the connected *sqlx.DB or an error after maxAttempts.
func WaitForConnection(ctx context.Context, dsn string, maxAttempts int, logger *slog.Logger) (*sqlx.DB, error) {
    var db *sqlx.DB
    var err error
    delay := 500 * time.Millisecond

    for attempt := 1; attempt <= maxAttempts; attempt++ {
        db, err = sqlx.ConnectContext(ctx, "postgres", dsn)
        if err == nil {
            logger.InfoContext(ctx, "database connected", "attempt", attempt)
            return db, nil
        }
        logger.WarnContext(ctx, "database connection failed",
            "attempt", attempt, "max", maxAttempts,
            "delay", delay.String(), "err", err,
        )
        select {
        case <-ctx.Done():
            return nil, fmt.Errorf("context cancelled waiting for db: %w", ctx.Err())
        case <-time.After(delay):
        }
        delay *= 2
        if delay > 30*time.Second {
            delay = 30 * time.Second
        }
    }
    return nil, fmt.Errorf("database unavailable after %d attempts: %w", maxAttempts, err)
}
```

## Circuit Breaker for Replica Unavailability

```go
// internal/db/circuit.go
package db

import (
    "context"
    "log/slog"
    "sync/atomic"
    "time"

    "github.com/jmoiron/sqlx"
)

// ReplicaCircuit tracks replica availability. When tripped, reads fall back to primary.
type ReplicaCircuit struct {
    tripped    atomic.Bool
    primary    *sqlx.DB
    replica    *sqlx.DB
    resetAfter time.Duration
    logger     *slog.Logger
}

func NewReplicaCircuit(primary, replica *sqlx.DB, resetAfter time.Duration, logger *slog.Logger) *ReplicaCircuit {
    return &ReplicaCircuit{primary: primary, replica: replica, resetAfter: resetAfter, logger: logger}
}

// DB returns the replica if healthy, or primary if the circuit is tripped.
func (rc *ReplicaCircuit) DB(ctx context.Context) *sqlx.DB {
    if rc.tripped.Load() {
        return rc.primary
    }
    return rc.replica
}

// RecordFailure trips the circuit and schedules an automatic reset.
func (rc *ReplicaCircuit) RecordFailure(ctx context.Context) {
    if rc.tripped.CompareAndSwap(false, true) {
        rc.logger.WarnContext(ctx, "replica circuit tripped — routing reads to primary",
            "reset_after", rc.resetAfter.String(),
        )
        go func() {
            time.Sleep(rc.resetAfter)
            if err := rc.replica.PingContext(context.Background()); err == nil {
                rc.tripped.Store(false)
                rc.logger.Info("replica circuit reset — replica is healthy again")
            }
        }()
    }
}
```

Usage in a repository:

```go
func (r *OrderRepo) ListWithCircuit(ctx context.Context, userID string) ([]Order, error) {
    db := r.circuit.DB(ctx)
    var orders []Order
    if err := db.SelectContext(ctx, &orders,
        `SELECT * FROM orders WHERE user_id = $1 ORDER BY created_at DESC LIMIT 100`, userID,
    ); err != nil {
        r.circuit.RecordFailure(ctx)
        if err2 := r.conns.Primary().SelectContext(ctx, &orders,
            `SELECT * FROM orders WHERE user_id = $1 ORDER BY created_at DESC LIMIT 100`, userID,
        ); err2 != nil {
            return nil, fmt.Errorf("order list fallback: %w", err2)
        }
    }
    return orders, nil
}
```

## Graceful Handling of Failover

During a Patroni or RDS failover, existing connections receive `driver: bad connection`. The standard `database/sql` pool retries transparently, but long-lived idle connections may need eviction:

```go
// ForceReconnect evicts idle connections to force re-dial against the new primary.
func ForceReconnect(db *sqlx.DB) {
    db.SetMaxIdleConns(0)               // evict all idle connections immediately
    time.Sleep(100 * time.Millisecond)
    db.SetMaxIdleConns(5)               // restore normal idle pool
}
```

Set `ConnMaxLifetime` shorter than the failover window (1–2 minutes) so connections naturally recycle.

## Checklist

- [ ] Streaming replication configured: `wal_level = replica`, `max_wal_senders` set, `replicator` role created
- [ ] Standby verified with `pg_stat_replication` showing `streaming` state
- [ ] Replication lag monitored: alert threshold at 30 seconds (`replay_lag`)
- [ ] Go application uses separate `primary` and `replica` connection pools
- [ ] Read-your-writes pattern implemented for post-write reads
- [ ] Failover tested: promote standby manually, verify app reconnects within `ConnMaxLifetime`
- [ ] Circuit breaker or fallback routes reads to primary when replica unreachable
- [ ] WAL archiving enabled if PITR required (`archive_mode = on`, `archive_command`)
