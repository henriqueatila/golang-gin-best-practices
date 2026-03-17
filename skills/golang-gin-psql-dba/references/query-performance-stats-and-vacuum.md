# Query Performance — pg_stat_statements, Autovacuum, and Bloat

## pg_stat_statements Setup

```ini
# postgresql.conf (requires server restart)
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.track = all
pg_stat_statements.max = 10000
```

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

### Key Queries

```sql
-- Top slow queries by mean time
SELECT left(query, 120) AS query_preview, calls,
    round(mean_exec_time::numeric, 2) AS mean_ms,
    round(total_exec_time::numeric, 2) AS total_ms, rows
FROM pg_stat_statements
WHERE calls > 10
ORDER BY mean_exec_time DESC LIMIT 20;

-- Top I/O queries (disk reads)
SELECT left(query, 120) AS query_preview, calls, shared_blks_read,
    round(shared_blks_hit::numeric / NULLIF(shared_blks_hit + shared_blks_read, 0) * 100, 2) AS cache_hit_pct
FROM pg_stat_statements
WHERE shared_blks_read > 0
ORDER BY shared_blks_read DESC LIMIT 20;
```

### Go Helper: Log Slow Queries

```go
func LogSlowQueries(ctx context.Context, db *sqlx.DB, thresholdMs float64, logger *slog.Logger) error {
    const q = `SELECT left(query, 120) AS query_preview, calls,
        round(mean_exec_time::numeric, 2) AS mean_ms, round(total_exec_time::numeric, 2) AS total_ms
        FROM pg_stat_statements WHERE calls > 5 AND mean_exec_time > $1 ORDER BY mean_exec_time DESC LIMIT 10`
    var rows []struct{ QueryPreview string `db:"query_preview"`; Calls int64 `db:"calls"`; MeanMs float64 `db:"mean_ms"` }
    if err := db.SelectContext(ctx, &rows, q, thresholdMs); err != nil {
        return fmt.Errorf("pg_stat_statements query: %w", err)
    }
    for _, r := range rows {
        logger.WarnContext(ctx, "slow query detected", "mean_ms", r.MeanMs, "calls", r.Calls, "query", r.QueryPreview)
    }
    return nil
}
```

---

## Autovacuum Tuning

PostgreSQL uses MVCC — every UPDATE/DELETE leaves dead tuples. Autovacuum reclaims them and updates planner statistics.

A table with 10M rows needs 2,000,050 dead tuples before default autovacuum fires. Too infrequent for hot tables.

### Per-Table Tuning

```sql
-- orders: 1% dead tuple threshold instead of default 20%
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor  = 0.01,
    autovacuum_analyze_scale_factor = 0.005,
    autovacuum_vacuum_cost_delay    = 2
);

-- job_queue: vacuum after every 100 dead rows
ALTER TABLE job_queue SET (
    autovacuum_vacuum_scale_factor  = 0.0,
    autovacuum_vacuum_threshold     = 100,
    autovacuum_analyze_scale_factor = 0.0,
    autovacuum_analyze_threshold    = 100
);
```

### Monitoring

```sql
SELECT relname AS table_name, n_dead_tup, n_live_tup,
    round(n_dead_tup::numeric / NULLIF(n_live_tup + n_dead_tup, 0) * 100, 2) AS dead_pct,
    last_autovacuum, last_autoanalyze
FROM pg_stat_user_tables ORDER BY n_dead_tup DESC LIMIT 20;

-- Wraparound risk (alert when xid_age > 150,000,000)
SELECT relname, age(relfrozenxid) AS xid_age
FROM pg_class WHERE relkind = 'r'
ORDER BY xid_age DESC LIMIT 10;
```

---

## Table Bloat Detection

### Detection

```sql
SELECT relname AS table_name,
    pg_size_pretty(pg_total_relation_size(relid)) AS total_size,
    n_dead_tup, n_live_tup,
    round(n_dead_tup::numeric / NULLIF(n_live_tup, 0) * 100, 2) AS dead_ratio_pct
FROM pg_stat_user_tables
WHERE n_live_tup > 1000
ORDER BY n_dead_tup DESC LIMIT 20;
```

Accurate measurement with pgstattuple (scans the full table):

```sql
CREATE EXTENSION IF NOT EXISTS pgstattuple;
SELECT table_len, dead_tuple_count, dead_tuple_percent, free_percent
FROM pgstattuple('orders');
```

### Fix Options

| Method | Downside | When to Use |
|---|---|---|
| `VACUUM tablename` | Doesn't shrink file on disk | Regular maintenance |
| `VACUUM FULL tablename` | Full `AccessExclusiveLock` — blocks reads/writes | Small tables, maintenance windows |
| `pg_repack` (extension) | Online rewrite — no long lock | Production tables > 1 GB |

## Planner Statistics

```sql
ANALYZE orders;  -- update statistics for one table

-- Per-column override for skewed distributions
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 500;
ALTER TABLE events ALTER COLUMN user_id SET STATISTICS 1000;
ANALYZE orders; ANALYZE events;

-- Extended stats for correlated columns
CREATE STATISTICS orders_status_region ON status, region FROM orders;
ANALYZE orders;
```
