# Index Strategy — Types, Partial, Expression, and Variants

> All Go examples use `sqlx` with raw SQL, `log/slog`, and `context.Context`. No GORM.

## Index Types Overview

| Type | Default? | Best For | Operators Supported |
|---|---|---|---|
| **B-tree** | Yes | Equality, range, ORDER BY | `=`, `<`, `>`, `<=`, `>=`, `BETWEEN`, `LIKE 'foo%'` |
| **GIN** | No | Full-text, JSONB, arrays | `@@`, `@>`, `<@`, `?`, `?|`, `?&`, `&&` |
| **GiST** | No | Geometry, ranges, nearest-neighbor | `&&`, `@>`, `<->`, exclusion |
| **BRIN** | No | Monotonic, append-only columns | `=`, `<`, `>`, `<=`, `>=` |
| **SP-GiST** | No | IP prefixes, partitioned search spaces | `=`, `<<`, `>>=`, `<<=` |
| **Hash** | No | Equality only, no range | `=` only |

**Decision rule:** Default to B-tree. Switch only when the query operator is not in B-tree's supported set.

---

## B-tree Indexes

### Equality, Range, and Composite

```sql
CREATE INDEX idx_users_email ON users (email);
CREATE INDEX idx_products_price ON products (price);

-- Composite: equality columns first, then range, then ORDER BY
CREATE INDEX idx_orders_tenant_status_created
    ON orders (tenant_id, status, created_at DESC);

SELECT * FROM orders WHERE tenant_id=$1 AND status='pending';         -- uses index
SELECT * FROM orders WHERE tenant_id=$1 AND status='pending'
    ORDER BY created_at DESC;                                          -- uses index
SELECT * FROM orders WHERE status = 'pending';                        -- skips index
```

### Covering Indexes with INCLUDE

`INCLUDE` adds non-key columns to leaf pages — enables Index Only Scan (no heap fetch).

```sql
CREATE INDEX idx_users_email_covering ON users (email) INCLUDE (id, role);
-- Index Only Scan — Heap Fetches: 0
SELECT id, role FROM users WHERE email = $1 AND deleted_at IS NULL;
```

### Sort Direction

```sql
CREATE INDEX idx_posts_created_desc ON posts (created_at DESC NULLS LAST);
CREATE INDEX idx_products_cat_price ON products (category_id ASC, price DESC NULLS LAST);
```

---

## GIN Indexes

### Full-text Search with tsvector

```sql
ALTER TABLE articles
    ADD COLUMN search_vector tsvector GENERATED ALWAYS AS (
        to_tsvector('english', coalesce(title,'') || ' ' || coalesce(body,''))
    ) STORED;

CREATE INDEX idx_articles_search ON articles USING gin (search_vector);

SELECT id, title FROM articles
WHERE search_vector @@ plainto_tsquery('english', $1) AND deleted_at IS NULL
ORDER BY ts_rank(search_vector, plainto_tsquery('english', $1)) DESC LIMIT $2;
```

### JSONB Containment and Key Existence

```sql
-- Default operator class: @>, ?, ?|, ?& (key existence + containment)
CREATE INDEX idx_products_metadata ON products USING gin (metadata);

-- jsonb_path_ops: smaller — supports @> containment only
CREATE INDEX idx_products_metadata_path ON products USING gin (metadata jsonb_path_ops);

SELECT id, name FROM products WHERE metadata @> '{"tags":["organic"]}';
SELECT id, name FROM products WHERE metadata ? 'discount_pct';
```

Use `jsonb_path_ops` when all queries use `@>`. Use default when you also need `?` / `?|` / `?&`.

### Array Operators

```sql
CREATE INDEX idx_products_tags ON products USING gin (tags);
SELECT id FROM products WHERE tags @> ARRAY['vegan','gluten-free'];  -- ALL tags
SELECT id FROM products WHERE tags && ARRAY['sale','new'];            -- ANY tag
```

### pg_trgm for LIKE '%pattern%'

B-tree cannot handle leading-wildcard LIKE.

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX idx_products_name_trgm ON products USING gin (name gin_trgm_ops);

SELECT id, name FROM products WHERE name ILIKE '%organic hemp%';
SELECT id, name, similarity(name, $1) AS score FROM products
WHERE name % $1 ORDER BY score DESC LIMIT 10;
```


> For GiST, BRIN, SP-GiST, Partial, and Expression index types, see [index-strategy-maintenance-and-explain.md](index-strategy-maintenance-and-explain.md).
