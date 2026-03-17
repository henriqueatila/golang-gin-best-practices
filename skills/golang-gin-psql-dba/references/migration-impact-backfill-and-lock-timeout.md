# Migration Impact — Batched Backfill, NOT VALID, lock_timeout, and Checklists

## Batched Backfill Pattern in Go

Never backfill millions of rows in a single `UPDATE`. Use cursor-based batches — O(batch_size) per batch regardless of position.

```sql
-- Cursor-based: uses PK as cursor to avoid OFFSET penalty
UPDATE users
SET    status = 'active'
WHERE  id IN (
    SELECT id FROM users
    WHERE  status IS NULL
    AND    id > $1
    ORDER  BY id
    LIMIT  $2
)
RETURNING id;
```

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

func BackfillUsersStatus(ctx context.Context, db *sqlx.DB, logger *slog.Logger) error {
    const batchSQL = `
        UPDATE users
        SET    status = 'active'
        WHERE  id IN (
            SELECT id FROM users
            WHERE  status IS NULL AND id > $1
            ORDER  BY id LIMIT $2
        )
        RETURNING id`

    var cursor int64
    var total, batch int

    for {
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
            break
        }
        total += count
        batch++
        logger.Info("backfill progress",
            slog.Int("batch", batch), slog.Int("total_rows", total), slog.Int64("cursor", cursor))
        time.Sleep(10 * time.Millisecond)
    }

    logger.Info("backfill complete", slog.Int("total_rows", total), slog.Int("batches", batch))
    return nil
}
```

**Design notes:** `ctx.Done()` respects graceful shutdown. `RETURNING id` advances cursor without a second query. Each batch commits independently — resumable on failure.

---

## NOT VALID Constraint Pattern

`NOT VALID` adds a constraint without scanning existing rows. Enforced immediately on new inserts/updates.

**Phase 1 — Add constraint (fast, metadata-level lock):**

```sql
ALTER TABLE orders
    ADD CONSTRAINT chk_orders_amount_positive CHECK (amount > 0) NOT VALID;

ALTER TABLE order_items
    ADD CONSTRAINT fk_order_items_orders
    FOREIGN KEY (order_id) REFERENCES orders (id) NOT VALID;
```

**Phase 2 — Validate (SHARE UPDATE EXCLUSIVE — allows concurrent writes):**

```sql
ALTER TABLE orders VALIDATE CONSTRAINT chk_orders_amount_positive;
ALTER TABLE order_items VALIDATE CONSTRAINT fk_order_items_orders;
```

**Check for unvalidated constraints:**

```sql
SELECT conname, conrelid::regclass AS table_name, contype
FROM   pg_constraint
WHERE  NOT convalidated
ORDER  BY conrelid::regclass::text, conname;
```

Once validated, PostgreSQL uses the constraint for query optimization.


> For lock_timeout strategy, Go migration runner with retry, checklist, and common mistakes: see [migration-impact-lock-timeout.md](migration-impact-lock-timeout.md).
