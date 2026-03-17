# Query Performance — work_mem, Connection Pools, Read Replicas, and Monitoring

## work_mem and Sorting

`work_mem` controls memory per sort/hash operation. Default is 4 MB. Exceeding it spills to disk.

Identify spill in EXPLAIN ANALYZE:
```
Sort Method: external merge  Disk: 42784kB   <-- spilling to disk
```

### Set Per-Transaction for Heavy Queries

```go
func (r *AnalyticsRepo) SalesReport(ctx context.Context, filters ReportFilters) ([]SalesRow, error) {
    tx, _ := r.db.BeginTxx(ctx, nil)
    defer tx.Rollback()
    if _, err := tx.ExecContext(ctx, "SET LOCAL work_mem = '256MB'"); err != nil {
        return nil, fmt.Errorf("set work_mem: %w", err)
    }
    const q = `SELECT date_trunc('day', created_at) AS day, region,
               sum(total_amount) AS revenue, count(*) AS order_count
        FROM orders WHERE created_at >= $1 AND created_at < $2
        GROUP BY 1, 2 ORDER BY 1, 2`
    var rows []SalesRow
    if err := tx.SelectContext(ctx, &rows, q, filters.From, filters.To); err != nil {
        return nil, fmt.Errorf("sales report: %w", err)
    }
    return rows, tx.Commit()
}
```

---

## Connection Pool Sizing

**Formula:** `max_connections = (core_count * 2) + 1` for SSD.

| Role | Connections |
|---|---|
| Application pool (`MaxOpenConns`) | 20 |
| PgBouncer reserve pool | 5 |
| Admin / monitoring / migrations | 5 |
| Total `max_connections` | 30 |

### Go Pool Settings

```go
func ConfigurePool(sqlDB *sql.DB) {
    sqlDB.SetMaxOpenConns(20)
    sqlDB.SetMaxIdleConns(5)
    sqlDB.SetConnMaxLifetime(5 * time.Minute)
    sqlDB.SetConnMaxIdleTime(1 * time.Minute)
}
```

### PgBouncer

```ini
[databases]
mydb = host=127.0.0.1 port=5432 dbname=mydb
[pgbouncer]
pool_mode = transaction   # recommended for stateless Go apps
max_client_conn = 1000; default_pool_size = 20; reserve_pool_size = 5; server_idle_timeout = 60
```

| Mode | Use When |
|---|---|
| **Transaction** (recommended) | Stateless apps — released after each transaction |
| **Session** | Required for `SET` commands, temp tables, advisory locks |

### Monitor Active Connections

```sql
SELECT datname, usename, state, count(*) AS count, max(now() - state_change) AS longest_in_state
FROM pg_stat_activity WHERE datname IS NOT NULL GROUP BY datname, usename, state ORDER BY count DESC;
```

---

## Read Replicas

```go
type Connections struct {
    Primary *sqlx.DB  // writes + strong-consistency reads
    Replica *sqlx.DB  // reads tolerating replication lag
}

// List reads from replica
func (r *OrderRepo) List(ctx context.Context, userID string) ([]Order, error) {
    const q = `SELECT * FROM orders WHERE user_id = $1 ORDER BY created_at DESC LIMIT 100`
    var orders []Order
    if err := r.conns.Replica.SelectContext(ctx, &orders, q, userID); err != nil {
        return nil, fmt.Errorf("order list: %w", err)
    }
    return orders, nil
}

// GetByID: strong=true forces primary (read-your-writes after a write)
func (r *OrderRepo) GetByID(ctx context.Context, id string, strong bool) (*Order, error) {
    db := r.conns.Replica
    if strong { db = r.conns.Primary }
    var o Order
    if err := db.GetContext(ctx, &o, `SELECT * FROM orders WHERE id = $1`, id); err != nil {
        return nil, fmt.Errorf("order get: %w", err)
    }
    return &o, nil
}
```

Replication lag monitoring: see [replication-and-ha-lag-monitoring.md](replication-and-ha-lag-monitoring.md).

---

## Monitoring Go Helper and Health Endpoint

```go
// GET /internal/health/db
func (h *HealthHandler) DBHealth(c *gin.Context) {
    health := appdb.CollectDBHealth(c.Request.Context(), h.db, h.logger)
    status := http.StatusOK
    if health.CacheHitPct < 90 && health.CacheHitPct > 0 {
        status = http.StatusMultiStatus  // 207 — degraded
    }
    c.JSON(status, health)
}
```

`CollectDBHealth` queries `db.Stats()`, `pg_stat_database` (cache hit %), `pg_stat_user_tables` (dead tuples top 5), and `pg_stat_statements` (slow queries top 5).

## Quick Tuning Checklist

```
shared_buffers           = 25% of RAM
effective_cache_size     = 75% of RAM
work_mem                 = RAM / max_connections / 4
maintenance_work_mem     = RAM / 8, max 2GB
random_page_cost         = 1.1 (SSD)
effective_io_concurrency = 200 (SSD)
max_parallel_workers     = core_count
wal_buffers              = 64MB
checkpoint_completion_target = 0.9
```

- [ ] Settings applied above (shared_buffers, random_page_cost, effective_io_concurrency)
- [ ] `pg_stat_statements` extension created; autovacuum scale factors lowered for hot tables
- [ ] `max_connections` formula applied; app `MaxOpenConns` leaves admin headroom
- [ ] Replication lag alert at 30s; `xid_age` alert at 150,000,000
