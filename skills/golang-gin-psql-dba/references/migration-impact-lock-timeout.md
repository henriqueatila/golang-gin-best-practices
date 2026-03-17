# Migration Impact — lock_timeout Strategy and Migration Checklist

## lock_timeout Strategy

`lock_timeout` causes a statement to fail rather than wait indefinitely — your primary defense against outage cascades.

### Always Set Before Migrations

```sql
-- At the top of every migration file
SET lock_timeout = '5s';
SET statement_timeout = '120s';

ALTER TABLE users ADD COLUMN ...;
```

If the lock cannot be acquired within 5 seconds, PostgreSQL raises `ERROR: canceling statement due to lock timeout` — far better than queuing all subsequent queries.

### Go Migration Wrapper with Retry

```go
// internal/migration/runner.go
package migration

import (
    "context"
    "fmt"
    "log/slog"
    "time"

    "github.com/jmoiron/sqlx"
)

type MigrationFn func(ctx context.Context, tx *sqlx.Tx) error

func RunWithLockTimeout(
    ctx context.Context, db *sqlx.DB, lockTimeout string, maxRetries int,
    logger *slog.Logger, migrateFn MigrationFn,
) error {
    var lastErr error
    for attempt := 1; attempt <= maxRetries; attempt++ {
        err := runOnce(ctx, db, lockTimeout, migrateFn)
        if err == nil {
            return nil
        }
        if isLockTimeout(err) {
            logger.Warn("lock timeout, retrying",
                slog.Int("attempt", attempt), slog.String("lock_timeout", lockTimeout))
            lastErr = err
            time.Sleep(time.Duration(attempt) * 2 * time.Second)
            continue
        }
        return fmt.Errorf("migration failed: %w", err)
    }
    return fmt.Errorf("migration failed after %d attempts: %w", maxRetries, lastErr)
}

func runOnce(ctx context.Context, db *sqlx.DB, lockTimeout string, migrateFn MigrationFn) error {
    tx, err := db.BeginTxx(ctx, nil)
    if err != nil {
        return fmt.Errorf("begin tx: %w", err)
    }
    defer tx.Rollback() //nolint:errcheck
    if _, err := tx.ExecContext(ctx, "SET LOCAL lock_timeout = '"+lockTimeout+"'"); err != nil {
        return fmt.Errorf("set lock_timeout: %w", err)
    }
    if err := migrateFn(ctx, tx); err != nil {
        return err
    }
    return tx.Commit()
}

func isLockTimeout(err error) bool {
    if err == nil {
        return false
    }
    s := err.Error()
    for _, sub := range []string{"lock timeout", "55P03", "lock_not_available"} {
        if len(s) >= len(sub) {
            for i := 0; i <= len(s)-len(sub); i++ {
                if s[i:i+len(sub)] == sub {
                    return true
                }
            }
        }
    }
    return false
}
```

### Setting lock_timeout in golang-migrate SQL files

```sql
-- 000015_add_phone_column.up.sql
SET LOCAL lock_timeout = '5s';
ALTER TABLE users ADD COLUMN phone TEXT;
```

---

## Migration Checklist

### Pre-Migration
- [ ] Query table size: `SELECT pg_size_pretty(pg_relation_size('orders')), COUNT(*) FROM orders`
- [ ] Identify long-running transactions: `SELECT pid, now() - xact_start AS duration FROM pg_stat_activity WHERE xact_start IS NOT NULL ORDER BY duration DESC LIMIT 10`
- [ ] Check replication lag: `SELECT client_addr, replay_lag FROM pg_stat_replication`
- [ ] Confirm `lock_timeout` is set in migration file
- [ ] Verify index creation uses `CONCURRENTLY`
- [ ] Verify FK/CHECK constraints use `NOT VALID` for large tables
- [ ] Test migration on staging first

### During Migration
- [ ] Monitor lock queues: `SELECT count(*) FROM pg_stat_activity WHERE wait_event_type = 'Lock'`
- [ ] Find blockers: `SELECT pg_blocking_pids(pid) FROM pg_stat_activity WHERE wait_event_type = 'Lock'`

### Post-Migration
- [ ] Run `ANALYZE tablename` to update planner statistics
- [ ] Validate `NOT VALID` constraints: `SELECT conname FROM pg_constraint WHERE NOT convalidated`
- [ ] Check for INVALID indexes via `pg_index.indisvalid`
- [ ] Confirm replication lag recovered

## Common Mistakes

1. **Running migrations during peak traffic** — `ACCESS EXCLUSIVE` queues behind long reads; every subsequent query queues behind the migration.
2. **Forgetting CONCURRENTLY on index creation** — `CREATE INDEX` holds SHARE lock blocking all DML.
3. **Adding NOT NULL without the 3-step pattern** — full table scan under `ACCESS EXCLUSIVE`.
4. **Not setting lock_timeout** — migration waits indefinitely; cascade fills connection pool.
5. **Validating NOT VALID in same transaction as adding** — validation must be a separate migration to get the `SHARE UPDATE EXCLUSIVE` benefit.
6. **Backfilling in a single UPDATE** — holds row locks on all rows for minutes, blows replication lag.
7. **Running CREATE INDEX CONCURRENTLY inside a transaction block** — raises an error; use `-- +migrate no transaction` or a raw psql script outside the migration framework.
