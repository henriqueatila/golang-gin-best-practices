# Index Strategy — Maintenance, EXPLAIN ANALYZE, and E-commerce Example

## Index Maintenance

### REINDEX CONCURRENTLY (PG 12+)

Rebuilds an index without holding a lock that blocks reads and writes.

```sql
-- Rebuild a single bloated index online
REINDEX INDEX CONCURRENTLY idx_orders_tenant_status_created;

-- Rebuild all indexes on a table online
REINDEX TABLE CONCURRENTLY orders;
```

**Note:** `CONCURRENTLY` requires more disk space (old + new index coexist during rebuild).

### Detecting Bloated Indexes with pgstattuple

```sql
CREATE EXTENSION IF NOT EXISTS pgstattuple;
SELECT index_size, leaf_fragmentation, avg_leaf_density
FROM pgstatindex('idx_orders_tenant_status_created');
-- leaf_fragmentation > 30% or avg_leaf_density < 50% → REINDEX
```

### Unused Index Detection

```sql
SELECT schemaname, tablename, indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    idx_scan AS scans
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelname NOT LIKE '%_pkey'
  AND indexrelname NOT LIKE '%_unique%'
ORDER BY pg_relation_size(indexrelid) DESC;

DROP INDEX CONCURRENTLY idx_products_legacy_column;
```

Run after at least a week of normal load. Drop unused indexes — they cost write overhead with zero read benefit.

### Auto-maintenance with pg_cron

```sql
SELECT cron.schedule('reindex-orders-status', '0 3 * * 0',
    $$REINDEX INDEX CONCURRENTLY idx_orders_tenant_status_created$$);
```

---

## EXPLAIN ANALYZE Deep Dive

Each node shows `(cost=startup..total rows=estimated)` then `(actual time=first..total rows=actual loops=N)`.
- Large `rows` discrepancy = stale stats → run `ANALYZE`
- `Buffers: shared hit=N read=M`: N from RAM cache, M from disk

### Scan Node Types

| Node | Meaning |
|---|---|
| `Seq Scan` | Full table scan — no usable index or planner judged it cheaper |
| `Index Scan` | Index used; heap fetch for non-`INCLUDE` columns |
| `Index Only Scan` | All data from covering index — `Heap Fetches: 0` is ideal |
| `Bitmap Index Scan` | Builds page bitmap; used for OR conditions or multi-index `BitmapAnd` |
| `Bitmap Heap Scan` | Fetches heap pages identified by bitmap |

### Go Helper: Run EXPLAIN ANALYZE

```go
// internal/repository/explain.go
package repository

import (
    "context"
    "fmt"
    "strings"

    "github.com/jmoiron/sqlx"
)

// ExplainQuery runs EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) and returns the full plan.
// Use only in development/staging — never in production hot paths.
func ExplainQuery(ctx context.Context, db *sqlx.DB, query string, args ...any) (string, error) {
    rows, err := db.QueryContext(ctx, "EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) "+query, args...)
    if err != nil {
        return "", fmt.Errorf("explain: %w", err)
    }
    defer rows.Close()
    var lines []string
    for rows.Next() {
        var line string
        if err := rows.Scan(&line); err != nil {
            return "", fmt.Errorf("explain scan: %w", err)
        }
        lines = append(lines, line)
    }
    return strings.Join(lines, "\n"), rows.Err()
}
```

### 5 Warning Signs

| Signal in EXPLAIN ANALYZE | Meaning | Fix |
|---|---|---|
| `Seq Scan` on large table | No usable index | Add index; check `random_page_cost` |
| `actual rows` ≫ `rows` estimate | Stale statistics | `ANALYZE tablename;` |
| `Buffers: shared hit=0 read=N` | Cold cache / high I/O | Warm cache; increase `shared_buffers` |
| `Sort Method: external merge` | `work_mem` too low | `SET work_mem = '64MB';` or add sort-order index |
| `Nested Loop` with large outer | Planner underestimates join cardinality | Check statistics; test `SET enable_nestloop = off` |

### Annotated Plans

**Plan 1 — Index Only Scan (ideal):**
```
Index Only Scan using idx_users_email_covering on users
  (cost=0.43..4.45 rows=1 width=40) (actual time=0.028..0.030 rows=1 loops=1)
  Index Cond: (email = 'user@example.com')
  Heap Fetches: 0
  Buffers: shared hit=2
Execution Time: 0.1 ms
```

**Plan 2 — Seq Scan (missing index):**
```
Seq Scan on orders
  (cost=0.00..45820.00 rows=1 width=120) (actual time=312.4..8204.1 rows=1 loops=1)
  Filter: (order_ref = 'ORD-2024-99999')
  Rows Removed by Filter: 2199999
  Buffers: shared hit=12450 read=8820
Execution Time: 8205.2 ms
```

Fix: `CREATE INDEX idx_orders_ref ON orders (order_ref);`


> For the complete e-commerce product search example with DDL, Go repository, and BitmapAnd EXPLAIN output, see [index-strategy-types-and-variants.md](index-strategy-types-and-variants.md) (bottom of file).
