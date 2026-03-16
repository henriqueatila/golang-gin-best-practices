# Background Jobs

Background processing patterns for Go Gin APIs — goroutines with `c.Copy()`, worker pools, DB-backed queues, external queues, and graceful shutdown.

## Common Use Cases

| Use Case | Recommended Pattern |
|----------|-------------------|
| Send email after signup | Simple goroutine |
| Generate PDF report | Worker pool |
| Process payment webhook | DB-backed queue |
| Sync data with external API | External queue (asynq) |

## Simple Goroutine Pattern

Use `c.Copy()` before passing `*gin.Context` to a goroutine. The original context is returned to the pool when the request ends — using it after that causes a data race.

```go
// internal/handler/user_handler.go
func (h *UserHandler) Create(c *gin.Context) {
    var req CreateUserRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "invalid request"})
        return
    }

    user, err := h.userSvc.Create(c.Request.Context(), req)
    if err != nil {
        h.logger.Error("create user failed", "err", err)
        c.JSON(http.StatusInternalServerError, gin.H{"error": "could not create user"})
        return
    }

    // c.Copy() returns a shallow copy safe to use outside the request lifecycle
    cCopy := c.Copy()
    go func() {
        if err := h.emailSvc.SendWelcome(cCopy.Request.Context(), user); err != nil {
            h.logger.Error("welcome email failed", "userID", user.ID, "err", err)
        }
    }()

    c.JSON(http.StatusCreated, user)
}
```

**Warning:** fire-and-forget goroutines have no retry, no persistence, and are lost on process crash. Use only for non-critical side effects.

## Worker Pool Pattern

A buffered channel acts as a bounded job queue. Prevents unbounded goroutine spawning under load.

```go
// internal/worker/worker.go
type Job func(ctx context.Context)

type Worker struct {
    jobs   chan Job
    logger *slog.Logger
}

func NewWorker(bufferSize int, logger *slog.Logger) *Worker {
    return &Worker{
        jobs:   make(chan Job, bufferSize),
        logger: logger,
    }
}

// Submit enqueues a job. Returns error if the queue is full (non-blocking).
func (w *Worker) Submit(job Job) error {
    select {
    case w.jobs <- job:
        return nil
    default:
        return fmt.Errorf("worker queue full")
    }
}

// Start processes jobs until ctx is cancelled. Run in a goroutine.
func (w *Worker) Start(ctx context.Context) {
    for {
        select {
        case job := <-w.jobs:
            job(ctx)
        case <-ctx.Done():
            // Drain remaining jobs before exit
            for {
                select {
                case job := <-w.jobs:
                    job(ctx)
                default:
                    w.logger.Info("worker pool stopped")
                    return
                }
            }
        }
    }
}
```

Usage in a handler — submit without blocking the request:

```go
// internal/handler/report_handler.go
func (h *ReportHandler) Generate(c *gin.Context) {
    var req GenerateReportRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "invalid request"})
        return
    }

    reportID := uuid.NewString()

    if err := h.worker.Submit(func(ctx context.Context) {
        if err := h.reportSvc.Generate(ctx, reportID, req); err != nil {
            h.logger.Error("report generation failed", "reportID", reportID, "err", err)
        }
    }); err != nil {
        c.JSON(http.StatusServiceUnavailable, gin.H{"error": "server busy, try again later"})
        return
    }

    c.JSON(http.StatusAccepted, gin.H{"report_id": reportID})
}
```

## Database-Backed Job Queue

For persistence across restarts and guaranteed at-least-once processing.

**Jobs table (PostgreSQL):**

```sql
CREATE TABLE jobs (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    type       TEXT NOT NULL,
    payload    JSONB NOT NULL,
    status     TEXT NOT NULL DEFAULT 'pending',  -- pending | running | done | failed
    attempts   INT NOT NULL DEFAULT 0,
    max_attempts INT NOT NULL DEFAULT 3,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    locked_at  TIMESTAMPTZ
);

CREATE INDEX ON jobs (status, locked_at) WHERE status = 'pending';
```

**Polling worker — claims with atomic UPDATE to prevent double-processing:**

```go
// internal/worker/db_worker.go
func (w *DBWorker) claimNext(ctx context.Context) (*Job, error) {
    var job Job
    err := w.db.QueryRowContext(ctx, `
        UPDATE jobs SET status = 'running', locked_at = now(), attempts = attempts + 1
        WHERE id = (
            SELECT id FROM jobs
            WHERE status = 'pending'
              AND attempts < max_attempts
              AND locked_at IS NULL
            ORDER BY created_at
            FOR UPDATE SKIP LOCKED
            LIMIT 1
        )
        RETURNING id, type, payload, attempts
    `).Scan(&job.ID, &job.Type, &job.Payload, &job.Attempts)
    if errors.Is(err, sql.ErrNoRows) {
        return nil, nil // no work available
    }
    return &job, err
}

func (w *DBWorker) Start(ctx context.Context) {
    ticker := time.NewTicker(5 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-ticker.C:
            job, err := w.claimNext(ctx)
            if err != nil {
                w.logger.Error("claim job failed", "err", err)
                continue
            }
            if job == nil {
                continue
            }
            go w.process(ctx, job)
        case <-ctx.Done():
            return
        }
    }
}
```

Retry with exponential backoff: on failure set `status = 'pending'`, `locked_at = NULL`. After `max_attempts` set `status = 'failed'` (dead letter).

## External Queue (Brief)

Use when you need multiple workers, guaranteed delivery, or complex retry logic.

**`github.com/hibiken/asynq`** — Redis-backed task queue, drop-in for most use cases:

```go
// Enqueue (in handler)
client := asynq.NewClient(asynq.RedisClientOpt{Addr: redisAddr})
task := asynq.NewTask("email:welcome", payload)
_, err = client.EnqueueContext(c.Request.Context(), task)

// Worker (in main.go)
srv := asynq.NewServer(asynq.RedisClientOpt{Addr: redisAddr}, asynq.Config{Concurrency: 10})
mux := asynq.NewServeMux()
mux.HandleFunc("email:welcome", emailHandler.ProcessWelcome)
srv.Run(mux)
```

Use NATS or RabbitMQ when you need fan-out, pub/sub, or cross-service messaging.

## Graceful Shutdown

Stop accepting new jobs and drain in-flight work before process exit. Ties to the graceful shutdown pattern in **golang-gin-deploy**.

```go
// cmd/api/main.go
ctx, cancel := context.WithCancel(context.Background())

go worker.Start(ctx) // start pool or DB poller

quit := make(chan os.Signal, 1)
signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
<-quit

slog.Info("shutting down, draining jobs...")
cancel()                          // signals worker to stop and drain
time.Sleep(5 * time.Second)       // give in-flight jobs time to finish
slog.Info("shutdown complete")
```

For the DB-backed worker, the drain loop finishes processing claimed jobs even after `ctx` is cancelled before releasing the process.
