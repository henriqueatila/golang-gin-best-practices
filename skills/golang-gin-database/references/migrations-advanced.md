# Migrations — Zero-Downtime Patterns

Zero-downtime migration patterns: column addition, rename, non-blocking index creation, and large table backfills.

For seeding, rollbacks, and example SQL files: see [migrations-examples.md](migrations-examples.md).

> **Architectural recommendation:** golang-migrate is not part of the Gin framework. These are mainstream Go community patterns.

## Zero-Downtime Migrations

Schema changes that require zero downtime (no table locks, backward-compatible for rolling deploys).

### Column Addition (Safe)

```sql
-- 000005_add_users_bio.up.sql
-- Adding a nullable column is instant — no table rewrite in PostgreSQL 11+
ALTER TABLE users ADD COLUMN bio TEXT;
```

```sql
-- 000005_add_users_bio.down.sql
ALTER TABLE users DROP COLUMN bio;
```

### Column Rename (Multi-Step)

Never rename directly — old app code still reads the old name during rollout.

```sql
-- Step 1: add new column (deploy v1 — reads old column)
-- 000006_add_users_full_name.up.sql
ALTER TABLE users ADD COLUMN full_name VARCHAR(200);
UPDATE users SET full_name = name WHERE full_name IS NULL;

-- Step 2: make old column obsolete (deploy v2 — reads new column)
-- 000007_drop_users_name.up.sql
ALTER TABLE users DROP COLUMN name;
ALTER TABLE users RENAME COLUMN full_name TO name;
```

### Index Creation (Non-Blocking)

```sql
-- 000008_add_users_email_index.up.sql
-- CONCURRENTLY prevents table lock — safe in production
-- Note: cannot run inside a transaction block
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_users_email ON users(email);
```

```sql
-- 000008_add_users_email_index.down.sql
DROP INDEX CONCURRENTLY IF EXISTS idx_users_email;
```

**Warning:** golang-migrate wraps each migration in a transaction by default. `CREATE INDEX CONCURRENTLY` cannot run inside a transaction. Workaround — run outside the migrate library:

```go
func createIndexConcurrently(db *sql.DB) error {
    _, err := db.Exec(`CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_users_email ON users(email)`)
    return err
}
```

### Backfill Large Tables

```sql
-- 000009_backfill_users_role.up.sql
-- Batch update to avoid long-running lock
DO $$
DECLARE
    batch_size INT := 1000;
    updated INT;
BEGIN
    LOOP
        UPDATE users SET role = 'user'
        WHERE role IS NULL
        AND id IN (SELECT id FROM users WHERE role IS NULL LIMIT batch_size);
        GET DIAGNOSTICS updated = ROW_COUNT;
        EXIT WHEN updated = 0;
        PERFORM pg_sleep(0.01); -- brief pause between batches
    END LOOP;
END $$;
```

See [migrations-examples.md](migrations-examples.md) for seeding, rollback strategies, and example SQL migration files.
