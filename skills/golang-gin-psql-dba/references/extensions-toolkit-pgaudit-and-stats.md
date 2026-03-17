# Extensions Toolkit — pg_stat_statements, pgAudit, and Compatibility Matrix

## pg_stat_statements — Query Statistics

### Setup

```ini
# postgresql.conf (requires server restart)
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.track = all
pg_stat_statements.max = 10000
pg_stat_statements.track_utility = off
```

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

### Key Queries

```sql
-- Top 10 slowest by total time
SELECT LEFT(query, 120) AS query, calls,
    ROUND(total_exec_time::numeric, 2) AS total_ms,
    ROUND(mean_exec_time::numeric, 2)  AS mean_ms,
    rows
FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 10;

-- High cache miss queries
SELECT LEFT(query, 120) AS query, shared_blks_read, shared_blks_hit,
    ROUND(100.0 * shared_blks_read /
        NULLIF(shared_blks_read + shared_blks_hit, 0), 1) AS cache_miss_pct
FROM pg_stat_statements
WHERE shared_blks_read + shared_blks_hit > 1000
ORDER BY cache_miss_pct DESC LIMIT 10;

-- Reset stats
SELECT pg_stat_statements_reset();
```

### Go Monitoring Function

```go
type SlowQuery struct {
    Query    string  `db:"query"`
    Calls    int64   `db:"calls"`
    TotalMS  float64 `db:"total_ms"`
    MeanMS   float64 `db:"mean_ms"`
}

func TopSlowQueries(ctx context.Context, db *sqlx.DB, n int) ([]SlowQuery, error) {
    const q = `
        SELECT LEFT(query, 200) AS query, calls,
            ROUND(total_exec_time::numeric, 2) AS total_ms,
            ROUND(mean_exec_time::numeric, 2)  AS mean_ms
        FROM pg_stat_statements WHERE calls > 10
        ORDER BY total_exec_time DESC LIMIT $1`
    var rows []SlowQuery
    if err := db.SelectContext(ctx, &rows, q, n); err != nil {
        return nil, fmt.Errorf("top slow queries: %w", err)
    }
    return rows, nil
}
```

---

## pgAudit — Audit Logging

### Setup

```ini
# postgresql.conf (requires server restart)
shared_preload_libraries = 'pgaudit'
pgaudit.log = 'write, ddl'
pgaudit.log_catalog = off
pgaudit.log_relation = on
pgaudit.log_parameter = off
```

```sql
CREATE EXTENSION IF NOT EXISTS pgaudit;
```

### Log Classes

| Class | Logs |
|---|---|
| `READ` | `SELECT`, `COPY FROM` |
| `WRITE` | `INSERT`, `UPDATE`, `DELETE`, `TRUNCATE` |
| `ROLE` | `GRANT`, `REVOKE`, `CREATE/ALTER/DROP ROLE` |
| `DDL` | All DDL except ROLE class |
| `ALL` | Everything |

**Recommended minimum for production:** `pgaudit.log = 'write, ddl, role'`

### Per-Role Auditing

```sql
ALTER ROLE admin_user SET pgaudit.log = 'all';
ALTER ROLE api_user SET pgaudit.log = 'write';
```

### pgAudit vs Application-Level Audit

| Factor | pgAudit | Application Audit Table |
|---|---|---|
| Coverage | All SQL, including direct DB access | Only operations through your app |
| Bypass risk | Cannot be bypassed by app code | Can be bypassed |
| Compliance | SOC 2, PCI-DSS require DB-level audit | Insufficient alone |

**Pattern:** Use both. pgAudit for compliance; application audit table for product features.

```sql
CREATE TABLE audit_log (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    table_name  TEXT NOT NULL, record_id TEXT NOT NULL,
    operation   TEXT NOT NULL,  -- INSERT, UPDATE, DELETE
    actor_id    UUID, old_data JSONB, new_data JSONB,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_log_record ON audit_log (table_name, record_id);
CREATE INDEX idx_audit_log_time   ON audit_log USING brin (created_at);
```

---

## Extension Compatibility Matrix

| Extension | Min PG | AWS RDS | Cloud SQL | Supabase | Needs `shared_preload_libraries` |
|---|---|---|---|---|---|
| pgcrypto | 8.3 | Yes | Yes | Yes | No |
| uuid-ossp | 8.3 | Yes | Yes | Yes | No |
| pg_trgm | 8.3 | Yes | Yes | Yes | No |
| pg_stat_statements | 9.4 | Yes | Yes | Yes | **Yes** |
| pg_cron | 9.5 | Yes (v10.4+) | No | No | **Yes** |
| pg_partman | 10 | No | No | No | No |
| pgAudit | 9.5 | Yes | Yes | No | **Yes** |
| pgvector | 11 | Yes (v15.2+) | Yes | Yes | No |
| PostGIS | 9.4 | Yes | Yes | Yes | No |
| TimescaleDB | 12 | No | No | No | **Yes** |
| ParadeDB (pg_search) | 14 | No | No | No | No |

**Notes:**
- `shared_preload_libraries` extensions require a **server restart**. On managed DBs, this means a maintenance window.
- **pg_cron on Cloud SQL / Supabase**: not supported. Use Cloud Scheduler or Kubernetes CronJob instead.
- **RDS pg_cron**: available on RDS PostgreSQL 12.5+ and Aurora PostgreSQL 12.6+.
