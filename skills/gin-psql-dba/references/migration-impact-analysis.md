# Migration Impact Analysis Reference

Every `ALTER TABLE` on a live PostgreSQL database acquires a lock. The lock level and duration determine whether your migration is invisible to users or causes a full outage. This reference covers PostgreSQL's lock hierarchy, which operations are safe online, zero-downtime patterns for every dangerous operation, batched backfill in Go, the NOT VALID constraint technique, lock_timeout defense, and a pre/post migration checklist. Use when evaluating any schema change against a production database running a Gin API.

> **Scope:** PostgreSQL 14+. All Go code uses `log/slog`, `context.Context`, `fmt.Errorf`, and `sqlx` — no GORM. For migration tooling (golang-migrate CLI, startup vs CI/CD strategy), see the **gin-database** skill's `references/migrations.md`.

## Table of Contents

1. [PostgreSQL Lock Hierarchy](#postgresql-lock-hierarchy)
2. [Safe vs Unsafe ALTER TABLE Operations](#safe-vs-unsafe-alter-table-operations)
3. [Zero-Downtime Patterns](#zero-downtime-patterns)
   - [Add NOT NULL Column](#a-add-not-null-column)
   - [Change Column Type](#b-change-column-type)
   - [Rename Column](#c-rename-column)
   - [Add Foreign Key](#d-add-foreign-key)
   - [Create Index](#e-create-index)
   - [Backfill Large Tables](#f-backfill-large-tables)
4. [Batched Backfill Pattern in Go](#batched-backfill-pattern-in-go)
5. [NOT VALID Constraint Pattern](#not-valid-constraint-pattern)
6. [lock_timeout Strategy](#lock_timeout-strategy)
7. [Migration Checklist](#migration-checklist)
8. [Common Mistakes](#common-mistakes)

---

## PostgreSQL Lock Hierarchy

PostgreSQL has 8 lock levels. Each level conflicts with some and is compatible with others. Migrations that acquire heavy locks block all conflicting operations — including reads — for the duration.

| # | Lock Level | Typical Acquired By | Conflicts With |
|---|---|---|---|
| 1 | `ACCESS SHARE` | `SELECT` | ACCESS EXCLUSIVE only |
| 2 | `ROW SHARE` | `SELECT FOR UPDATE / SHARE` | EXCLUSIVE, ACCESS EXCLUSIVE |
| 3 | `ROW EXCLUSIVE` | `INSERT`, `UPDATE`, `DELETE` | SHARE, SHARE ROW EXCLUSIVE, EXCLUSIVE, ACCESS EXCLUSIVE |
| 4 | `SHARE UPDATE EXCLUSIVE` | `VACUUM`, `ANALYZE`, `CREATE INDEX CONCURRENTLY` | Levels 4–8 |
| 5 | `SHARE` | `CREATE INDEX` (non-concurrent) | ROW EXCLUSIVE, levels 4–8 |
| 6 | `SHARE ROW EXCLUSIVE` | `CREATE TRIGGER`, some `ALTER TABLE` | ROW EXCLUSIVE, levels 4–8 |
| 7 | `EXCLUSIVE` | Rare — explicit `LOCK TABLE ... EXCLUSIVE` | Levels 2–8 |
| 8 | `ACCESS EXCLUSIVE` | Most `ALTER TABLE`, `DROP TABLE`, `TRUNCATE`, `VACUUM FULL` | ALL levels — blocks even SELECT |

### Lock Conflict Matrix (abbreviated)

| Held \ Requested | ACCESS SHARE | ROW EXCLUSIVE | SHARE UPDATE EXCL | SHARE | ACCESS EXCLUSIVE |
|---|---|---|---|---|---|
| ACCESS SHARE | OK | OK | OK | OK | **BLOCK** |
| ROW EXCLUSIVE | OK | OK | **BLOCK** | **BLOCK** | **BLOCK** |
| SHARE UPDATE EXCL | OK | **BLOCK** | **BLOCK** | **BLOCK** | **BLOCK** |
| SHARE | OK | **BLOCK** | **BLOCK** | OK | **BLOCK** |
| ACCESS EXCLUSIVE | **BLOCK** | **BLOCK** | **BLOCK** | **BLOCK** | **BLOCK** |

### Why ACCESS EXCLUSIVE Is the Danger Zone

`ACCESS EXCLUSIVE` is acquired by the majority of `ALTER TABLE` variants. It conflicts with **every other lock including plain SELECT**. On a busy table:

1. Your `ALTER TABLE` waits for all active transactions to finish before acquiring the lock.
2. While waiting, all new queries that touch the table queue behind it.
3. The queue grows. Connections exhaust. The database appears to hang.

Even a fast `ALTER TABLE` (milliseconds) can cause a 30-second outage if it has to wait 30 seconds for a long-running read to finish. This is why `SET lock_timeout = '5s'` is mandatory — fail fast rather than queue.

---

## Safe vs Unsafe ALTER TABLE Operations

"Safe online" means the operation completes in milliseconds (metadata-only change) and holds the lock for an imperceptible duration. "Unsafe" means a full table scan or rewrite, blocking all traffic for seconds to minutes on large tables.

| Operation | Lock Level | Duration | Safe Online? | Notes |
|---|---|---|---|---|
| `ADD COLUMN` — nullable, no default | ACCESS EXCLUSIVE | **Fast** — metadata only | Yes | Safest column addition |
| `ADD COLUMN ... DEFAULT expr` (PG 11+) | ACCESS EXCLUSIVE | **Fast** — stored default | Yes | PG 11+ avoids table rewrite for non-volatile defaults |
| `ADD COLUMN ... NOT NULL DEFAULT expr` (PG 11+) | ACCESS EXCLUSIVE | **Fast** — stored default | Yes | Same optimization as above |
| `DROP COLUMN` | ACCESS EXCLUSIVE | **Fast** — marks column invisible | Yes | Data stays until `VACUUM`; column name is reserved |
| `SET NOT NULL` (bare) | ACCESS EXCLUSIVE | **Slow** — full table scan | No | Use 3-step NOT VALID pattern instead |
| `DROP NOT NULL` | ACCESS EXCLUSIVE | **Fast** — metadata only | Yes | Always safe |
| `ALTER COLUMN TYPE` (compatible cast) | ACCESS EXCLUSIVE | **Slow** — full table rewrite | No | Use new column + backfill pattern |
| `ALTER COLUMN TYPE` (TEXT → TEXT) | ACCESS EXCLUSIVE | **Fast** — no rewrite needed | Yes | Only if cast is binary-compatible |
| `SET DEFAULT` / `DROP DEFAULT` | ACCESS EXCLUSIVE | **Fast** — metadata only | Yes | Does not affect existing rows |
| `ADD CONSTRAINT CHECK` (validated) | ACCESS EXCLUSIVE | **Slow** — full table scan | No | Use NOT VALID + VALIDATE pattern |
| `ADD CONSTRAINT CHECK NOT VALID` | ACCESS EXCLUSIVE | **Fast** — no scan | Yes | Does not validate existing rows |
| `VALIDATE CONSTRAINT` | SHARE UPDATE EXCLUSIVE | **Slow** — full scan, allows writes | Yes | Safe to run on live table |
| `ADD CONSTRAINT FOREIGN KEY` (validated) | SHARE ROW EXCLUSIVE | **Slow** — validates all rows | No | Use NOT VALID + VALIDATE pattern |
| `ADD CONSTRAINT FOREIGN KEY NOT VALID` | SHARE ROW EXCLUSIVE | **Fast** — no scan | Yes | Validates only new/updated rows |
| `CREATE INDEX` | SHARE | **Slow** — full index build, blocks writes | No | Always use CONCURRENTLY |
| `CREATE INDEX CONCURRENTLY` | SHARE UPDATE EXCLUSIVE | **Slow** — two passes, allows writes | Yes | Takes ~2× longer; cannot run in transaction block |
| `DROP INDEX` | ACCESS EXCLUSIVE | **Fast** — metadata | No | Use `DROP INDEX CONCURRENTLY` |
| `DROP INDEX CONCURRENTLY` | SHARE UPDATE EXCLUSIVE | **Slow** — waits for readers | Yes | Safer than non-concurrent drop |
| `RENAME COLUMN` | ACCESS EXCLUSIVE | **Fast** — metadata | No (app break) | Breaks existing queries; use multi-step expand/contract |
| `RENAME TABLE` | ACCESS EXCLUSIVE | **Fast** — metadata | No (app break) | Same concern as RENAME COLUMN |

---

## Zero-Downtime Patterns

### A. Add NOT NULL Column

**Problem:** `ALTER TABLE t ADD COLUMN c TEXT NOT NULL` does a full table scan to verify no nulls exist, holding `ACCESS EXCLUSIVE` the entire time.

**3-Step Pattern:**

```sql
-- migrations/0012_add_users_status.up.sql

-- Step 1: Add column nullable (fast — metadata only)
ALTER TABLE users ADD COLUMN status TEXT;

-- (deploy new app code that writes status on all INSERT/UPDATE)
```

```sql
-- migrations/0013_backfill_users_status.up.sql

-- Step 2: Backfill existing rows (run via batched Go function — see section 4)
-- After backfill completes:
UPDATE users SET status = 'active' WHERE status IS NULL;
```

```sql
-- migrations/0014_users_status_not_null.up.sql

-- Step 3a: Add CHECK constraint as NOT VALID (fast — no scan)
ALTER TABLE users
    ADD CONSTRAINT users_status_not_null CHECK (status IS NOT NULL) NOT VALID;

-- Step 3b: Validate in separate transaction (scans but allows concurrent writes)
ALTER TABLE users VALIDATE CONSTRAINT users_status_not_null;
```

After `VALIDATE CONSTRAINT`, PostgreSQL knows the constraint holds for all rows and will use it in query planning. At this point you can also convert the nullable column to a proper `NOT NULL` constraint — though `CHECK NOT NULL` is functionally equivalent and avoids the `SET NOT NULL` scan.

---

### B. Change Column Type

**Problem:** `ALTER COLUMN c TYPE new_type` rewrites the entire table, holding `ACCESS EXCLUSIVE` for the full duration.

**Expand/Contract Pattern:**

```sql
-- Phase 1: Add new column (nullable)
ALTER TABLE orders ADD COLUMN total_cents BIGINT;

-- Phase 2: Backfill (batched — see section 4)
-- Convert total (NUMERIC) -> total_cents (BIGINT)

-- Phase 3: Add write trigger so new rows stay in sync during cutover
```

```sql
-- Trigger to keep new column in sync during backfill window
CREATE OR REPLACE FUNCTION orders_sync_total_cents()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    NEW.total_cents := (NEW.total * 100)::BIGINT;
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_orders_sync_total_cents
BEFORE INSERT OR UPDATE ON orders
FOR EACH ROW EXECUTE FUNCTION orders_sync_total_cents();
```

```sql
-- Phase 4: After backfill complete and app reads from new column
DROP TRIGGER trg_orders_sync_total_cents ON orders;
DROP FUNCTION orders_sync_total_cents();

-- Phase 5: Drop old column when no app code references it
ALTER TABLE orders DROP COLUMN total;
ALTER TABLE orders RENAME COLUMN total_cents TO total;
```

**Key rule:** Deploy app changes that write to both columns before backfilling. Deploy the read switchover after backfill. Only then drop the old column.

---

### C. Rename Column

**Problem:** `RENAME COLUMN` is a fast metadata change but breaks all queries, ORMs, and application code that references the old name in the same deploy.

**Multi-Step Expand/Contract:**

```sql
-- Step 1: Add new column (nullable)
ALTER TABLE users ADD COLUMN full_name TEXT;

-- Step 2: Backfill from old column
UPDATE users SET full_name = name WHERE full_name IS NULL;

-- Step 3: Add NOT VALID constraint so new rows must populate it
ALTER TABLE users
    ADD CONSTRAINT users_full_name_not_null CHECK (full_name IS NOT NULL) NOT VALID;
ALTER TABLE users VALIDATE CONSTRAINT users_full_name_not_null;
```

```sql
-- Step 4: (After app code writes full_name and reads full_name)
-- Drop old column
ALTER TABLE users DROP COLUMN name;
```

This pattern takes 2 deploys minimum: one to add the new column and update writes, one to remove the old column after reads are migrated.

---

### D. Add Foreign Key

**Problem:** `ADD CONSTRAINT ... FOREIGN KEY` validates every row in the child table, holding `SHARE ROW EXCLUSIVE` on both tables.

```sql
-- Step 1: Add FK as NOT VALID (fast — no row scan)
-- SHARE ROW EXCLUSIVE held only briefly
ALTER TABLE orders
    ADD CONSTRAINT fk_orders_users FOREIGN KEY (user_id) REFERENCES users (id)
    NOT VALID;

-- Step 2: Validate in a separate migration / transaction
-- SHARE UPDATE EXCLUSIVE — allows concurrent reads AND writes
ALTER TABLE orders VALIDATE CONSTRAINT fk_orders_users;
```

After validation, PostgreSQL enforces the constraint on all future writes. The two-step approach means neither step blocks traffic for long.

---

### E. Create Index

**Rule: always use CONCURRENTLY.** Never omit it on a table with live traffic.

```sql
-- WRONG — holds SHARE lock (blocks INSERT/UPDATE/DELETE) for full build duration
CREATE INDEX idx_orders_user_id ON orders (user_id);

-- CORRECT — holds SHARE UPDATE EXCLUSIVE (allows all DML) but takes ~2x longer
CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders (user_id);

-- Drop also uses CONCURRENTLY
DROP INDEX CONCURRENTLY idx_orders_user_id;
```

**Constraints with CONCURRENTLY:**
- Cannot run inside an explicit transaction block (`BEGIN` ... `COMMIT`)
- If interrupted, leaves an INVALID index — check with `\d orders` or query `pg_indexes` and drop manually before retrying
- Check for invalid indexes: `SELECT indexname FROM pg_indexes JOIN pg_class ON indexname = relname WHERE pg_class.relam IS NOT NULL` — or use `pg_index.indisvalid = false`

---

### F. Backfill Large Tables

Never backfill millions of rows in a single `UPDATE`. It holds a row-level lock on every updated row for the full statement duration, causes table bloat, and can run for minutes.

Use batched updates — see the full Go implementation in [section 4](#batched-backfill-pattern-in-go).

**Cursor-based batch SQL (preferred over OFFSET for large tables):**

```sql
-- Cursor-based: uses the primary key as a cursor to avoid OFFSET penalty
UPDATE users
SET    status = 'active'
WHERE  id IN (
    SELECT id FROM users
    WHERE  status IS NULL
    AND    id > $1          -- cursor: last processed id
    ORDER  BY id
    LIMIT  $2               -- batch size
)
RETURNING id;
```

**Why cursor over OFFSET:** `LIMIT N OFFSET M` scans and discards M rows on every batch. Cursor-based (keyset pagination) is O(batch_size) per batch regardless of position.

---

## Batched Backfill Pattern in Go

```go
// internal/migration/backfill.go
package migration

import (
    "context"
    "fmt"
    "log/slog"
    "time"

    "github.com/jmoiron/sqlx"
)

const defaultBatchSize = 1000

// BackfillUsersStatus sets status = 'active' for all rows where status IS NULL.
// Uses cursor-based batching to avoid long-held locks and OFFSET performance penalty.
func BackfillUsersStatus(ctx context.Context, db *sqlx.DB, logger *slog.Logger) error {
    const batchSQL = `
        UPDATE users
        SET    status = 'active'
        WHERE  id IN (
            SELECT id FROM users
            WHERE  status IS NULL
            AND    id > $1
            ORDER  BY id
            LIMIT  $2
        )
        RETURNING id`

    var (
        cursor   int64 // last processed id; 0 = start from beginning
        total    int
        batch    int
    )

    for {
        // Respect context cancellation (graceful shutdown, timeout)
        select {
        case <-ctx.Done():
            return fmt.Errorf("backfill cancelled after %d rows: %w", total, ctx.Err())
        default:
        }

        rows, err := db.QueryContext(ctx, batchSQL, cursor, defaultBatchSize)
        if err != nil {
            return fmt.Errorf("backfill batch %d: %w", batch, err)
        }

        var ids []int64
        for rows.Next() {
            var id int64
            if err := rows.Scan(&id); err != nil {
                rows.Close()
                return fmt.Errorf("scan id batch %d: %w", batch, err)
            }
            ids = append(ids, id)
            if id > cursor {
                cursor = id
            }
        }
        rows.Close()

        if err := rows.Err(); err != nil {
            return fmt.Errorf("rows error batch %d: %w", batch, err)
        }

        count := len(ids)
        if count == 0 {
            break // backfill complete
        }

        total += count
        batch++

        logger.Info("backfill progress",
            slog.Int("batch", batch),
            slog.Int("batch_rows", count),
            slog.Int("total_rows", total),
            slog.Int64("cursor", cursor),
        )

        // Optional: small sleep to reduce I/O pressure on busy replicas
        time.Sleep(10 * time.Millisecond)
    }

    logger.Info("backfill complete", slog.Int("total_rows", total), slog.Int("batches", batch))
    return nil
}
```

**Usage from a migration runner or CLI tool:**

```go
func runBackfill(ctx context.Context, db *sqlx.DB) error {
    logger := slog.Default()
    logger.Info("starting users status backfill")

    if err := migration.BackfillUsersStatus(ctx, db, logger); err != nil {
        return fmt.Errorf("backfill users status: %w", err)
    }
    return nil
}
```

**Design notes:**
- `ctx.Done()` check before each batch respects graceful shutdown signals.
- Cursor advances to the maximum `id` seen in each batch — safe even if rows are deleted mid-backfill.
- `RETURNING id` lets us advance the cursor without a second query.
- `time.Sleep(10ms)` is optional; remove for maximum throughput in a maintenance window.
- Run this outside of a transaction block — each batch commits independently, so a failure mid-backfill is resumable (restart from cursor = 0; already-set rows skip the `WHERE status IS NULL` filter).

---

## NOT VALID Constraint Pattern

`NOT VALID` is a PostgreSQL modifier that adds a constraint without scanning existing rows. The constraint is enforced on all new inserts and updates immediately, but existing rows are not checked until `VALIDATE CONSTRAINT` is run.

### When to Use NOT VALID

- Adding a `CHECK` constraint on a large table
- Adding a `FOREIGN KEY` constraint
- Implementing the NOT NULL pattern (step 3 above)
- Any time you need the constraint active for new data without blocking existing traffic

### Two-Phase: Add (Fast) → Validate (Non-Blocking)

**Phase 1 — Add constraint (fast, metadata-level lock):**

```sql
-- CHECK constraint
ALTER TABLE orders
    ADD CONSTRAINT chk_orders_amount_positive
    CHECK (amount > 0) NOT VALID;

-- FOREIGN KEY constraint
ALTER TABLE order_items
    ADD CONSTRAINT fk_order_items_orders
    FOREIGN KEY (order_id) REFERENCES orders (id)
    NOT VALID;
```

These complete in milliseconds. `ACCESS EXCLUSIVE` / `SHARE ROW EXCLUSIVE` is held only briefly (no row scan).

**Phase 2 — Validate (slow scan, but only holds SHARE UPDATE EXCLUSIVE):**

```sql
-- SHARE UPDATE EXCLUSIVE — concurrent reads AND writes proceed normally
ALTER TABLE orders VALIDATE CONSTRAINT chk_orders_amount_positive;
ALTER TABLE order_items VALIDATE CONSTRAINT fk_order_items_orders;
```

`VALIDATE CONSTRAINT` scans the table but holds only `SHARE UPDATE EXCLUSIVE`, which allows concurrent INSERT, UPDATE, and DELETE. This is safe on a live database.

### Query Planner Benefit

Once validated, PostgreSQL can use the constraint for query optimization (e.g., constraint exclusion, provability of NOT NULL). An unvalidated `NOT VALID` constraint is enforced for new rows but invisible to the planner.

### Checking Constraint Status

```sql
-- Find NOT VALID constraints still pending validation
SELECT conname, conrelid::regclass AS table_name, contype
FROM   pg_constraint
WHERE  NOT convalidated
ORDER  BY conrelid::regclass::text, conname;
```

---

## lock_timeout Strategy

`lock_timeout` causes a statement to fail with an error rather than wait indefinitely for a lock. This is your primary defense against migration-induced outage cascades.

### Always Set Before Migrations

```sql
-- At the top of every migration file
SET lock_timeout = '5s';
SET statement_timeout = '120s';  -- fail-safe for unexpectedly long operations

-- The rest of your migration
ALTER TABLE users ADD COLUMN ...;
```

If the lock cannot be acquired within 5 seconds, PostgreSQL raises:
```
ERROR:  canceling statement due to lock timeout
```

This is far better than the migration queuing behind a long-running transaction, causing every subsequent query to queue behind the migration.

### Go Migration Wrapper with lock_timeout and Retry

```go
// internal/migration/runner.go
package migration

import (
    "context"
    "errors"
    "fmt"
    "log/slog"
    "time"

    "github.com/jmoiron/sqlx"
)

// MigrationFn is a function that executes migration SQL against a transaction.
type MigrationFn func(ctx context.Context, tx *sqlx.Tx) error

// RunWithLockTimeout executes migrateFn inside a transaction with lock_timeout set.
// Retries up to maxRetries times if a lock timeout error occurs.
func RunWithLockTimeout(
    ctx context.Context,
    db *sqlx.DB,
    lockTimeout string,
    maxRetries int,
    logger *slog.Logger,
    migrateFn MigrationFn,
) error {
    var lastErr error

    for attempt := 1; attempt <= maxRetries; attempt++ {
        err := runOnce(ctx, db, lockTimeout, migrateFn)
        if err == nil {
            return nil
        }

        // Check if this is a lock timeout (retry-able)
        if isLockTimeout(err) {
            logger.Warn("lock timeout acquiring migration lock, retrying",
                slog.Int("attempt", attempt),
                slog.Int("max_retries", maxRetries),
                slog.String("lock_timeout", lockTimeout),
            )
            lastErr = err
            time.Sleep(time.Duration(attempt) * 2 * time.Second) // exponential backoff
            continue
        }

        // Non-retryable error
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

    if err := tx.Commit(); err != nil {
        return fmt.Errorf("commit: %w", err)
    }
    return nil
}

func isLockTimeout(err error) bool {
    // pq driver wraps the SQLSTATE in the error message
    // SQLSTATE 55P03 = lock_not_available
    return err != nil && errors.Is(err, context.Canceled) == false &&
        containsAny(err.Error(), "lock timeout", "55P03", "lock_not_available")
}

func containsAny(s string, subs ...string) bool {
    for _, sub := range subs {
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

**Usage:**

```go
err := migration.RunWithLockTimeout(ctx, db, "5s", 3, logger,
    func(ctx context.Context, tx *sqlx.Tx) error {
        _, err := tx.ExecContext(ctx,
            `ALTER TABLE users ADD COLUMN phone TEXT`)
        return err
    },
)
if err != nil {
    return fmt.Errorf("add phone column: %w", err)
}
```

### Setting lock_timeout in golang-migrate SQL files

Since `SET LOCAL` is scoped to the current transaction and golang-migrate wraps each migration in a transaction:

```sql
-- 000015_add_phone_column.up.sql
SET LOCAL lock_timeout = '5s';

ALTER TABLE users ADD COLUMN phone TEXT;
```

```sql
-- 000015_add_phone_column.down.sql
ALTER TABLE users DROP COLUMN phone;
```

---

## Migration Checklist

### Pre-Migration

- [ ] Query table size and row count: `SELECT pg_size_pretty(pg_relation_size('orders')), COUNT(*) FROM orders`
- [ ] Identify long-running transactions: `SELECT pid, now() - xact_start AS duration, query FROM pg_stat_activity WHERE xact_start IS NOT NULL ORDER BY duration DESC LIMIT 10`
- [ ] Check active connections: `SELECT count(*) FROM pg_stat_activity WHERE state != 'idle'`
- [ ] Check replication lag on replicas: `SELECT client_addr, replay_lag FROM pg_stat_replication`
- [ ] Confirm `lock_timeout` is set in migration file
- [ ] Verify index creation uses `CONCURRENTLY`
- [ ] Verify FK/CHECK constraints use `NOT VALID` for large tables
- [ ] Test migration on staging database first
- [ ] Plan rollback SQL (`.down.sql` files tested)
- [ ] Schedule during lowest traffic window if the operation cannot be made safe

### During Migration

- [ ] Monitor `pg_stat_activity` for lock queues: `SELECT count(*) FROM pg_stat_activity WHERE wait_event_type = 'Lock'`
- [ ] Watch for statements waiting on locks: `SELECT pid, wait_event, query FROM pg_stat_activity WHERE wait_event_type = 'Lock'`
- [ ] If migration hangs: find blocker and evaluate — `SELECT pg_blocking_pids(pid) FROM pg_stat_activity WHERE wait_event_type = 'Lock'`
- [ ] Watch replication lag during heavy backfills

### Post-Migration

- [ ] Run `ANALYZE tablename` to update planner statistics after backfill or major structural change
- [ ] Validate any `NOT VALID` constraints added in earlier phases: `SELECT conname FROM pg_constraint WHERE NOT convalidated`
- [ ] Check for INVALID indexes: `SELECT indexname FROM pg_indexes i JOIN pg_class c ON c.relname = i.indexname WHERE NOT EXISTS (SELECT 1 FROM pg_index WHERE indexrelid = c.oid AND indisvalid)`
- [ ] Confirm replication lag has recovered
- [ ] Verify application error rates are normal
- [ ] Archive down migration files (keep them — they are your rollback)

---

## Common Mistakes

**1. Running migrations during peak traffic**

Even a "fast" metadata-only `ALTER TABLE` acquires `ACCESS EXCLUSIVE`. If a long-running OLAP query is running at migration time, your migration queues — and so does every subsequent query. Schedule migrations during low-traffic windows or use maintenance windows for anything involving table rewrites.

**2. Forgetting CONCURRENTLY on index creation**

`CREATE INDEX` (without CONCURRENTLY) holds a SHARE lock that blocks all INSERT/UPDATE/DELETE for the entire index build. On a 100M-row table this can take minutes. Always use `CREATE INDEX CONCURRENTLY`. Exception: initial migrations on empty tables (pre-launch) where no live traffic exists.

**3. Adding NOT NULL without the 3-step pattern**

`ALTER TABLE t ALTER COLUMN c SET NOT NULL` does a full table scan under `ACCESS EXCLUSIVE`. On a 50M-row table at peak traffic, this can cause a minutes-long outage. Always use the `CHECK NOT VALID` + backfill + `VALIDATE` pattern described in [section 3A](#a-add-not-null-column).

**4. Changing column types directly**

`ALTER COLUMN c TYPE new_type` rewrites the entire table. Even changing `VARCHAR(100)` to `VARCHAR(200)` used to require this (PostgreSQL now optimizes this specific case, but most other changes still rewrite). Use the new column + backfill + drop old column pattern instead.

**5. Not setting lock_timeout**

Without `lock_timeout`, a migration will wait indefinitely for a lock. During the wait, every new query that needs the same table queues behind it. The result is a cascade: the database appears to hang, connection pools fill, and the application becomes unavailable — even though the migration itself may only take 10ms once it finally acquires the lock.

**6. Validating NOT VALID constraints in the same transaction as adding them**

```sql
-- WRONG: validation runs inside same transaction, adds no safety benefit
BEGIN;
ALTER TABLE t ADD CONSTRAINT c CHECK (x > 0) NOT VALID;
ALTER TABLE t VALIDATE CONSTRAINT c;  -- holds ACCESS EXCLUSIVE for scan duration
COMMIT;
```

The point of `NOT VALID` is to split the lock-heavy validation into a **separate migration/transaction** that holds only `SHARE UPDATE EXCLUSIVE`.

**7. Backfilling in a single UPDATE**

`UPDATE orders SET v2 = compute(v1)` on a 20M-row table holds row locks on every updated row for minutes, causes massive WAL generation, and can blow replication lag. Always batch — see [section 4](#batched-backfill-pattern-in-go).

**8. Running CREATE INDEX CONCURRENTLY inside a transaction block**

`CONCURRENTLY` cannot run inside an explicit `BEGIN`/`COMMIT`. golang-migrate wraps migrations in transactions by default. Either run the index creation in a separate migration marked `-- +migrate no transaction` (depends on the tool) or use a raw psql script outside the migration framework for concurrent index builds.
