# TimescaleDB Time-Series Reference

TimescaleDB is a PostgreSQL extension that turns ordinary tables into hypertables — automatically partitioned by time into chunks — and adds first-class time-series functions (`time_bucket`, continuous aggregates, compression, retention policies). It is transparent to standard SQL tooling: every query, driver, and ORM that works with PostgreSQL works unchanged. Use it for IoT sensor data, API analytics, financial tick data, and operational monitoring where plain PostgreSQL partitioning would require manual chunk management and lacks specialized aggregation functions.

> **Architectural note:** TimescaleDB patterns are PostgreSQL DBA decisions, not part of the Gin framework API. All Go examples use sqlx with raw SQL.

## Table of Contents

1. [TimescaleDB Overview](#timescaledb-overview)
2. [Setup](#setup)
3. [Hypertables](#hypertables)
4. [time_bucket — Time-Based Aggregation](#time_bucket--time-based-aggregation)
5. [Continuous Aggregates](#continuous-aggregates)
6. [Compression](#compression)
7. [Retention Policies](#retention-policies)
8. [Downsampling](#downsampling)
9. [Go Integration](#go-integration)
10. [API Metrics Example (Complete)](#api-metrics-example-complete)

---

## TimescaleDB Overview

| Feature | TimescaleDB | Plain PostgreSQL partitioning |
|---|---|---|
| Partition creation | Automatic, by time interval | Manual `CREATE TABLE ... PARTITION OF` |
| Chunk management | Built-in drop/compress | Manual per-partition DDL |
| Time aggregation | `time_bucket()` built-in | Custom expressions |
| Materialized aggregates | Continuous aggregates with auto-refresh | Manual `REFRESH MATERIALIZED VIEW` |
| Compression | Columnar per-chunk, 10–20x ratio | Not available |
| Retention | `add_retention_policy()` — drops chunks | Manual `DROP TABLE partition_name` |

**When to choose TimescaleDB:**
- Table is append-dominant and ordered by time (sensor data, logs, events, metrics)
- Queries always include a time range filter
- Need time-bucketed aggregations (per-minute, per-hour)
- Need to retain summarized data longer than raw data (downsampling)

**When plain partitioning is enough:**
- Table is partitioned by something other than time (region, tenant)
- No need for aggregation functions or compression
- DBA preference to avoid extensions

---

## Setup

**Docker image:**

```yaml
# docker-compose.yml — development
services:
  db:
    image: timescale/timescaledb:latest-pg16
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: appdb
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./migrations:/docker-entrypoint-initdb.d  # runs *.sql on first start

volumes:
  pgdata:
```

For production image with PostGIS: `timescale/timescaledb-ha:pg16-latest` includes PostGIS and pg_vector.

**Enable the extension (once per database):**

```sql
-- 000001_enable_timescaledb.sql
CREATE EXTENSION IF NOT EXISTS timescaledb;
```

Run this in your first migration. All subsequent migrations use standard SQL — no TimescaleDB-specific migration tool needed.

**Cross-reference:** For full docker-compose patterns with health checks and Air hot reload, see the **golang-gin-deploy** skill → `references/docker-compose.md`.

---

## Hypertables

A hypertable is a regular PostgreSQL table with automatic time-based partitioning into *chunks*. Each chunk is a child table covering a time interval (default: 7 days). Creating a hypertable is a one-time DDL call after `CREATE TABLE`.

**Create and convert to hypertable:**

```sql
-- Standard CREATE TABLE first
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

**Indexes on hypertables** — create on the parent table, TimescaleDB propagates to all chunks:

```sql
CREATE INDEX idx_metrics_device_time ON metrics (device_id, time DESC);
CREATE INDEX idx_metrics_metric_time ON metrics (metric, time DESC);
```

**Space partitioning (multi-dimensional)** — add a second partition dimension for very high cardinality:

```sql
-- Partition by time AND spread across 4 hash buckets by device_id
-- Useful when a single time chunk would be too large (>10M rows/chunk)
SELECT add_dimension('metrics', by_hash('device_id', 4));
```

**Complete DDL for API metrics table:**

```sql
-- migrations/000002_create_api_metrics.sql

CREATE TABLE api_metrics (
    time        TIMESTAMPTZ     NOT NULL DEFAULT now(),
    endpoint    TEXT            NOT NULL,  -- e.g. '/api/v1/users'
    method      TEXT            NOT NULL,  -- GET, POST, etc.
    status      INTEGER         NOT NULL,  -- HTTP status code
    duration_ms DOUBLE PRECISION NOT NULL, -- request duration
    user_id     TEXT,                      -- nullable for unauthenticated
    ip          TEXT
);

SELECT create_hypertable('api_metrics', by_range('time', INTERVAL '1 day'));

CREATE INDEX idx_api_metrics_endpoint ON api_metrics (endpoint, time DESC);
CREATE INDEX idx_api_metrics_status   ON api_metrics (status, time DESC);
```

**Converting an existing table** (must be empty, or use `migrate_data => true`):

```sql
-- Existing table with data — TimescaleDB will migrate rows into chunks
SELECT create_hypertable('events', by_range('created_at'), migrate_data => true);
```

---

## time_bucket — Time-Based Aggregation

`time_bucket` truncates a timestamp to a fixed interval, like `date_trunc` but for arbitrary intervals. It is the core function for time-series aggregations.

**Syntax:**

```sql
time_bucket('interval', timestamp_column) → TIMESTAMPTZ
```

**Examples:**

```sql
-- Per-minute request count
SELECT time_bucket('1 minute', time) AS bucket,
       COUNT(*)                       AS requests
FROM api_metrics
WHERE time >= now() - INTERVAL '1 hour'
GROUP BY bucket
ORDER BY bucket DESC;

-- Per-hour average duration by endpoint
SELECT time_bucket('1 hour', time)  AS bucket,
       endpoint,
       AVG(duration_ms)              AS avg_ms,
       MAX(duration_ms)              AS max_ms,
       COUNT(*)                      AS total
FROM api_metrics
WHERE time >= now() - INTERVAL '24 hours'
GROUP BY bucket, endpoint
ORDER BY bucket DESC, total DESC;

-- Per-day error rate
SELECT time_bucket('1 day', time)                          AS bucket,
       COUNT(*) FILTER (WHERE status >= 500)               AS errors,
       COUNT(*)                                            AS total,
       ROUND(100.0 * COUNT(*) FILTER (WHERE status >= 500)
             / NULLIF(COUNT(*), 0), 2)                     AS error_pct
FROM api_metrics
WHERE time >= now() - INTERVAL '30 days'
GROUP BY bucket
ORDER BY bucket DESC;

-- Per-week unique endpoints
SELECT time_bucket('1 week', time) AS bucket,
       COUNT(DISTINCT endpoint)    AS unique_endpoints
FROM api_metrics
GROUP BY bucket
ORDER BY bucket DESC;
```

**Go query for hourly request count:**

```go
// internal/repository/metrics_repository.go
package repository

import (
    "context"
    "fmt"
    "time"

    "github.com/jmoiron/sqlx"
)

type HourlyStat struct {
    Bucket   time.Time `db:"bucket"`
    Requests int64     `db:"requests"`
    AvgMs    float64   `db:"avg_ms"`
    Errors   int64     `db:"errors"`
}

func (r *MetricsRepository) GetHourlyStats(
    ctx context.Context,
    start, end time.Time,
) ([]HourlyStat, error) {
    const q = `
        SELECT time_bucket('1 hour', time) AS bucket,
               COUNT(*)                    AS requests,
               AVG(duration_ms)            AS avg_ms,
               COUNT(*) FILTER (WHERE status >= 500) AS errors
        FROM api_metrics
        WHERE time >= $1 AND time < $2
        GROUP BY bucket
        ORDER BY bucket DESC
    `
    var rows []HourlyStat
    if err := r.db.SelectContext(ctx, &rows, q, start, end); err != nil {
        return nil, fmt.Errorf("GetHourlyStats: %w", err)
    }
    return rows, nil
}
```

---

## Continuous Aggregates

A continuous aggregate is a materialized view backed by TimescaleDB that automatically refreshes as new data arrives. Unlike plain `MATERIALIZED VIEW`, it only reprocesses new/changed chunks — making refresh efficient even on large hypertables.

**Create a continuous aggregate:**

```sql
-- Hourly API metrics — refreshed automatically
CREATE MATERIALIZED VIEW api_metrics_hourly
WITH (timescaledb.continuous) AS
    SELECT time_bucket('1 hour', time)                         AS bucket,
           endpoint,
           method,
           COUNT(*)                                            AS requests,
           AVG(duration_ms)                                    AS avg_ms,
           PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY duration_ms) AS p95_ms,
           COUNT(*) FILTER (WHERE status >= 500)               AS errors,
           COUNT(*) FILTER (WHERE status >= 400)               AS client_errors
    FROM api_metrics
    GROUP BY bucket, endpoint, method
WITH NO DATA;  -- populate via policy, not immediately
```

**Refresh policy** — tells TimescaleDB how often to update and how far back:

```sql
-- Refresh every hour; keep the last 30 days materialized; allow 1-hour lag
SELECT add_continuous_aggregate_policy('api_metrics_hourly',
    start_offset => INTERVAL '30 days',
    end_offset   => INTERVAL '1 hour',  -- recent 1h uses real-time fallback
    schedule_interval => INTERVAL '1 hour'
);
```

**Real-time aggregates** (default `ON`) — for the gap between `end_offset` and `now()`, TimescaleDB automatically combines the materialized data with a live query on the raw hypertable. No code change needed.

**Disable real-time if you prefer strict materialized data:**

```sql
ALTER MATERIALIZED VIEW api_metrics_hourly
    SET (timescaledb.materialized_only = true);
```

**Manual refresh (e.g., backfill after schema change):**

```sql
CALL refresh_continuous_aggregate('api_metrics_hourly',
    '2026-01-01'::TIMESTAMPTZ,
    '2026-02-01'::TIMESTAMPTZ);
```

**Go query against continuous aggregate:**

```go
// internal/repository/metrics_repository.go

type EndpointStat struct {
    Bucket       time.Time `db:"bucket"`
    Endpoint     string    `db:"endpoint"`
    Method       string    `db:"method"`
    Requests     int64     `db:"requests"`
    AvgMs        float64   `db:"avg_ms"`
    P95Ms        float64   `db:"p95_ms"`
    Errors       int64     `db:"errors"`
}

func (r *MetricsRepository) GetEndpointStats(
    ctx context.Context,
    endpoint string,
    start, end time.Time,
) ([]EndpointStat, error) {
    const q = `
        SELECT bucket, endpoint, method, requests, avg_ms, p95_ms, errors
        FROM api_metrics_hourly
        WHERE endpoint = $1
          AND bucket >= $2 AND bucket < $3
        ORDER BY bucket DESC
    `
    var rows []EndpointStat
    if err := r.db.SelectContext(ctx, &rows, q, endpoint, start, end); err != nil {
        return nil, fmt.Errorf("GetEndpointStats: %w", err)
    }
    return rows, nil
}
```

Querying `api_metrics_hourly` is identical to querying any table — no TimescaleDB-specific Go code needed.

---

## Compression

TimescaleDB compresses old chunks in columnar format, achieving 10–20x size reduction on typical time-series workloads. Compressed chunks are still queryable (transparent decompression), but cannot be updated or inserted into directly.

**Enable compression on a hypertable:**

```sql
ALTER TABLE api_metrics SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'endpoint, method',  -- group related rows
    timescaledb.compress_orderby   = 'time DESC'          -- sort within segment
);
```

- `compress_segmentby`: columns whose values repeat across rows (high reuse = better compression). Good candidates: endpoint, device_id, metric name, status.
- `compress_orderby`: determines sort order within a compressed segment. Use the primary time column descending.

**Add a compression policy** — automatically compresses chunks older than a threshold:

```sql
-- Compress chunks older than 7 days
SELECT add_compression_policy('api_metrics', INTERVAL '7 days');
```

**Check compression status:**

```sql
SELECT chunk_name,
       before_compression_total_bytes,
       after_compression_total_bytes,
       ROUND(100 - 100.0 * after_compression_total_bytes
             / NULLIF(before_compression_total_bytes, 0), 1) AS savings_pct
FROM chunk_compression_stats('api_metrics')
ORDER BY chunk_name;
```

**Querying compressed data** is transparent — no SQL changes needed:

```sql
-- Works on both compressed and uncompressed chunks
SELECT time_bucket('1 hour', time), COUNT(*)
FROM api_metrics
WHERE time >= now() - INTERVAL '30 days'
GROUP BY 1;
```

**Decompressing for updates** (manual, only when mutations are needed):

```sql
-- Decompress a specific chunk by name
SELECT decompress_chunk('_timescaledb_internal._hyper_1_5_chunk');

-- Or decompress all chunks in a range
SELECT decompress_chunk(c) FROM show_chunks('api_metrics',
    older_than => now() - INTERVAL '8 days',
    newer_than => now() - INTERVAL '10 days') c;
```

**Design rule:** If you need to UPDATE or DELETE individual rows frequently, do not compress that data. Compress only immutable historical data (IoT readings, request logs, event streams).

---

## Retention Policies

Retention policies automatically drop old chunks. Dropping a chunk deletes an entire partition file — it is orders of magnitude faster than `DELETE FROM ... WHERE time < threshold` on a non-partitioned table.

**Add a retention policy:**

```sql
-- Drop raw data older than 90 days
SELECT add_retention_policy('api_metrics', INTERVAL '90 days');
```

**Remove a policy:**

```sql
SELECT remove_retention_policy('api_metrics');
```

**How it works:** TimescaleDB's background worker runs the policy on `schedule_interval` (default: 1 day). Eligible chunks (all data older than `drop_after`) are dropped atomically. Ongoing queries on other chunks are unaffected.

**Combining retention with continuous aggregates — the standard pattern:**

```sql
-- Raw data: keep 30 days
SELECT add_retention_policy('api_metrics', INTERVAL '30 days');

-- Hourly aggregates: keep 1 year
SELECT add_retention_policy('api_metrics_hourly', INTERVAL '365 days');

-- (Optional) daily aggregates view for long-term trend: keep forever — no retention policy
```

This pattern gives you:
- Full-resolution data for recent incident investigation (30 days)
- Hour-granularity analytics for dashboards (1 year)
- Day-granularity trends indefinitely

**Manual chunk drop** (for immediate cleanup, skipping the policy schedule):

```sql
SELECT drop_chunks('api_metrics', older_than => INTERVAL '90 days');
```

---

## Downsampling

Downsampling is the pattern of moving from high-resolution raw data to progressively lower-resolution aggregates, retaining each level for a different duration.

**Pattern: three-tier hierarchy**

```
1-second raw  →  retained 7 days
     ↓  (continuous aggregate)
1-minute agg  →  retained 90 days
     ↓  (continuous aggregate of agg)
1-hour agg    →  retained forever
```

**Implementation:**

```sql
-- Tier 1: raw hypertable (1-second sensor data, 7-day retention)
CREATE TABLE sensor_readings (
    time      TIMESTAMPTZ    NOT NULL,
    sensor_id TEXT           NOT NULL,
    value     DOUBLE PRECISION NOT NULL
);
SELECT create_hypertable('sensor_readings', by_range('time', INTERVAL '1 day'));
SELECT add_retention_policy('sensor_readings', INTERVAL '7 days');

-- Tier 2: 1-minute aggregate (90-day retention)
CREATE MATERIALIZED VIEW sensor_readings_1min
WITH (timescaledb.continuous) AS
    SELECT time_bucket('1 minute', time) AS bucket,
           sensor_id,
           AVG(value)   AS avg_value,
           MIN(value)   AS min_value,
           MAX(value)   AS max_value,
           COUNT(*)     AS samples
    FROM sensor_readings
    GROUP BY bucket, sensor_id
WITH NO DATA;

SELECT add_continuous_aggregate_policy('sensor_readings_1min',
    start_offset      => INTERVAL '7 days',
    end_offset        => INTERVAL '1 minute',
    schedule_interval => INTERVAL '1 minute');

SELECT add_retention_policy('sensor_readings_1min', INTERVAL '90 days');

-- Tier 3: 1-hour aggregate (no retention — kept forever)
CREATE MATERIALIZED VIEW sensor_readings_1hour
WITH (timescaledb.continuous) AS
    SELECT time_bucket('1 hour', bucket) AS bucket,
           sensor_id,
           AVG(avg_value) AS avg_value,
           MIN(min_value) AS min_value,
           MAX(max_value) AS max_value,
           SUM(samples)   AS samples
    FROM sensor_readings_1min
    GROUP BY time_bucket('1 hour', bucket), sensor_id
WITH NO DATA;

SELECT add_continuous_aggregate_policy('sensor_readings_1hour',
    start_offset      => INTERVAL '90 days',
    end_offset        => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');
```

**Query routing by time range** (done in Go, selects the right tier):

```go
func (r *SensorRepository) QueryReadings(
    ctx context.Context,
    sensorID string,
    start, end time.Time,
) (string, []any) {
    age := time.Since(start)
    switch {
    case age <= 7*24*time.Hour:
        return `SELECT time AS bucket, value AS avg_value FROM sensor_readings
                WHERE sensor_id=$1 AND time>=$2 AND time<$3 ORDER BY bucket`, nil
    case age <= 90*24*time.Hour:
        return `SELECT bucket, avg_value FROM sensor_readings_1min
                WHERE sensor_id=$1 AND bucket>=$2 AND bucket<$3 ORDER BY bucket`, nil
    default:
        return `SELECT bucket, avg_value FROM sensor_readings_1hour
                WHERE sensor_id=$1 AND bucket>=$2 AND bucket<$3 ORDER BY bucket`, nil
    }
}
```

---

## Go Integration

### APIMetric struct

```go
// internal/domain/metric.go
package domain

import "time"

type APIMetric struct {
    Time       time.Time `db:"time"`
    Endpoint   string    `db:"endpoint"`
    Method     string    `db:"method"`
    Status     int       `db:"status"`
    DurationMs float64   `db:"duration_ms"`
    UserID     string    `db:"user_id"` // empty string → NULL via pointer pattern below
    IP         string    `db:"ip"`
}
```

### Batch insert pattern

Writing every request synchronously would serialize inserts and add latency. Instead, collect metrics in a buffered channel and flush in batches every few seconds.

```go
// internal/repository/metrics_writer.go
package repository

import (
    "context"
    "fmt"
    "log/slog"
    "time"

    "github.com/jmoiron/sqlx"
    "myapp/internal/domain"
)

const batchSize = 500

type MetricsBatchWriter struct {
    db     *sqlx.DB
    ch     chan domain.APIMetric
    logger *slog.Logger
}

func NewMetricsBatchWriter(db *sqlx.DB, logger *slog.Logger) *MetricsBatchWriter {
    w := &MetricsBatchWriter{
        db:     db,
        ch:     make(chan domain.APIMetric, 10_000), // buffer 10k metrics
        logger: logger,
    }
    return w
}

// Enqueue is non-blocking; drops metric if buffer is full.
func (w *MetricsBatchWriter) Enqueue(m domain.APIMetric) {
    select {
    case w.ch <- m:
    default:
        w.logger.Warn("metrics buffer full, dropping metric",
            slog.String("endpoint", m.Endpoint))
    }
}

// Run starts the flush loop. Call as a goroutine; cancel ctx to stop.
func (w *MetricsBatchWriter) Run(ctx context.Context) {
    ticker := time.NewTicker(5 * time.Second)
    defer ticker.Stop()

    batch := make([]domain.APIMetric, 0, batchSize)

    flush := func() {
        if len(batch) == 0 {
            return
        }
        if err := w.insertBatch(ctx, batch); err != nil {
            w.logger.Error("metrics flush failed", slog.Any("error", err))
        }
        batch = batch[:0]
    }

    for {
        select {
        case m := <-w.ch:
            batch = append(batch, m)
            if len(batch) >= batchSize {
                flush()
            }
        case <-ticker.C:
            flush()
        case <-ctx.Done():
            // Drain remaining metrics before shutdown
            for len(w.ch) > 0 {
                batch = append(batch, <-w.ch)
            }
            flush()
            return
        }
    }
}

func (w *MetricsBatchWriter) insertBatch(ctx context.Context, batch []domain.APIMetric) error {
    if len(batch) == 0 {
        return nil
    }
    // Build multi-row INSERT
    const q = `
        INSERT INTO api_metrics (time, endpoint, method, status, duration_ms, user_id, ip)
        VALUES (:time, :endpoint, :method, :status, :duration_ms, :user_id, :ip)
    `
    _, err := w.db.NamedExecContext(ctx, q, batch)
    if err != nil {
        return fmt.Errorf("insertBatch: %w", err)
    }
    return nil
}
```

### MetricsRepository

```go
// internal/repository/metrics_repository.go
package repository

import (
    "context"
    "fmt"
    "time"

    "github.com/jmoiron/sqlx"
)

type MetricsRepository struct {
    db *sqlx.DB
}

func NewMetricsRepository(db *sqlx.DB) *MetricsRepository {
    return &MetricsRepository{db: db}
}

// GetHourlyStats returns per-hour aggregates from the continuous aggregate view.
func (r *MetricsRepository) GetHourlyStats(
    ctx context.Context,
    start, end time.Time,
) ([]HourlyStat, error) {
    const q = `
        SELECT bucket, endpoint, method, requests, avg_ms, p95_ms, errors
        FROM api_metrics_hourly
        WHERE bucket >= $1 AND bucket < $2
        ORDER BY bucket DESC, requests DESC
        LIMIT 1000
    `
    var rows []HourlyStat
    if err := r.db.SelectContext(ctx, &rows, q, start, end); err != nil {
        return nil, fmt.Errorf("GetHourlyStats: %w", err)
    }
    return rows, nil
}

// GetEndpointStats returns hourly stats for a specific endpoint.
func (r *MetricsRepository) GetEndpointStats(
    ctx context.Context,
    endpoint string,
    start, end time.Time,
) ([]EndpointStat, error) {
    const q = `
        SELECT bucket, endpoint, method, requests, avg_ms, p95_ms, errors
        FROM api_metrics_hourly
        WHERE endpoint = $1
          AND bucket >= $2 AND bucket < $3
        ORDER BY bucket DESC
    `
    var rows []EndpointStat
    if err := r.db.SelectContext(ctx, &rows, q, endpoint, start, end); err != nil {
        return nil, fmt.Errorf("GetEndpointStats: %w", err)
    }
    return rows, nil
}
```

---

## API Metrics Example (Complete)

A full end-to-end example: DDL, Gin middleware, batch writer goroutine, and stats handler.

### DDL

```sql
-- migrations/000002_api_metrics.sql

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

-- Enable compression on chunks older than 7 days
ALTER TABLE api_metrics SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'endpoint, method',
    timescaledb.compress_orderby   = 'time DESC'
);
SELECT add_compression_policy('api_metrics', INTERVAL '7 days');

-- Retain raw data for 30 days
SELECT add_retention_policy('api_metrics', INTERVAL '30 days');

-- Continuous aggregate: hourly stats per endpoint+method
CREATE MATERIALIZED VIEW api_metrics_hourly
WITH (timescaledb.continuous) AS
    SELECT time_bucket('1 hour', time)                               AS bucket,
           endpoint,
           method,
           COUNT(*)                                                  AS requests,
           AVG(duration_ms)                                          AS avg_ms,
           PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY duration_ms) AS p95_ms,
           COUNT(*) FILTER (WHERE status >= 500)                     AS errors,
           COUNT(*) FILTER (WHERE status >= 400)                     AS client_errors
    FROM api_metrics
    GROUP BY bucket, endpoint, method
WITH NO DATA;

SELECT add_continuous_aggregate_policy('api_metrics_hourly',
    start_offset      => INTERVAL '30 days',
    end_offset        => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');

-- Retain hourly aggregates for 1 year
SELECT add_retention_policy('api_metrics_hourly', INTERVAL '365 days');
```

### Gin middleware

```go
// internal/middleware/metrics_middleware.go
package middleware

import (
    "time"

    "github.com/gin-gonic/gin"
    "myapp/internal/domain"
    "myapp/internal/repository"
)

// MetricsRecorder returns a Gin middleware that enqueues an APIMetric
// after each request completes. Non-blocking: uses the batch writer channel.
func MetricsRecorder(writer *repository.MetricsBatchWriter) gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()
        c.Next() // execute handler chain

        writer.Enqueue(domain.APIMetric{
            Time:       start,
            Endpoint:   c.FullPath(), // registered route pattern, e.g. /api/v1/users/:id
            Method:     c.Request.Method,
            Status:     c.Writer.Status(),
            DurationMs: float64(time.Since(start).Microseconds()) / 1000.0,
            UserID:     c.GetString("user_id"), // set by auth middleware
            IP:         c.ClientIP(),
        })
    }
}
```

### Wiring in main.go

```go
// cmd/api/main.go (relevant snippet)
package main

import (
    "context"
    "log/slog"

    "github.com/gin-gonic/gin"
    "myapp/internal/middleware"
    "myapp/internal/repository"
)

func main() {
    logger := slog.Default()
    db := connectDB() // returns *sqlx.DB

    metricsWriter := repository.NewMetricsBatchWriter(db, logger)
    metricsRepo   := repository.NewMetricsRepository(db)

    // Start background flush goroutine
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    go metricsWriter.Run(ctx)

    r := gin.New()
    r.Use(middleware.MetricsRecorder(metricsWriter)) // after recovery, before routes

    // Stats endpoint
    statsHandler := handler.NewStatsHandler(metricsRepo)
    r.GET("/api/v1/stats/hourly",  statsHandler.Hourly)
    r.GET("/api/v1/stats/endpoint", statsHandler.Endpoint)

    r.Run(":8080")
}
```

### Stats handler

```go
// internal/handler/stats_handler.go
package handler

import (
    "net/http"
    "time"

    "github.com/gin-gonic/gin"
    "myapp/internal/repository"
)

type StatsHandler struct {
    repo *repository.MetricsRepository
}

func NewStatsHandler(repo *repository.MetricsRepository) *StatsHandler {
    return &StatsHandler{repo: repo}
}

// GET /api/v1/stats/hourly?from=2026-03-01T00:00:00Z&to=2026-03-04T00:00:00Z
func (h *StatsHandler) Hourly(c *gin.Context) {
    start, end, ok := parseTimeRange(c)
    if !ok {
        return
    }
    stats, err := h.repo.GetHourlyStats(c.Request.Context(), start, end)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
        return
    }
    c.JSON(http.StatusOK, gin.H{"data": stats, "count": len(stats)})
}

// GET /api/v1/stats/endpoint?endpoint=/api/v1/users&from=...&to=...
func (h *StatsHandler) Endpoint(c *gin.Context) {
    endpoint := c.Query("endpoint")
    if endpoint == "" {
        c.JSON(http.StatusBadRequest, gin.H{"error": "endpoint required"})
        return
    }
    start, end, ok := parseTimeRange(c)
    if !ok {
        return
    }
    stats, err := h.repo.GetEndpointStats(c.Request.Context(), endpoint, start, end)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
        return
    }
    c.JSON(http.StatusOK, gin.H{"data": stats, "count": len(stats)})
}

func parseTimeRange(c *gin.Context) (start, end time.Time, ok bool) {
    fromStr := c.Query("from")
    toStr   := c.Query("to")
    if fromStr == "" || toStr == "" {
        c.JSON(http.StatusBadRequest, gin.H{"error": "from and to required (RFC3339)"})
        return time.Time{}, time.Time{}, false
    }
    var err error
    start, err = time.Parse(time.RFC3339, fromStr)
    if err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "invalid from: use RFC3339"})
        return time.Time{}, time.Time{}, false
    }
    end, err = time.Parse(time.RFC3339, toStr)
    if err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "invalid to: use RFC3339"})
        return time.Time{}, time.Time{}, false
    }
    if end.Before(start) {
        c.JSON(http.StatusBadRequest, gin.H{"error": "to must be after from"})
        return time.Time{}, time.Time{}, false
    }
    return start, end, true
}
```

### Dashboard queries

```sql
-- Top 10 slowest endpoints (last 24 hours)
SELECT endpoint, method,
       ROUND(AVG(avg_ms)::numeric, 2)  AS avg_ms,
       ROUND(MAX(p95_ms)::numeric, 2)  AS peak_p95_ms,
       SUM(requests)                   AS total_requests
FROM api_metrics_hourly
WHERE bucket >= now() - INTERVAL '24 hours'
GROUP BY endpoint, method
ORDER BY avg_ms DESC
LIMIT 10;

-- Error rate by endpoint (last 7 days)
SELECT endpoint,
       SUM(requests)                                   AS total,
       SUM(errors)                                     AS total_errors,
       ROUND(100.0 * SUM(errors) / NULLIF(SUM(requests), 0), 2) AS error_pct
FROM api_metrics_hourly
WHERE bucket >= now() - INTERVAL '7 days'
GROUP BY endpoint
HAVING SUM(requests) > 100
ORDER BY error_pct DESC;

-- Request volume trend (last 30 days, daily rollup)
SELECT time_bucket('1 day', bucket) AS day,
       SUM(requests)                AS requests,
       ROUND(AVG(avg_ms)::numeric, 1) AS avg_ms
FROM api_metrics_hourly
WHERE bucket >= now() - INTERVAL '30 days'
GROUP BY day
ORDER BY day;
```
