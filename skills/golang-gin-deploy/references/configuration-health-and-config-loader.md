# Configuration — Health Check Handler and 12-Factor Config Loader

## Health Check Endpoint

Expose `/health` for container orchestrators and K8s probes.

```go
// internal/handler/health_handler.go
type HealthHandler struct {
    db     DBPinger
    logger *slog.Logger
}

// DBPinger is satisfied by *sql.DB and *sqlx.DB
type DBPinger interface { PingContext(ctx context.Context) error }

func NewHealthHandler(db DBPinger, logger *slog.Logger) *HealthHandler {
    return &HealthHandler{db: db, logger: logger}
}

func (h *HealthHandler) Check(c *gin.Context) {
    ctx, cancel := context.WithTimeout(c.Request.Context(), 2*time.Second)
    defer cancel()
    status, httpStatus := "ok", http.StatusOK
    if err := h.db.PingContext(ctx); err != nil {
        h.logger.Error("health: db ping failed", "error", err)
        status, httpStatus = "degraded", http.StatusServiceUnavailable
    }
    c.JSON(httpStatus, gin.H{"status": status})
}
```

Register: `r.GET("/health", handler.NewHealthHandler(db, logger).Check)`.

**Readiness vs Liveness:**

| Probe | Checks | On failure |
|-------|--------|------------|
| Liveness | Is the process alive? | Restart container |
| Readiness | Can the app serve traffic? | Remove from load balancer |

Use one `/health` (DB ping) for both. Split into `/health/live` (no DB check) and `/health/ready` (DB check) only when startup time causes false liveness failures.

---

## Configuration via Environment Variables (12-Factor)

```go
// internal/config/config.go
type Config struct {
    Port, DatabaseURL, MigrationsPath, RedisURL, JWTSecret, GinMode string
    ReadTimeout, WriteTimeout, ShutdownTimeout time.Duration
}

func Load() (*Config, error) {
    dbURL := os.Getenv("DATABASE_URL")
    if dbURL == "" { return nil, fmt.Errorf("DATABASE_URL is required") }
    jwtSecret := os.Getenv("JWT_SECRET")
    if jwtSecret == "" { return nil, fmt.Errorf("JWT_SECRET is required") }

    port := os.Getenv("PORT"); if port == "" { port = "8080" }
    migrationsPath := os.Getenv("MIGRATIONS_PATH"); if migrationsPath == "" { migrationsPath = "db/migrations" }

    return &Config{
        Port: port, DatabaseURL: dbURL, MigrationsPath: migrationsPath,
        RedisURL: os.Getenv("REDIS_URL"), JWTSecret: jwtSecret,
        GinMode: os.Getenv("GIN_MODE"),
        ReadTimeout:     parseDuration(os.Getenv("READ_TIMEOUT"), 10*time.Second),
        WriteTimeout:    parseDuration(os.Getenv("WRITE_TIMEOUT"), 10*time.Second),
        ShutdownTimeout: parseDuration(os.Getenv("SHUTDOWN_TIMEOUT"), 30*time.Second),
    }, nil
}

func parseDuration(s string, fallback time.Duration) time.Duration {
    if s == "" { return fallback }
    if d, err := time.ParseDuration(s); err == nil { return d }
    return fallback
}
```

Required env vars: `DATABASE_URL`, `JWT_SECRET`. Optional with defaults: `PORT` (8080), `MIGRATIONS_PATH` (db/migrations), `GIN_MODE` (set to `release` in production), `READ_TIMEOUT`/`WRITE_TIMEOUT`/`SHUTDOWN_TIMEOUT`.

> For Dockerfile patterns, .dockerignore, and docker-compose quick-reference: see [configuration-docker-quickref.md](configuration-docker-quickref.md).
