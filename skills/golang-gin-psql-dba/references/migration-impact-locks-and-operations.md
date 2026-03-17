# Migration Impact — Lock Hierarchy and Safe vs Unsafe Operations

Every `ALTER TABLE` on a live PostgreSQL database acquires a lock. The lock level and duration determine whether your migration is invisible to users or causes a full outage.

> **Scope:** PostgreSQL 14+. For migration tooling (golang-migrate CLI), see the **golang-gin-database** skill's `references/migrations.md`.

## PostgreSQL Lock Hierarchy

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

### Why ACCESS EXCLUSIVE Is the Danger Zone

`ACCESS EXCLUSIVE` conflicts with **every other lock including plain SELECT**. On a busy table:

1. Your `ALTER TABLE` waits for all active transactions to finish.
2. While waiting, all new queries that touch the table queue behind it.
3. The queue grows. Connections exhaust. The database appears to hang.

Even a fast `ALTER TABLE` (milliseconds) can cause a 30-second outage if it has to wait for a long-running read. This is why `SET lock_timeout = '5s'` is mandatory — fail fast rather than queue.

---

## Safe vs Unsafe ALTER TABLE Operations

"Safe online" = completes in milliseconds (metadata-only change). "Unsafe" = full table scan or rewrite.

| Operation | Lock Level | Duration | Safe Online? | Notes |
|---|---|---|---|---|
| `ADD COLUMN` — nullable, no default | ACCESS EXCLUSIVE | Fast | Yes | Safest column addition |
| `ADD COLUMN ... NOT NULL DEFAULT expr` (PG 11+) | ACCESS EXCLUSIVE | Fast | Yes | PG 11+ avoids table rewrite |
| `DROP COLUMN` | ACCESS EXCLUSIVE | Fast | Yes | Data stays until `VACUUM` |
| `SET NOT NULL` (bare) | ACCESS EXCLUSIVE | Slow — full table scan | No | Use 3-step NOT VALID pattern |
| `DROP NOT NULL` | ACCESS EXCLUSIVE | Fast | Yes | Always safe |
| `ALTER COLUMN TYPE` (compatible cast) | ACCESS EXCLUSIVE | Slow — full table rewrite | No | Use new column + backfill pattern |
| `SET DEFAULT` / `DROP DEFAULT` | ACCESS EXCLUSIVE | Fast | Yes | Does not affect existing rows |
| `ADD CONSTRAINT CHECK` (validated) | ACCESS EXCLUSIVE | Slow — full table scan | No | Use NOT VALID + VALIDATE pattern |
| `ADD CONSTRAINT CHECK NOT VALID` | ACCESS EXCLUSIVE | Fast | Yes | Does not validate existing rows |
| `VALIDATE CONSTRAINT` | SHARE UPDATE EXCLUSIVE | Slow — full scan, allows writes | Yes | Safe to run on live table |
| `ADD CONSTRAINT FOREIGN KEY` (validated) | SHARE ROW EXCLUSIVE | Slow — validates all rows | No | Use NOT VALID + VALIDATE pattern |
| `ADD CONSTRAINT FOREIGN KEY NOT VALID` | SHARE ROW EXCLUSIVE | Fast | Yes | Validates only new/updated rows |
| `CREATE INDEX` | SHARE | Slow — blocks writes | No | Always use CONCURRENTLY |
| `CREATE INDEX CONCURRENTLY` | SHARE UPDATE EXCLUSIVE | Slow — two passes, allows writes | Yes | Cannot run in transaction block |
| `DROP INDEX CONCURRENTLY` | SHARE UPDATE EXCLUSIVE | Slow | Yes | Safer than non-concurrent drop |
| `RENAME COLUMN` | ACCESS EXCLUSIVE | Fast | No (app break) | Use multi-step expand/contract |

---

## Zero-Downtime Patterns

### A. Add NOT NULL Column

**3-Step Pattern:**

```sql
-- migrations/0012_add_users_status.up.sql
-- Step 1: Add column nullable (fast — metadata only)
ALTER TABLE users ADD COLUMN status TEXT;
-- (deploy new app code that writes status on all INSERT/UPDATE)
```

```sql
-- migrations/0013_backfill_users_status.up.sql
-- Step 2: Backfill existing rows (use batched Go function)
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

### B. Change Column Type — Expand/Contract Pattern

```sql
-- Phase 1: Add new column (nullable)
ALTER TABLE orders ADD COLUMN total_cents BIGINT;

-- Phase 3: Sync trigger during backfill window
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

-- Phase 4: After backfill complete and app reads from new column
DROP TRIGGER trg_orders_sync_total_cents ON orders;
DROP FUNCTION orders_sync_total_cents();

-- Phase 5: Drop old column when no app code references it
ALTER TABLE orders DROP COLUMN total;
ALTER TABLE orders RENAME COLUMN total_cents TO total;
```

**Key rule:** Deploy app changes that write to both columns before backfilling. Deploy the read switchover after backfill.

### C. Rename Column — Expand/Contract

Add new column → backfill from old → `NOT VALID` constraint → `VALIDATE CONSTRAINT` → switch app to read from new column → drop old column. Requires 2 deploys minimum.

### D. Add Foreign Key

```sql
-- Step 1: Add FK as NOT VALID (fast — no row scan)
ALTER TABLE orders
    ADD CONSTRAINT fk_orders_users FOREIGN KEY (user_id) REFERENCES users (id)
    NOT VALID;

-- Step 2: Validate in a separate migration / transaction
-- SHARE UPDATE EXCLUSIVE — allows concurrent reads AND writes
ALTER TABLE orders VALIDATE CONSTRAINT fk_orders_users;
```

### E. Create Index

```sql
-- WRONG — holds SHARE lock (blocks INSERT/UPDATE/DELETE)
CREATE INDEX idx_orders_user_id ON orders (user_id);

-- CORRECT — allows all DML, takes ~2x longer
CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders (user_id);
DROP INDEX CONCURRENTLY idx_orders_user_id;
```

**Constraints:** Cannot run inside an explicit transaction block. If interrupted, leaves an INVALID index — drop manually before retrying.
