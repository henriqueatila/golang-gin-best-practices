# Query Performance Reference

This reference covers PostgreSQL query performance for Go/Gin APIs: enabling and querying `pg_stat_statements`, tuning autovacuum for hot tables, detecting and repairing table bloat, controlling planner statistics, sizing `work_mem` for heavy queries, connection pool formulas, read-replica routing, and a complete Go monitoring helper that surfaces all of these metrics through a Gin health endpoint. All Go examples use `sqlx` with raw SQL, `log/slog`, and `context.Context`.

## Table of Contents

1. [pg_stat_statements Setup](#pg_stat_statements-setup)
2. [Autovacuum Tuning](#autovacuum-tuning)
3. [Table Bloat Detection](#table-bloat-detection)
4. [Planner Statistics](#planner-statistics)
5. [work_mem and Sorting](#work_mem-and-sorting)
6. [Connection Pool Sizing](#connection-pool-sizing)
7. [Read Replicas](#read-replicas)
8. [Monitoring Go Helper](#monitoring-go-helper)
9. [Quick Tuning Checklist](#quick-tuning-checklist)

---

## pg_stat_statements Setup

`pg_stat_statements` is the single most useful extension for finding slow queries. It tracks execution statistics for every normalized query type seen by the server.

### Enable

In `postgresql.conf`:

```ini
# postgresql.conf
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.track = all          # top (default) | all | none
pg_stat_statements.max = 10000          # number of query fingerprints to track
```

Requires a server restart. Then in each database you want to monitor:

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

### Key Columns

| Column | Meaning |
|---|---|
| `calls` | How many times this query type was executed |
| `mean_exec_time` | Average wall-clock time in milliseconds |
| `total_exec_time` | Total time spent — product of calls × mean |
| `rows` | Average rows returned or affected |
| `shared_blks_hit` | Buffer cache hits |
| `shared_blks_read` | Blocks read from disk (expensive) |
| `stddev_exec_time` | High stddev → inconsistent performance |

### Top Slow Queries (by mean time)

```sql
SELECT
    left(query, 120)            AS query_preview,
    calls,
    round(mean_exec_time::numeric, 2) AS mean_ms,
    round(total_exec_time::numeric, 2) AS total_ms,
    rows
FROM pg_stat_statements
WHERE calls > 10
ORDER BY mean_exec_time DESC
LIMIT 20;
```

### Top I/O Queries (by disk reads)

```sql
SELECT
    left(query, 120)                              AS query_preview,
    calls,
    shared_blks_read,
    shared_blks_hit,
    round(
        shared_blks_hit::numeric /
        NULLIF(shared_blks_hit + shared_blks_read, 0) * 100,
    2) AS cache_hit_pct
FROM pg_stat_statements
WHERE shared_blks_read > 0
ORDER BY shared_blks_read DESC
LIMIT 20;
```

Cache hit below 95% for a query that runs frequently usually means it needs a better index or `shared_buffers` is too small.

### Go Helper: Log Slow Queries at Startup

```go
// internal/db/pg_stat_statements.go
package db

import (
    "context"
    "fmt"
    "log/slog"

    "github.com/jmoiron/sqlx"
)

type SlowQuery struct {
    QueryPreview string  `db:"query_preview"`
    Calls        int64   `db:"calls"`
    MeanMs       float64 `db:"mean_ms"`
    TotalMs      float64 `db:"total_ms"`
}

// LogSlowQueries queries pg_stat_statements and logs any query with
// mean execution time above thresholdMs.
func LogSlowQueries(ctx context.Context, db *sqlx.DB, thresholdMs float64, logger *slog.Logger) error {
    const q = `
        SELECT
            left(query, 120)                          AS query_preview,
            calls,
            round(mean_exec_time::numeric, 2)         AS mean_ms,
            round(total_exec_time::numeric, 2)        AS total_ms
        FROM pg_stat_statements
        WHERE calls > 5
          AND mean_exec_time > $1
        ORDER BY mean_exec_time DESC
        LIMIT 10`

    var rows []SlowQuery
    if err := db.SelectContext(ctx, &rows, q, thresholdMs); err != nil {
        return fmt.Errorf("pg_stat_statements query: %w", err)
    }

    for _, r := range rows {
        logger.WarnContext(ctx, "slow query detected",
            "mean_ms", r.MeanMs,
            "calls", r.Calls,
            "total_ms", r.TotalMs,
            "query", r.QueryPreview,
        )
    }
    return nil
}
```

Call this at startup or from a scheduled background goroutine. Reset statistics after a deploy with `SELECT pg_stat_statements_reset();`.

---

## Autovacuum Tuning

### How Autovacuum Works

PostgreSQL uses MVCC — every UPDATE and DELETE leaves a dead tuple. Autovacuum reclaims those dead tuples so the table doesn't grow unboundedly and to update the visibility map (required for index-only scans). It also runs ANALYZE to keep planner statistics fresh.

**Default trigger thresholds (per table):**

| Setting | Default | Meaning |
|---|---|---|
| `autovacuum_vacuum_threshold` | 50 | Minimum dead tuples before a vacuum is triggered |
| `autovacuum_vacuum_scale_factor` | 0.2 | Plus 20% of `reltuples` |
| `autovacuum_analyze_threshold` | 50 | Minimum changed rows before ANALYZE |
| `autovacuum_analyze_scale_factor` | 0.1 | Plus 10% of `reltuples` |

A table with 10 million rows needs 2,000,050 dead tuples before default autovacuum fires. That is far too infrequent for any hot table.

### Per-Table Tuning for Hot Tables

```sql
-- orders gets 1% dead tuple threshold instead of 20%
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor   = 0.01,
    autovacuum_analyze_scale_factor  = 0.005,
    autovacuum_vacuum_cost_delay     = 2       -- ms, lower = more aggressive
);

-- A tiny lookup table updated constantly (e.g., a counters table)
ALTER TABLE job_queue SET (
    autovacuum_vacuum_scale_factor  = 0.0,
    autovacuum_vacuum_threshold     = 100,     -- vacuum after every 100 dead rows
    autovacuum_analyze_scale_factor = 0.0,
    autovacuum_analyze_threshold    = 100
);
```

### Monitoring Autovacuum Activity

```sql
SELECT
    schemaname,
    relname                                       AS table_name,
    n_dead_tup,
    n_live_tup,
    round(n_dead_tup::numeric / NULLIF(n_live_tup + n_dead_tup, 0) * 100, 2)
                                                  AS dead_pct,
    last_autovacuum,
    last_autoanalyze,
    autovacuum_count,
    autoanalyze_count
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 20;
```

### Warning Signs

| Sign | Risk | Action |
|---|---|---|
| `dead_pct` consistently > 10% | Table bloat, slower scans | Lower `vacuum_scale_factor` |
| `last_autovacuum` is NULL or > 1 day old on hot table | Bloat accumulating | Force `VACUUM tablename;` manually, tune thresholds |
| `age(relfrozenxid)` approaching `autovacuum_freeze_max_age` (200M) | Transaction ID wraparound — database will shut down | Run `VACUUM FREEZE tablename;` immediately |
| `autovacuum_count` not incrementing | Autovacuum blocked by a long-running transaction | Find and terminate the blocker |

Check wraparound risk:

```sql
SELECT relname, age(relfrozenxid) AS xid_age
FROM pg_class
WHERE relkind = 'r'
ORDER BY xid_age DESC
LIMIT 10;
```

Alert when `xid_age` exceeds 150,000,000.

---

## Table Bloat Detection

### What Causes Bloat

Every UPDATE writes a new tuple version and leaves the old one as dead. VACUUM marks those dead tuples as reusable but does not return space to the OS until the page is fully empty. If dead tuples accumulate faster than autovacuum reclaims them — or if autovacuum is blocked — the table grows and reads slow down even though `n_live_tup` stays constant.

### Detection via pg_stat_user_tables

```sql
SELECT
    schemaname,
    relname                                           AS table_name,
    pg_size_pretty(pg_total_relation_size(relid))     AS total_size,
    pg_size_pretty(pg_relation_size(relid))           AS table_size,
    n_dead_tup,
    n_live_tup,
    round(n_dead_tup::numeric / NULLIF(n_live_tup, 0) * 100, 2) AS dead_ratio_pct
FROM pg_stat_user_tables
WHERE n_live_tup > 1000
ORDER BY n_dead_tup DESC
LIMIT 20;
```

### Accurate Measurement with pgstattuple

For a precise bloat estimate (scans the table — avoid on very large tables during peak traffic):

```sql
CREATE EXTENSION IF NOT EXISTS pgstattuple;

SELECT
    table_len,
    tuple_count,
    dead_tuple_count,
    dead_tuple_percent,
    free_space,
    free_percent
FROM pgstattuple('orders');
```

`dead_tuple_percent` above 10–20% on a hot table warrants intervention.

### Fix Options

| Method | Downside | When to Use |
|---|---|---|
| `VACUUM tablename` | Does not shrink file on disk | Regular maintenance — keeps autovacuum healthy |
| `VACUUM FULL tablename` | Full `AccessExclusiveLock` — blocks all reads and writes | Small tables, maintenance windows only |
| `CLUSTER tablename USING index_name` | `AccessExclusiveLock`, rewrites in index order | When you also want physical sort for range scans |
| `pg_repack` (extension) | Online rewrite — no long lock | Production tables that can't tolerate downtime |

`pg_repack` is the correct choice for tables > 1 GB that receive continuous traffic:

```sql
-- Run from psql or a migration step (pg_repack must be installed)
-- pg_repack --no-order --table orders mydb
```

---

## Planner Statistics

The query planner uses column statistics to estimate row counts. Wrong estimates lead to bad plan choices (nested loop over 10M rows, sequential scans that should be index scans).

### ANALYZE

```sql
-- Update statistics for a specific table immediately
ANALYZE orders;

-- Full database — runs fast on small tables
ANALYZE;
```

Autovacuum runs ANALYZE automatically, but run it manually after a large bulk load or after adding a new index on a skewed column.

### default_statistics_target

Controls how many histogram buckets are collected per column. Default is 100. Increase for columns with skewed distributions or high cardinality that appear in WHERE clauses.

```sql
-- postgresql.conf (global)
default_statistics_target = 100   -- default, fine for most columns

-- Per-column override (preferred — targeted, no restart needed)
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 500;
ALTER TABLE events ALTER COLUMN user_id SET STATISTICS 1000;

-- Then run ANALYZE to apply
ANALYZE orders;
ANALYZE events;
```

Higher values improve estimates but use more memory during planning. 200–500 is a practical range for important columns; 1000 for highly skewed ones.

### Extended Statistics for Correlated Columns

When two columns are correlated (e.g., `city` and `zip_code`) the planner multiplies their selectivities independently, massively underestimating how many rows a combined WHERE clause will match.

```sql
-- Tell the planner these columns are correlated
CREATE STATISTICS orders_status_region ON status, region FROM orders;

-- Apply
ANALYZE orders;
```

Check if extended stats are being used:

```sql
SELECT stxname, stxkeys, stxkind
FROM pg_statistic_ext
WHERE stxrelid = 'orders'::regclass;
```

---

## work_mem and Sorting

### The Problem

`work_mem` controls the amount of memory allocated for each sort, hash, or merge operation. The default is 4 MB. Any sort that exceeds this spills to disk as a temporary file.

Identify spill-to-disk in EXPLAIN ANALYZE output:

```
Sort  (cost=... rows=...) (actual rows=...)
  Sort Key: created_at DESC
  Sort Method: external merge  Disk: 42784kB    <-- spilling to disk
```

### Set Per-Session for Heavy Queries

Never set `work_mem` globally to a high value — it applies per operation, and a complex query can run dozens of concurrent sorts. The safe pattern is to set it per session or per transaction before the query, then reset.

```go
// internal/repository/analytics_repo.go
package repository

import (
    "context"
    "fmt"

    "github.com/jmoiron/sqlx"
)

func (r *AnalyticsRepo) SalesReport(ctx context.Context, filters ReportFilters) ([]SalesRow, error) {
    tx, err := r.db.BeginTxx(ctx, nil)
    if err != nil {
        return nil, fmt.Errorf("begin tx: %w", err)
    }
    defer tx.Rollback() //nolint:errcheck

    // Bump work_mem for this transaction only
    if _, err := tx.ExecContext(ctx, "SET LOCAL work_mem = '256MB'"); err != nil {
        return nil, fmt.Errorf("set work_mem: %w", err)
    }

    const q = `
        SELECT date_trunc('day', created_at) AS day,
               region,
               sum(total_amount)             AS revenue,
               count(*)                      AS order_count
        FROM orders
        WHERE created_at >= $1 AND created_at < $2
        GROUP BY 1, 2
        ORDER BY 1, 2`

    var rows []SalesRow
    if err := tx.SelectContext(ctx, &rows, q, filters.From, filters.To); err != nil {
        return nil, fmt.Errorf("sales report: %w", err)
    }

    return rows, tx.Commit()
}
```

`SET LOCAL` scopes the change to the current transaction — it resets automatically on commit or rollback.

### Signs of Insufficient work_mem

- `Sort Method: external merge Disk` in EXPLAIN ANALYZE
- Temporary files growing large: `SELECT pg_size_pretty(temp_bytes) FROM pg_stat_statements WHERE temp_bytes > 0 ORDER BY temp_bytes DESC`
- Analytics queries slow despite a good index (GROUP BY and ORDER BY also use sort memory)

---

## Connection Pool Sizing

### Formula

```
PostgreSQL max_connections = (core_count * 2) + effective_spindle_count
```

For SSDs: `effective_spindle_count = 1` (SSD handles parallel I/O natively). A 4-core server with SSD → `max_connections = 9`. A 16-core server → 33.

This is the PostgreSQL backend limit, not the application pool size. The application pool should be smaller — leave headroom for migrations, psql sessions, and monitoring queries.

**Practical split for a single app on 4 cores:**

| Role | Connections |
|---|---|
| Application pool (`MaxOpenConns`) | 20 |
| PgBouncer reserve pool | 5 |
| Admin / monitoring / migrations | 5 |
| Total `max_connections` | 30 |

### Go `database/sql` Pool Settings

```go
// internal/db/pool.go
package db

import (
    "database/sql"
    "time"
)

func ConfigurePool(sqlDB *sql.DB) {
    sqlDB.SetMaxOpenConns(20)                    // Hard cap on concurrent connections
    sqlDB.SetMaxIdleConns(5)                     // Keep 5 warm — avoid reconnect overhead
    sqlDB.SetConnMaxLifetime(5 * time.Minute)    // Recycle to distribute load across replicas
    sqlDB.SetConnMaxIdleTime(1 * time.Minute)    // Release idle connections in quiet periods
}
```

`ConnMaxLifetime` matters most when you have a load balancer or PgBouncer in front — it ensures connections are periodically re-established and land on different backends.

### PgBouncer: Transaction vs Session Pooling

| Mode | How It Works | Use When |
|---|---|---|
| **Transaction** (recommended) | Connection released to pool after each transaction | Stateless apps — works with all Go drivers |
| **Session** | Connection held for the client's entire session | Required for `SET` commands, temp tables, advisory locks |
| **Statement** | Released after each statement | Rarely useful; breaks multi-statement transactions |

PgBouncer `pgbouncer.ini` essentials:

```ini
[databases]
mydb = host=127.0.0.1 port=5432 dbname=mydb

[pgbouncer]
pool_mode = transaction
max_client_conn = 1000       # clients (app goroutines)
default_pool_size = 20       # backend connections to PostgreSQL
reserve_pool_size = 5        # extra for spikes
reserve_pool_timeout = 5
server_idle_timeout = 60
```

**Go-side pool with PgBouncer:** Set `MaxOpenConns` to match `default_pool_size`. Setting it higher wastes connections in the PgBouncer queue. Setting it lower underutilizes the pool.

### Monitor Active Connections

```sql
SELECT
    datname,
    usename,
    state,
    count(*) AS count,
    max(now() - state_change) AS longest_in_state
FROM pg_stat_activity
WHERE datname IS NOT NULL
GROUP BY datname, usename, state
ORDER BY count DESC;
```

---

## Read Replicas

### Streaming Replication Overview

PostgreSQL streaming replication sends WAL records from the primary to one or more standbys in near real-time. Standbys are read-only. Replication lag (the standby being slightly behind the primary) is the key operational concern.

### Go Pattern: Separate Read/Write Pools

```go
// internal/db/replica.go
package db

import (
    "fmt"
    "log/slog"
    "time"

    "github.com/jmoiron/sqlx"
    _ "github.com/lib/pq"
)

type Connections struct {
    Primary *sqlx.DB // writes + reads that require strong consistency
    Replica *sqlx.DB // reads that tolerate replication lag
}

func NewConnections(primaryDSN, replicaDSN string, logger *slog.Logger) (*Connections, error) {
    primary, err := sqlx.Connect("postgres", primaryDSN)
    if err != nil {
        return nil, fmt.Errorf("primary connect: %w", err)
    }
    primary.SetMaxOpenConns(20)
    primary.SetMaxIdleConns(5)
    primary.SetConnMaxLifetime(5 * time.Minute)

    replica, err := sqlx.Connect("postgres", replicaDSN)
    if err != nil {
        return nil, fmt.Errorf("replica connect: %w", err)
    }
    replica.SetMaxOpenConns(30) // replicas can serve more reads
    replica.SetMaxIdleConns(10)
    replica.SetConnMaxLifetime(5 * time.Minute)

    logger.Info("database connections established", "primary", primaryDSN, "replica", replicaDSN)
    return &Connections{Primary: primary, Replica: replica}, nil
}
```

### Query Routing in Repository Layer

```go
// internal/repository/order_repo.go
package repository

import (
    "context"
    "fmt"

    "github.com/jmoiron/sqlx"
    "myapp/internal/db"
)

type OrderRepo struct {
    conns *db.Connections
}

// Create always writes to primary.
func (r *OrderRepo) Create(ctx context.Context, o *Order) error {
    const q = `INSERT INTO orders (user_id, total_amount, status) VALUES (:user_id, :total_amount, :status) RETURNING id, created_at`
    rows, err := r.conns.Primary.NamedQueryContext(ctx, q, o)
    if err != nil {
        return fmt.Errorf("order insert: %w", err)
    }
    defer rows.Close()
    if rows.Next() {
        return rows.Scan(&o.ID, &o.CreatedAt)
    }
    return nil
}

// List reads from replica — tolerate slight lag.
func (r *OrderRepo) List(ctx context.Context, userID string) ([]Order, error) {
    const q = `SELECT id, user_id, total_amount, status, created_at FROM orders WHERE user_id = $1 ORDER BY created_at DESC LIMIT 100`
    var orders []Order
    if err := r.conns.Replica.SelectContext(ctx, &orders, q, userID); err != nil {
        return nil, fmt.Errorf("order list: %w", err)
    }
    return orders, nil
}

// GetByID reads from primary when called after a write (e.g., post-create redirect).
func (r *OrderRepo) GetByID(ctx context.Context, id string, strong bool) (*Order, error) {
    db := r.conns.Replica
    if strong {
        db = r.conns.Primary // caller signals: "I just wrote this, read my own write"
    }
    var o Order
    if err := db.GetContext(ctx, &o, `SELECT * FROM orders WHERE id = $1`, id); err != nil {
        return nil, fmt.Errorf("order get: %w", err)
    }
    return &o, nil
}
```

### Replication Lag Monitoring

```sql
-- Run on primary
SELECT
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    write_lag,
    flush_lag,
    replay_lag
FROM pg_stat_replication;
```

Alert when `replay_lag` exceeds 30 seconds. On the replica itself:

```sql
SELECT now() - pg_last_xact_replay_timestamp() AS replication_lag;
```

---

## Monitoring Go Helper

A single struct and function that collects key metrics from `pg_stat_*` views and exposes them on a Gin health endpoint.

```go
// internal/db/health.go
package db

import (
    "context"
    "database/sql"
    "fmt"
    "log/slog"
    "time"

    "github.com/jmoiron/sqlx"
)

// DBHealth holds the metrics returned by the health endpoint.
type DBHealth struct {
    Timestamp      time.Time `json:"timestamp"`
    MaxConnections int       `json:"max_connections"`
    OpenConns      int       `json:"open_connections"`
    InUseConns     int       `json:"in_use_connections"`
    IdleConns      int       `json:"idle_connections"`
    CacheHitPct    float64   `json:"cache_hit_pct"`
    DeadTuplesTop  []TableBloat `json:"dead_tuples_top"`
    SlowQueriesTop []SlowQuery  `json:"slow_queries_top"`
}

type TableBloat struct {
    TableName   string  `json:"table_name"  db:"table_name"`
    DeadTuples  int64   `json:"dead_tuples" db:"dead_tuples"`
    DeadPct     float64 `json:"dead_pct"    db:"dead_pct"`
}

// CollectDBHealth gathers pool stats, cache hit ratio, top bloated tables,
// and top slow queries. Each sub-query is best-effort — failures are logged
// but do not prevent the rest from running.
func CollectDBHealth(ctx context.Context, db *sqlx.DB, logger *slog.Logger) DBHealth {
    h := DBHealth{Timestamp: time.Now()}

    // Pool stats from database/sql
    stats := db.Stats()
    h.OpenConns = stats.OpenConnections
    h.InUseConns = stats.InUse
    h.IdleConns = stats.Idle

    var maxConns int
    if err := db.QueryRowContext(ctx, "SHOW max_connections").Scan(&maxConns); err != nil {
        logger.WarnContext(ctx, "health: max_connections query failed", "err", err)
    }
    h.MaxConnections = maxConns

    // Cache hit ratio
    var hit, read sql.NullFloat64
    err := db.QueryRowContext(ctx, `
        SELECT
            sum(blks_hit)  AS hit,
            sum(blks_read) AS read
        FROM pg_stat_database
        WHERE datname = current_database()`).Scan(&hit, &read)
    if err != nil {
        logger.WarnContext(ctx, "health: cache hit query failed", "err", err)
    } else if hit.Valid && read.Valid && (hit.Float64+read.Float64) > 0 {
        h.CacheHitPct = hit.Float64 / (hit.Float64 + read.Float64) * 100
    }

    // Top bloated tables
    bloatQ := `
        SELECT
            relname                                                              AS table_name,
            n_dead_tup                                                           AS dead_tuples,
            round(n_dead_tup::numeric / NULLIF(n_live_tup + n_dead_tup, 0) * 100, 2)
                                                                                 AS dead_pct
        FROM pg_stat_user_tables
        WHERE n_live_tup > 1000
        ORDER BY n_dead_tup DESC
        LIMIT 5`
    if err := db.SelectContext(ctx, &h.DeadTuplesTop, bloatQ); err != nil {
        logger.WarnContext(ctx, "health: bloat query failed", "err", err)
    }

    // Top slow queries (requires pg_stat_statements)
    slowQ := `
        SELECT
            left(query, 80)                   AS query_preview,
            calls,
            round(mean_exec_time::numeric, 2) AS mean_ms,
            round(total_exec_time::numeric, 2) AS total_ms
        FROM pg_stat_statements
        WHERE calls > 5
        ORDER BY mean_exec_time DESC
        LIMIT 5`
    if err := db.SelectContext(ctx, &h.SlowQueriesTop, slowQ); err != nil {
        // pg_stat_statements may not be installed — warn, don't error
        logger.WarnContext(ctx, "health: pg_stat_statements unavailable", "err", err)
    }

    logger.InfoContext(ctx, "db health collected",
        "cache_hit_pct", fmt.Sprintf("%.2f", h.CacheHitPct),
        "open_conns", h.OpenConns,
        "dead_tables_found", len(h.DeadTuplesTop),
    )
    return h
}
```

### Gin Health Endpoint

```go
// internal/handler/health_handler.go
package handler

import (
    "net/http"

    "github.com/gin-gonic/gin"
    "github.com/jmoiron/sqlx"
    appdb "myapp/internal/db"
    "log/slog"
)

type HealthHandler struct {
    db     *sqlx.DB
    logger *slog.Logger
}

func NewHealthHandler(db *sqlx.DB, logger *slog.Logger) *HealthHandler {
    return &HealthHandler{db: db, logger: logger}
}

// GET /internal/health/db
func (h *HealthHandler) DBHealth(c *gin.Context) {
    health := appdb.CollectDBHealth(c.Request.Context(), h.db, h.logger)

    status := http.StatusOK
    if health.CacheHitPct < 90 && health.CacheHitPct > 0 {
        status = http.StatusMultiStatus // 207 — degraded
    }

    c.JSON(status, health)
}
```

Register in your router:

```go
internal := r.Group("/internal")
internal.Use(internalAuthMiddleware())
{
    internal.GET("/health/db", healthHandler.DBHealth)
}
```

---

## Quick Tuning Checklist

Apply these settings when provisioning a new PostgreSQL instance. Adjust based on actual RAM and core count.

```
shared_buffers             = 25% of RAM
                             (e.g., 32GB RAM → 8GB)
                             First layer of cache — most impactful single setting.

effective_cache_size       = 75% of RAM
                             (e.g., 32GB RAM → 24GB)
                             Hints to planner how much OS + PostgreSQL cache is available.
                             Does not allocate memory — planner estimate only.

work_mem                   = RAM / max_connections / 4
                             (e.g., 32GB / 100 / 4 = 80MB)
                             Applies per sort/hash operation. Keep conservative globally;
                             override per session for analytics.

maintenance_work_mem       = RAM / 8, max 2GB
                             (e.g., 32GB / 8 = 4GB → cap at 2GB)
                             Used by VACUUM, CREATE INDEX, CLUSTER. Higher = faster vacuums.

random_page_cost           = 1.1 for SSD  (default 4.0 assumes spinning disk)
                             Tells planner index scans are almost as cheap as sequential.

effective_io_concurrency   = 200 for SSD  (default 1)
                             Lets the planner issue more parallel I/O for bitmap heap scans.

max_worker_processes       = core_count
max_parallel_workers       = core_count
max_parallel_workers_per_gather = core_count / 2
                             Enable parallel query for large table scans and aggregations.

wal_buffers                = 64MB  (default 4MB — often the bottleneck for write-heavy apps)

checkpoint_completion_target = 0.9  (spread checkpoint I/O over 90% of checkpoint interval)
```

**Checklist:**

- [ ] `shared_buffers` set to 25% of RAM
- [ ] `effective_cache_size` set to 75% of RAM
- [ ] `random_page_cost = 1.1` if running on SSD
- [ ] `effective_io_concurrency = 200` if running on SSD
- [ ] `work_mem` sized conservatively; heavy queries use `SET LOCAL`
- [ ] `maintenance_work_mem` at least 512 MB
- [ ] `pg_stat_statements` extension created in target database
- [ ] Autovacuum scale factors lowered for all tables > 1M rows
- [ ] `max_connections` matches formula: `(core_count * 2) + 1`
- [ ] Application pool `MaxOpenConns` ≤ PostgreSQL `max_connections` minus admin headroom
- [ ] Replication lag monitored; alert at 30 seconds
- [ ] `xid_age` monitored; alert at 150,000,000

**Cross-references:**

- Index selection and EXPLAIN ANALYZE deep dive: [index-strategy.md](index-strategy.md)
- Schema design and data types: [schema-design.md](schema-design.md)
- Zero-downtime migrations: [migration-impact-analysis.md](migration-impact-analysis.md)
- Extension ecosystem (pg_cron, pg_partman, pgstattuple): [extensions-toolkit.md](extensions-toolkit.md)
