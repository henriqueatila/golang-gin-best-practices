# TimescaleDB — Continuous Aggregates, Compression, and Retention

## Continuous Aggregates

A continuous aggregate is a materialized view that automatically refreshes as new data arrives. It only reprocesses new/changed chunks — efficient even on large hypertables.

```sql
CREATE MATERIALIZED VIEW api_metrics_hourly
WITH (timescaledb.continuous) AS
    SELECT time_bucket('1 hour', time)                              AS bucket,
           endpoint, method,
           COUNT(*)                                                 AS requests,
           AVG(duration_ms)                                         AS avg_ms,
           PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY duration_ms) AS p95_ms,
           COUNT(*) FILTER (WHERE status >= 500)                    AS errors,
           COUNT(*) FILTER (WHERE status >= 400)                    AS client_errors
    FROM api_metrics
    GROUP BY bucket, endpoint, method
WITH NO DATA;

-- Refresh policy: update every hour, keep last 30 days materialized
SELECT add_continuous_aggregate_policy('api_metrics_hourly',
    start_offset      => INTERVAL '30 days',
    end_offset        => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour'
);

-- Manual backfill after schema change
CALL refresh_continuous_aggregate('api_metrics_hourly',
    '2026-01-01'::TIMESTAMPTZ, '2026-02-01'::TIMESTAMPTZ);
```

Real-time aggregates (default ON) — for the gap between `end_offset` and `now()`, TimescaleDB automatically combines materialized data with a live query. No code change needed.

Go query against continuous aggregate (identical to any table query):

```go
func (r *MetricsRepository) GetEndpointStats(ctx context.Context, endpoint string, start, end time.Time) ([]EndpointStat, error) {
    const q = `
        SELECT bucket, endpoint, method, requests, avg_ms, p95_ms, errors
        FROM api_metrics_hourly
        WHERE endpoint = $1 AND bucket >= $2 AND bucket < $3
        ORDER BY bucket DESC`
    var rows []EndpointStat
    if err := r.db.SelectContext(ctx, &rows, q, endpoint, start, end); err != nil {
        return nil, fmt.Errorf("GetEndpointStats: %w", err)
    }
    return rows, nil
}
```

---

## Compression

Compresses old chunks in columnar format — 10–20x size reduction on typical time-series workloads. Compressed chunks are still queryable but cannot be updated directly.

```sql
ALTER TABLE api_metrics SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'endpoint, method',  -- group related rows
    timescaledb.compress_orderby   = 'time DESC'
);

-- Automatically compress chunks older than 7 days
SELECT add_compression_policy('api_metrics', INTERVAL '7 days');

-- Check compression savings
SELECT chunk_name,
       before_compression_total_bytes,
       after_compression_total_bytes,
       ROUND(100 - 100.0 * after_compression_total_bytes /
             NULLIF(before_compression_total_bytes, 0), 1) AS savings_pct
FROM chunk_compression_stats('api_metrics')
ORDER BY chunk_name;
```

Querying compressed data is transparent — no SQL changes needed. To update compressed data, decompress first:

```sql
SELECT decompress_chunk('_timescaledb_internal._hyper_1_5_chunk');
```

**Design rule:** Only compress immutable historical data (IoT readings, request logs, event streams). Do not compress data you update frequently.

---

## Retention Policies

Drops old chunks atomically — orders of magnitude faster than `DELETE FROM ... WHERE time < threshold`.

```sql
-- Drop raw data older than 90 days
SELECT add_retention_policy('api_metrics', INTERVAL '90 days');
SELECT remove_retention_policy('api_metrics');

-- Manual immediate drop (skipping the policy schedule)
SELECT drop_chunks('api_metrics', older_than => INTERVAL '90 days');
```

**Standard pattern — combine retention with continuous aggregates:**

```sql
SELECT add_retention_policy('api_metrics',         INTERVAL '30 days');   -- raw
SELECT add_retention_policy('api_metrics_hourly',  INTERVAL '365 days');  -- hourly agg
-- daily aggregates: no retention policy — kept forever
```

This gives: full-resolution data for incident investigation (30 days), hour-granularity analytics (1 year), day-granularity trends forever.

---

## Downsampling

Three-tier hierarchy for high-frequency sensor data:

```sql
-- Tier 1: raw 1-second data, 7-day retention
CREATE TABLE sensor_readings (
    time TIMESTAMPTZ NOT NULL, sensor_id TEXT NOT NULL, value DOUBLE PRECISION NOT NULL
);
SELECT create_hypertable('sensor_readings', by_range('time', INTERVAL '1 day'));
SELECT add_retention_policy('sensor_readings', INTERVAL '7 days');

-- Tier 2: 1-minute aggregate, 90-day retention
CREATE MATERIALIZED VIEW sensor_readings_1min WITH (timescaledb.continuous) AS
    SELECT time_bucket('1 minute', time) AS bucket, sensor_id,
           AVG(value) AS avg_value, MIN(value) AS min_value,
           MAX(value) AS max_value, COUNT(*) AS samples
    FROM sensor_readings GROUP BY bucket, sensor_id WITH NO DATA;

SELECT add_continuous_aggregate_policy('sensor_readings_1min',
    start_offset => INTERVAL '7 days', end_offset => INTERVAL '1 minute',
    schedule_interval => INTERVAL '1 minute');
SELECT add_retention_policy('sensor_readings_1min', INTERVAL '90 days');

-- Tier 3: 1-hour aggregate, no retention (kept forever)
CREATE MATERIALIZED VIEW sensor_readings_1hour WITH (timescaledb.continuous) AS
    SELECT time_bucket('1 hour', bucket) AS bucket, sensor_id,
           AVG(avg_value) AS avg_value, MIN(min_value) AS min_value,
           MAX(max_value) AS max_value, SUM(samples) AS samples
    FROM sensor_readings_1min GROUP BY time_bucket('1 hour', bucket), sensor_id
WITH NO DATA;
```

In Go, route queries by `time.Since(start)`: ≤7 days → `sensor_readings`, ≤90 days → `sensor_readings_1min`, older → `sensor_readings_1hour`.
