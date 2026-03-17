# TimescaleDB — Go Integration and Complete Example

## Batch Writer

Writing every request synchronously adds latency. Collect metrics in a buffered channel and flush in batches every few seconds.

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
    return &MetricsBatchWriter{
        db:     db,
        ch:     make(chan domain.APIMetric, 10_000),
        logger: logger,
    }
}

// Enqueue is non-blocking; drops metric if buffer is full.
func (w *MetricsBatchWriter) Enqueue(m domain.APIMetric) {
    select {
    case w.ch <- m:
    default:
        w.logger.Warn("metrics buffer full, dropping metric", slog.String("endpoint", m.Endpoint))
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
            for len(w.ch) > 0 {
                batch = append(batch, <-w.ch)
            }
            flush()
            return
        }
    }
}

func (w *MetricsBatchWriter) insertBatch(ctx context.Context, batch []domain.APIMetric) error {
    const q = `INSERT INTO api_metrics (time, endpoint, method, status, duration_ms, user_id, ip)
               VALUES (:time, :endpoint, :method, :status, :duration_ms, :user_id, :ip)`
    _, err := w.db.NamedExecContext(ctx, q, batch)
    if err != nil {
        return fmt.Errorf("insertBatch: %w", err)
    }
    return nil
}
```

## Gin Middleware and Wiring

```go
// Middleware: record start time, call c.Next(), enqueue APIMetric{Time, Endpoint: c.FullPath(),
// Method, Status: c.Writer.Status(), DurationMs, UserID: c.GetString("user_id"), IP: c.ClientIP()}

// main.go wiring:
metricsWriter := repository.NewMetricsBatchWriter(db, logger)
ctx, cancel := context.WithCancel(context.Background())
defer cancel()
go metricsWriter.Run(ctx)
r.Use(middleware.MetricsRecorder(metricsWriter))
```

## Dashboard Queries

```sql
-- Top 10 slowest endpoints (last 24 hours)
SELECT endpoint, method,
       ROUND(AVG(avg_ms)::numeric, 2) AS avg_ms,
       SUM(requests) AS total_requests
FROM api_metrics_hourly
WHERE bucket >= now() - INTERVAL '24 hours'
GROUP BY endpoint, method ORDER BY avg_ms DESC LIMIT 10;

-- Error rate by endpoint (last 7 days)
SELECT endpoint,
       SUM(requests) AS total,
       ROUND(100.0 * SUM(errors) / NULLIF(SUM(requests), 0), 2) AS error_pct
FROM api_metrics_hourly
WHERE bucket >= now() - INTERVAL '7 days'
GROUP BY endpoint HAVING SUM(requests) > 100
ORDER BY error_pct DESC;
```

> For complete DDL (hypertable + compression + retention + continuous aggregate): see [timescaledb-aggregates-compression-retention.md](timescaledb-aggregates-compression-retention.md).
