# Extensions Toolkit — pg_partman and pg_trgm

## pg_partman — Partition Management

### Setup

```sql
CREATE SCHEMA IF NOT EXISTS partman;
CREATE EXTENSION IF NOT EXISTS pg_partman SCHEMA partman;
GRANT ALL ON SCHEMA partman TO myapp_user;
GRANT ALL ON ALL TABLES IN SCHEMA partman TO myapp_user;
```

### create_parent() Parameters

```
partman.create_parent(
    p_parent_table   TEXT,     -- fully qualified: 'public.events'
    p_control        TEXT,     -- partition key column: 'created_at'
    p_type           TEXT,     -- 'native' (PG 10+) or 'partman' (trigger-based, legacy)
    p_interval       TEXT,     -- 'monthly', 'weekly', 'daily', or integer for serial
    p_premake        INT,      -- how many future partitions to pre-create (default 4)
    p_start_partition TEXT     -- optional: first partition start date
)
```

### Time-Based Partitioning

```sql
CREATE TABLE events (
    id         BIGINT GENERATED ALWAYS AS IDENTITY,
    tenant_id  UUID          NOT NULL,
    event_type TEXT          NOT NULL,
    payload    JSONB,
    created_at TIMESTAMPTZ   NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

SELECT partman.create_parent(
    p_parent_table => 'public.events',
    p_control      => 'created_at',
    p_type         => 'native',
    p_interval     => 'monthly',
    p_premake      => 3
);
```

### Retention Policies

```sql
UPDATE partman.part_config
SET
    retention             = '12 months',
    retention_keep_table  = false,
    retention_keep_index  = false,
    infinite_time_partitions = true
WHERE parent_table = 'public.events';
```

### Maintenance

```sql
CALL partman.run_maintenance_proc();  -- run manually or via pg_cron

SELECT parent_table, control, partition_interval, premake, retention
FROM partman.part_config;
```

### Complete Migration Example

```sql
-- 000010_events_partitioned.up.sql
CREATE EXTENSION IF NOT EXISTS pg_partman SCHEMA partman;

CREATE TABLE events (
    id         BIGINT GENERATED ALWAYS AS IDENTITY,
    tenant_id  UUID         NOT NULL,
    event_type TEXT         NOT NULL,
    payload    JSONB,
    created_at TIMESTAMPTZ  NOT NULL DEFAULT now(),
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

CREATE INDEX ON events (tenant_id, created_at);
CREATE INDEX ON events USING gin (payload jsonb_path_ops);

SELECT partman.create_parent('public.events', 'created_at', 'native', 'monthly', 3);

UPDATE partman.part_config SET retention = '12 months', retention_keep_table = false
WHERE parent_table = 'public.events';

SELECT cron.schedule('partman-maintenance', '*/30 * * * *', 'CALL partman.run_maintenance_proc()');
```

Down migration: `SELECT cron.unschedule('partman-maintenance')`, `SELECT partman.undo_partition('public.events', p_keep_table => false)`, `DROP TABLE IF EXISTS events`.

---

## pg_trgm — Trigram Matching

### Setup

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;
```

### Key Functions

| Function | Returns | Use Case |
|---|---|---|
| `similarity(a, b)` | float (0–1) | Overall string similarity |
| `word_similarity(a, b)` | float (0–1) | Best matching substring |
| `strict_word_similarity(a, b)` | float (0–1) | Whole-word matching |
| `show_trgm(text)` | text[] | Debug: show trigrams |

```sql
SET pg_trgm.similarity_threshold = 0.3;  -- default 0.3; lower = more results

-- GIN index: faster for LIKE/ILIKE
CREATE INDEX idx_products_name_trgm ON products USING gin (name gin_trgm_ops);

-- GiST index: better for ORDER BY similarity()
CREATE INDEX idx_users_username_trgm ON users USING gist (username gist_trgm_ops);

SELECT username, similarity(username, 'johndoe') AS score
FROM users WHERE username % 'johndoe'
ORDER BY score DESC LIMIT 10;
```

**GIN vs GiST:** GIN is faster for `LIKE`/`ILIKE`; GiST is smaller and better for `ORDER BY similarity()`.

### Fuzzy Search in Go

```go
// minSimilarity: 0.1 (broad) to 0.9 (strict). 0.3 is a good default.
func SearchProducts(ctx context.Context, db *sqlx.DB, query string, minSimilarity float64, limit int) ([]Product, error) {
    const q = `SELECT id, name, similarity(name, $1) AS similarity FROM products
        WHERE name % $1 OR name ILIKE '%' || $1 || '%' ORDER BY similarity DESC, name LIMIT $3`
    if _, err := db.ExecContext(ctx, `SET pg_trgm.similarity_threshold = $1`, minSimilarity); err != nil {
        return nil, fmt.Errorf("set trgm threshold: %w", err)
    }
    var products []Product
    if err := db.SelectContext(ctx, &products, q, query, minSimilarity, limit); err != nil {
        return nil, fmt.Errorf("search products: %w", err)
    }
    return products, nil
}
```
