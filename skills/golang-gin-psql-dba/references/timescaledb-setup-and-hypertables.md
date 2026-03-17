# TimescaleDB — Setup and Hypertables

TimescaleDB turns ordinary PostgreSQL tables into hypertables — automatically partitioned by time — with first-class time-series functions. Transparent to standard SQL: every driver and ORM that works with PostgreSQL works unchanged.

> **Architectural note:** TimescaleDB patterns are PostgreSQL DBA decisions, not part of the Gin framework API. All Go examples use sqlx with raw SQL.

## Overview

| Feature | TimescaleDB | Plain PostgreSQL partitioning |
|---|---|---|
| Partition creation | Automatic, by time interval | Manual `CREATE TABLE ... PARTITION OF` |
| Time aggregation | `time_bucket()` built-in | Custom expressions |
| Materialized aggregates | Continuous aggregates with auto-refresh | Manual `REFRESH MATERIALIZED VIEW` |
| Compression | Columnar per-chunk, 10–20x ratio | Not available |
| Retention | `add_retention_policy()` — drops chunks | Manual `DROP TABLE partition_name` |

**When to choose TimescaleDB:** table is append-dominant ordered by time, queries always include a time range filter, need time-bucketed aggregations or downsampling.

## Setup

Use Docker image `timescale/timescaledb:latest-pg16` (or `timescale/timescaledb-ha:pg16-latest` for production with PostGIS).

```sql
-- 000001_enable_timescaledb.sql
CREATE EXTENSION IF NOT EXISTS timescaledb;
```

## Hypertables

A hypertable is a regular PostgreSQL table with automatic time-based partitioning into *chunks*. Each chunk covers a time interval (default: 7 days).

```sql
CREATE TABLE metrics (
    time        TIMESTAMPTZ        NOT NULL,
    device_id   TEXT               NOT NULL,
    metric      TEXT               NOT NULL,
    value       DOUBLE PRECISION   NOT NULL,
    tags        JSONB
);

-- Convert to hypertable — partitioned by 'time', 7-day chunks (default)
SELECT create_hypertable('metrics', by_range('time'));

-- Custom chunk interval: 1 day (use for high-volume tables)
SELECT create_hypertable('metrics', by_range('time', INTERVAL '1 day'));
```

Indexes created on the parent propagate to all chunks:

```sql
CREATE INDEX idx_metrics_device_time ON metrics (device_id, time DESC);
CREATE INDEX idx_metrics_metric_time ON metrics (metric, time DESC);
```

Space partitioning for very high cardinality:

```sql
-- Spread across 4 hash buckets by device_id
SELECT add_dimension('metrics', by_hash('device_id', 4));
```

### API Metrics Table DDL

```sql
-- migrations/000002_create_api_metrics.sql
CREATE TABLE api_metrics (
    time        TIMESTAMPTZ      NOT NULL DEFAULT now(),
    endpoint    TEXT             NOT NULL,
    method      TEXT             NOT NULL,
    status      INTEGER          NOT NULL,
    duration_ms DOUBLE PRECISION NOT NULL,
    user_id     TEXT,
    ip          TEXT
);

SELECT create_hypertable('api_metrics', by_range('time', INTERVAL '1 day'));

CREATE INDEX idx_api_metrics_endpoint ON api_metrics (endpoint, time DESC);
CREATE INDEX idx_api_metrics_status   ON api_metrics (status, time DESC);
```

Converting an existing table with data:

```sql
SELECT create_hypertable('events', by_range('created_at'), migrate_data => true);
```

## time_bucket — Time-Based Aggregation

`time_bucket` truncates a timestamp to a fixed interval — the core function for time-series aggregations.

```sql
-- Per-minute request count
SELECT time_bucket('1 minute', time) AS bucket, COUNT(*) AS requests
FROM api_metrics
WHERE time >= now() - INTERVAL '1 hour'
GROUP BY bucket ORDER BY bucket DESC;

-- Per-hour average duration by endpoint
SELECT time_bucket('1 hour', time) AS bucket, endpoint,
       AVG(duration_ms) AS avg_ms, MAX(duration_ms) AS max_ms, COUNT(*) AS total
FROM api_metrics
WHERE time >= now() - INTERVAL '24 hours'
GROUP BY bucket, endpoint ORDER BY bucket DESC, total DESC;

-- Per-day error rate
SELECT time_bucket('1 day', time) AS bucket,
       COUNT(*) FILTER (WHERE status >= 500) AS errors,
       ROUND(100.0 * COUNT(*) FILTER (WHERE status >= 500) / NULLIF(COUNT(*), 0), 2) AS error_pct
FROM api_metrics
WHERE time >= now() - INTERVAL '30 days'
GROUP BY bucket ORDER BY bucket DESC;
```

### Go Query for Hourly Stats

```go
type HourlyStat struct {
    Bucket   time.Time `db:"bucket"`
    Requests int64     `db:"requests"`
    AvgMs    float64   `db:"avg_ms"`
    Errors   int64     `db:"errors"`
}

func (r *MetricsRepository) GetHourlyStats(ctx context.Context, start, end time.Time) ([]HourlyStat, error) {
    const q = `
        SELECT time_bucket('1 hour', time) AS bucket,
               COUNT(*) AS requests, AVG(duration_ms) AS avg_ms,
               COUNT(*) FILTER (WHERE status >= 500) AS errors
        FROM api_metrics
        WHERE time >= $1 AND time < $2
        GROUP BY bucket ORDER BY bucket DESC`
    var rows []HourlyStat
    if err := r.db.SelectContext(ctx, &rows, q, start, end); err != nil {
        return nil, fmt.Errorf("GetHourlyStats: %w", err)
    }
    return rows, nil
}
```
