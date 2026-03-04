# Index Strategy Reference

This reference covers every PostgreSQL index type relevant to Go/Gin APIs: when to use B-tree, GIN, GiST, BRIN, SP-GiST, and Hash; composite, covering, partial, and expression variants; index maintenance and bloat detection; and a deep dive into reading EXPLAIN ANALYZE output — including a Go helper function that runs and logs query plans. Ends with a complete e-commerce product search example combining multiple index types. Use when choosing indexes, diagnosing slow queries, or reviewing schema DDL.

> All Go examples use `sqlx` with raw SQL, `log/slog`, and `context.Context`. No GORM.

## Table of Contents

1. [Index Types Overview](#index-types-overview)
2. [B-tree Indexes](#b-tree-indexes)
3. [GIN Indexes](#gin-indexes)
4. [GiST Indexes](#gist-indexes)
5. [BRIN Indexes](#brin-indexes)
6. [SP-GiST Indexes](#sp-gist-indexes)
7. [Partial Indexes](#partial-indexes)
8. [Expression Indexes](#expression-indexes)
9. [Index Maintenance](#index-maintenance)
10. [EXPLAIN ANALYZE Deep Dive](#explain-analyze-deep-dive)
11. [Complete Example — E-commerce Product Search](#complete-example--e-commerce-product-search)

---

## Index Types Overview

| Type | Default? | Best For | Operators Supported | Rough Size |
|---|---|---|---|---|
| **B-tree** | Yes | Equality, range, ORDER BY | `=`, `<`, `>`, `<=`, `>=`, `BETWEEN`, `LIKE 'foo%'` | Moderate |
| **GIN** | No | Full-text, JSONB, arrays | `@@`, `@>`, `<@`, `?`, `?|`, `?&`, `&&` | Larger than B-tree |
| **GiST** | No | Geometry, ranges, nearest-neighbor | `&&`, `@>`, `<->`, `<<`, `>>`, exclusion | Moderate |
| **BRIN** | No | Monotonic, append-only columns | `=`, `<`, `>`, `<=`, `>=` | Tiny (pages summary) |
| **SP-GiST** | No | IP prefixes, partitioned search spaces | `=`, `<<`, `>>=`, `<<=` | Moderate |
| **Hash** | No | Equality only, no range | `=` only | Similar to B-tree |

**Decision rule:** Default to B-tree. Switch only when the query operator is not in B-tree's supported set or cardinality/physical layout makes another type a clear win.

---

## B-tree Indexes

The default index type. Covers the vast majority of query patterns in a typical Gin API.

### Equality and Range

```sql
-- Single-column equality — standard lookup
CREATE INDEX idx_users_email ON users (email);

-- Range scan — price queries
CREATE INDEX idx_products_price ON products (price);

-- BETWEEN uses the same index
SELECT * FROM products WHERE price BETWEEN 10.00 AND 50.00;
```

### Composite Indexes and the Leftmost Prefix Rule

PostgreSQL uses a composite index only when the query references a **prefix** of the indexed columns. Put equality columns first, then range columns, then ORDER BY columns.

```sql
CREATE INDEX idx_orders_tenant_status_created
    ON orders (tenant_id, status, created_at DESC);

SELECT * FROM orders WHERE tenant_id=$1 AND status='pending';            -- uses index
SELECT * FROM orders WHERE tenant_id=$1 AND status='pending'
    ORDER BY created_at DESC;                                             -- uses index
SELECT * FROM orders WHERE status = 'pending';                           -- skips index (no tenant_id)
```

### Covering Indexes with INCLUDE

`INCLUDE` adds non-key columns to leaf pages so the planner can do an Index Only Scan — no heap fetch. Use for high-frequency lookups; avoid including wide columns.

```sql
CREATE INDEX idx_users_email_covering ON users (email) INCLUDE (id, role);
-- Index Only Scan — Heap Fetches: 0
SELECT id, role FROM users WHERE email = $1 AND deleted_at IS NULL;
```

### Sort Direction

Specify `ASC/DESC NULLS FIRST/LAST` explicitly — mismatches with ORDER BY force an extra Sort node even when the index is used.

```sql
CREATE INDEX idx_posts_created_desc ON posts (created_at DESC NULLS LAST);
CREATE INDEX idx_products_cat_price ON products (category_id ASC, price DESC NULLS LAST);
```

---

## GIN Indexes

Generalized Inverted Indexes. Store a posting list per element value. Required for full-text search, JSONB operators, and array overlap.

### Full-text Search with tsvector

```sql
-- Generated stored column (PG 12+) + GIN index
ALTER TABLE articles
    ADD COLUMN search_vector tsvector GENERATED ALWAYS AS (
        to_tsvector('english', coalesce(title,'') || ' ' || coalesce(body,''))
    ) STORED;

CREATE INDEX idx_articles_search ON articles USING gin (search_vector);

-- Search with ranking
SELECT id, title
FROM articles
WHERE search_vector @@ plainto_tsquery('english', $1)
ORDER BY ts_rank(search_vector, plainto_tsquery('english', $1)) DESC
LIMIT 20;
```

```go
// sqlx — same expression in Go query
const q = `SELECT id, title FROM articles
    WHERE search_vector @@ plainto_tsquery('english', $1) AND deleted_at IS NULL
    ORDER BY ts_rank(search_vector, plainto_tsquery('english', $1)) DESC LIMIT $2`
if err := db.SelectContext(ctx, &rows, q, query, limit); err != nil {
    return nil, fmt.Errorf("article search: %w", err)
}
```

### JSONB Containment and Key Existence

```sql
-- Default operator class: @>, ?, ?|, ?& (key existence + containment)
CREATE INDEX idx_products_metadata ON products USING gin (metadata);

-- jsonb_path_ops: smaller — supports @> containment only
CREATE INDEX idx_products_metadata_path ON products USING gin (metadata jsonb_path_ops);

SELECT id, name FROM products WHERE metadata @> '{"tags":["organic"]}';  -- containment
SELECT id, name FROM products WHERE metadata ? 'discount_pct';            -- key exists
SELECT id, name FROM products WHERE metadata ?| ARRAY['sale','clearance']; -- any key
```

Use `jsonb_path_ops` when all queries use `@>` — it's smaller and faster. Use default when you also need `?` / `?|` / `?&`.

### Array Operators

```sql
CREATE INDEX idx_products_tags ON products USING gin (tags);
SELECT id FROM products WHERE tags @> ARRAY['vegan','gluten-free'];  -- ALL tags
SELECT id FROM products WHERE tags && ARRAY['sale','new'];            -- ANY tag
```

### pg_trgm for LIKE '%pattern%'

B-tree cannot handle leading-wildcard LIKE. `pg_trgm` breaks strings into trigrams for GIN indexing.

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX idx_products_name_trgm ON products USING gin (name gin_trgm_ops);

SELECT id, name FROM products WHERE name ILIKE '%organic hemp%';        -- accelerated
SELECT id, name, similarity(name, $1) AS score FROM products            -- fuzzy
WHERE name % $1 ORDER BY score DESC LIMIT 10;
```

---

## GiST Indexes

Generalized Search Tree. Extensible index type used for spatial data, range types, and exclusion constraints.

### Spatial Types (PostGIS), Range Types, Exclusion Constraints

```sql
-- Nearest-neighbor KNN — GiST <-> operator
CREATE INDEX idx_stores_location ON stores USING gist (location);
SELECT id, name, location <-> ST_MakePoint($1,$2)::geography AS dist
FROM stores ORDER BY dist LIMIT 10;

-- Range overlap — find conflicting bookings
CREATE INDEX idx_bookings_range ON bookings USING gist (booked_at);
SELECT id FROM bookings WHERE resource_id = $1 AND booked_at && $2::tstzrange;

-- Exclusion constraint — prevent double-booking (implicitly creates GiST index)
ALTER TABLE bookings
    ADD CONSTRAINT no_double_booking
    EXCLUDE USING gist (resource_id WITH =, booked_at WITH &&);
```

---

## BRIN Indexes

Block Range INdexes store min/max summaries per page range — often 1000x smaller than B-tree on the same column. Effective only when column values are **correlated with physical storage order** (monotonically inserted rows).

```sql
-- Append-only events log — created_at always increases
CREATE INDEX idx_events_created_brin ON events USING brin (created_at);

-- Tune pages_per_range (default 128) — smaller = more precise but larger index
CREATE INDEX idx_events_created_brin ON events
    USING brin (created_at) WITH (pages_per_range = 32);

-- Efficient range scan on a 100M-row table with a ~48KB index (vs ~3GB B-tree)
SELECT id, event_type FROM events
WHERE created_at >= now() - interval '7 days';
```

**When BRIN fails:** out-of-order inserts (back-dated events, bulk historical imports) cause overlapping min/max ranges — planner degrades to a full table scan. Use B-tree or time-based partitioning instead.

---

## SP-GiST Indexes

Space-Partitioned GiST. Efficient for hierarchical, non-overlapping data: IP subnets, phone number prefixes, text with shared prefixes.

```sql
-- inet subnet containment
CREATE INDEX idx_logs_ip ON access_logs USING spgist (client_ip);
SELECT * FROM access_logs WHERE client_ip << '192.168.1.0/24'::inet;

-- Text prefix match
CREATE INDEX idx_contacts_phone ON contacts USING spgist (phone_number);
SELECT id, name FROM contacts WHERE phone_number LIKE '+1415%';
```

**GiST vs SP-GiST:** GiST handles overlapping regions (geometry, ranges). SP-GiST handles partitioned, non-overlapping spaces (IP ranges, prefixes). Default to GiST; use SP-GiST only when benchmarks show a clear win.

---

## Partial Indexes

Index only rows matching a `WHERE` clause — smaller, cheaper to maintain, tighter planner scans.

```sql
-- Active-only email lookup (most common soft-delete pattern)
CREATE INDEX idx_users_email_active ON users (email)
    WHERE deleted_at IS NULL;

-- Partial indexes for low-cardinality filter columns
-- Never index boolean alone — partial index the interesting subset instead
CREATE INDEX idx_orders_pending_created ON orders (created_at DESC)
    WHERE status = 'pending';

-- Partial unique constraint — one active email, unlimited soft-deleted duplicates
CREATE UNIQUE INDEX idx_users_email_unique_active ON users (email)
    WHERE deleted_at IS NULL;
```

**Size impact:** A partial index covering 5% of rows is ~20x smaller than a full index. For soft-delete patterns, this is typically the highest-ROI index optimization.

---

## Expression Indexes

Index the result of a function or expression. The planner uses the index only when the query's WHERE clause uses the **identical** expression.

```sql
-- Case-insensitive email — query must use LOWER() to match
CREATE INDEX idx_users_email_lower ON users (LOWER(email));
SELECT id FROM users WHERE LOWER(email) = LOWER($1) AND deleted_at IS NULL;

-- Daily aggregation — query must use date_trunc('day', ...) to match
CREATE INDEX idx_orders_day ON orders (date_trunc('day', created_at));
SELECT COUNT(*), SUM(total)
FROM orders WHERE date_trunc('day', created_at) = date_trunc('day', now());

-- JSONB path as text — double-paren syntax required
CREATE INDEX idx_products_brand ON products ((metadata->>'brand'));
SELECT id, name FROM products WHERE metadata->>'brand' = $1;
```

**Expression indexes vs generated columns (PG 12+):** `GENERATED ALWAYS AS (...) STORED` columns can be indexed with a normal B-tree and are visible in the schema. Prefer generated columns for values queried frequently — expression indexes are invisible to schema introspection tools.

---

## Index Maintenance

### REINDEX CONCURRENTLY (PG 12+)

Rebuilds an index without holding a lock that blocks reads and writes.

```sql
-- Rebuild a single bloated index online
REINDEX INDEX CONCURRENTLY idx_orders_tenant_status_created;

-- Rebuild all indexes on a table online
REINDEX TABLE CONCURRENTLY orders;
```

**Note:** `CONCURRENTLY` requires more disk space (old + new index coexist during rebuild) and takes longer, but never blocks production traffic. Use on any table serving live requests.

### Detecting Bloated Indexes with pgstattuple

```sql
CREATE EXTENSION IF NOT EXISTS pgstattuple;
SELECT index_size, leaf_fragmentation, avg_leaf_density
FROM pgstatindex('idx_orders_tenant_status_created');
-- leaf_fragmentation > 30% or avg_leaf_density < 50% → REINDEX
```

### Unused Index Detection

Run after at least a week of normal load. Drop unused indexes — they cost write overhead with zero read benefit.

```sql
SELECT schemaname, tablename, indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    idx_scan AS scans
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelname NOT LIKE '%_pkey'
  AND indexrelname NOT LIKE '%_unique%'
ORDER BY pg_relation_size(indexrelid) DESC;

DROP INDEX CONCURRENTLY idx_products_legacy_column;  -- non-blocking
```

### Auto-maintenance with pg_cron

```sql
-- Weekly off-peak REINDEX (requires pg_cron extension)
SELECT cron.schedule('reindex-orders-status', '0 3 * * 0',
    $$REINDEX INDEX CONCURRENTLY idx_orders_tenant_status_created$$);
```

---

## EXPLAIN ANALYZE Deep Dive

Each node shows `(cost=startup..total rows=estimated)` followed by `(actual time=first..total rows=actual loops=N)`. Large `rows` discrepancy = stale stats → run `ANALYZE`. `Buffers: shared hit=N read=M`: N from RAM cache, M from disk.

### Scan Node Types

| Node | Meaning |
|---|---|
| `Seq Scan` | Full table scan — no usable index or planner judged it cheaper |
| `Index Scan` | Index used; heap fetch for non-`INCLUDE` columns |
| `Index Only Scan` | All data from covering index — `Heap Fetches: 0` is ideal |
| `Bitmap Index Scan` | Builds page bitmap; used for OR conditions or multi-index `BitmapAnd` |
| `Bitmap Heap Scan` | Fetches heap pages identified by bitmap — always follows Bitmap Index Scan |

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

### 5 Warning Signs — Quick Reference

| Signal in EXPLAIN ANALYZE | Meaning | Fix |
|---|---|---|
| `Seq Scan` on large table | No usable index | Add index; check `random_page_cost` |
| `actual rows` ≫ `rows` estimate | Stale statistics | `ANALYZE tablename;` |
| `Buffers: shared hit=0 read=N` | Cold cache / high I/O | Warm cache; increase `shared_buffers` |
| `Sort Method: external merge` | `work_mem` too low for in-memory sort | `SET work_mem = '64MB';` or add sort-order index |
| `Nested Loop` with large outer | Planner underestimates join cardinality | Check statistics; test `SET enable_nestloop = off` |

### Annotated Plan 1 — Index Only Scan (ideal)

```
Index Only Scan using idx_users_email_covering on users
  (cost=0.43..4.45 rows=1 width=40)
  (actual time=0.028..0.030 rows=1 loops=1)
  Index Cond: (email = 'user@example.com')
  Heap Fetches: 0
  Buffers: shared hit=2
Execution Time: 0.1 ms
```

`Heap Fetches: 0` — all data from the covering index, no table touch. 2 buffer hits (RAM). Ideal outcome for a high-frequency lookup.

### Annotated Plan 2 — Seq Scan (missing index)

```
Seq Scan on orders
  (cost=0.00..45820.00 rows=1 width=120)
  (actual time=312.4..8204.1 rows=1 loops=1)
  Filter: (order_ref = 'ORD-2024-99999')
  Rows Removed by Filter: 2199999
  Buffers: shared hit=12450 read=8820
Execution Time: 8205.2 ms
```

Scanned 2.2M rows, returned 1. `read=8820` = 8820 pages read from disk. Fix: `CREATE INDEX idx_orders_ref ON orders (order_ref);`.

---

## Complete Example — E-commerce Product Search

A product catalog with price-range filtering (B-tree), JSONB tag/attribute search (GIN), and active-only partial index.

### DDL

```sql
-- Products table
CREATE TABLE products (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        TEXT NOT NULL,
    description TEXT,
    price       NUMERIC(12,2) NOT NULL,
    category_id UUID NOT NULL REFERENCES categories (id),
    metadata    JSONB NOT NULL DEFAULT '{}',
    active      BOOLEAN NOT NULL DEFAULT true,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- 1. B-tree on (category_id, price) for filtered range scans
CREATE INDEX idx_products_cat_price
    ON products (category_id, price);

-- 2. GIN on metadata with jsonb_path_ops — containment queries only
--    (smaller than default operator class)
CREATE INDEX idx_products_metadata_path
    ON products USING gin (metadata jsonb_path_ops);

-- 3. Partial index — active products only, ordered for pagination
CREATE INDEX idx_products_active_created
    ON products (created_at DESC)
    WHERE active = true;

-- 4. Trigram index for name search with leading wildcards
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX idx_products_name_trgm
    ON products USING gin (name gin_trgm_ops);

-- 5. Expression index for case-insensitive slug lookup
CREATE INDEX idx_products_name_lower
    ON products (LOWER(name))
    WHERE active = true;
```

### Sample Queries and Index Usage

```sql
-- Combined filter: B-tree (category+price) + GIN (JSONB tags) → BitmapAnd
SELECT id, name, price
FROM products
WHERE category_id = $1
  AND price BETWEEN $2 AND $3
  AND metadata @> $3::jsonb   -- e.g. '{"tags":["organic","vegan"]}'
  AND active = true
ORDER BY price ASC
LIMIT 20;

-- Fuzzy name search (pg_trgm) — % is the similarity threshold operator
SELECT id, name, similarity(name, $1) AS score
FROM products
WHERE name % $1 AND active = true
ORDER BY score DESC LIMIT 10;
```

### Go Repository

```go
// internal/repository/product_repository.go
package repository

import (
    "context"
    "encoding/json"
    "fmt"

    "github.com/jmoiron/sqlx"
)

type ProductFilter struct {
    CategoryID string
    MinPrice   float64
    MaxPrice   float64
    Tags       []string // matched with JSONB @> containment
    Limit      int
    Offset     int
}

type productRow struct {
    ID    string  `db:"id"`
    Name  string  `db:"name"`
    Price float64 `db:"price"`
}

type ProductRepository struct{ db *sqlx.DB }

func (r *ProductRepository) Search(ctx context.Context, f ProductFilter) ([]productRow, error) {
    if f.Limit <= 0 || f.Limit > 100 {
        f.Limit = 20
    }
    // Build JSONB containment argument: {"tags":["organic","vegan"]}
    type tagsDoc struct {
        Tags []string `json:"tags"`
    }
    tagsJSON, _ := json.Marshal(tagsDoc{Tags: f.Tags})

    const q = `
        SELECT id, name, price
        FROM products
        WHERE category_id = $1
          AND price BETWEEN $2 AND $3
          AND metadata @> $4::jsonb
          AND active = true
        ORDER BY price ASC
        LIMIT $5 OFFSET $6
    `
    var rows []productRow
    if err := r.db.SelectContext(ctx, &rows, q,
        f.CategoryID, f.MinPrice, f.MaxPrice, tagsJSON, f.Limit, f.Offset,
    ); err != nil {
        return nil, fmt.Errorf("product search: %w", err)
    }
    return rows, nil
}
```

### EXPLAIN ANALYZE Output for Query C

```
Bitmap Heap Scan on products
  (cost=42.30..198.40 rows=12 width=64)
  (actual time=1.8..3.2 rows=9 loops=1)
  Recheck Cond: ((category_id = '...') AND (metadata @> '{"tags":["organic"]}'))
  Filter: ((price < 50.00) AND active)
  Rows Removed by Filter: 3
  Buffers: shared hit=28 read=4
  ->  BitmapAnd
      ->  Bitmap Index Scan on idx_products_cat_price
            Index Cond: (category_id = '...' AND price < 50.00)
            Buffers: shared hit=4
      ->  Bitmap Index Scan on idx_products_metadata_path
            Index Cond: (metadata @> '{"tags":["organic"]}')
            Buffers: shared hit=8 read=4
Planning Time: 1.1 ms
Execution Time: 3.4 ms
```

Analysis: PostgreSQL combined two Bitmap Index Scans with `BitmapAnd` — one on the B-tree (category + price), one on the GIN (JSONB tags). Final heap scan checked `active = true` as a filter on the 12 surviving rows. 28 buffer hits, 4 disk reads. Sub-5ms for a multi-condition search on a large catalog. Healthy.
