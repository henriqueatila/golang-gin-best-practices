# Configuration and Health Check Implementations

Full code for health check handler, 12-factor config, .dockerignore, multi-stage Dockerfile, and docker-compose for local development.

## Health Check Endpoint

Expose `/health` for container orchestrators. The endpoint is used directly by K8s probes.

```go
// internal/handler/health_handler.go
package handler

import (
    "context"
    "net/http"
    "time"

    "github.com/gin-gonic/gin"
    "log/slog"
)

type HealthHandler struct {
    db     DBPinger
    logger *slog.Logger
}

// DBPinger is satisfied by *sql.DB and *sqlx.DB
type DBPinger interface {
    PingContext(ctx context.Context) error
}

func NewHealthHandler(db DBPinger, logger *slog.Logger) *HealthHandler {
    return &HealthHandler{db: db, logger: logger}
}

func (h *HealthHandler) Check(c *gin.Context) {
    ctx, cancel := context.WithTimeout(c.Request.Context(), 2*time.Second)
    defer cancel()

    status := "ok"
    httpStatus := http.StatusOK

    if err := h.db.PingContext(ctx); err != nil {
        h.logger.Error("health: db ping failed", "error", err)
        status = "degraded"
        httpStatus = http.StatusServiceUnavailable
    }

    c.JSON(httpStatus, gin.H{"status": status})
}
```

Register in `main.go`:

```go
healthHandler := handler.NewHealthHandler(db, logger)
r.GET("/health", healthHandler.Check)
```

## Readiness vs Liveness Probes

| Probe | Checks | On failure |
|-------|--------|------------|
| **Liveness** | Is the process alive? | Restart container |
| **Readiness** | Can the app serve traffic? | Remove from load balancer (no restart) |

Use a single `/health` endpoint for both, or split:

- `/health/live` — returns 200 if process is running (no DB check)
- `/health/ready` — returns 200 only when DB is reachable

**Architectural recommendation:** For most Gin APIs, one `/health` endpoint that pings the DB is sufficient. Split only when startup time causes false liveness failures.

## Configuration via Environment Variables (12-Factor)

Never hardcode values. Read all configuration from the environment.

```go
// internal/config/config.go
package config

import (
    "fmt"
    "os"
    "time"
)

type Config struct {
    Port            string
    DatabaseURL     string
    MigrationsPath  string
    RedisURL        string
    JWTSecret       string
    ReadTimeout     time.Duration
    WriteTimeout    time.Duration
    ShutdownTimeout time.Duration
    GinMode         string
}

func Load() (*Config, error) {
    port := os.Getenv("PORT")
    if port == "" {
        port = "8080"
    }

    dbURL := os.Getenv("DATABASE_URL")
    if dbURL == "" {
        return nil, fmt.Errorf("DATABASE_URL is required")
    }

    jwtSecret := os.Getenv("JWT_SECRET")
    if jwtSecret == "" {
        return nil, fmt.Errorf("JWT_SECRET is required")
    }

    readTimeout := parseDuration(os.Getenv("READ_TIMEOUT"), 10*time.Second)
    writeTimeout := parseDuration(os.Getenv("WRITE_TIMEOUT"), 10*time.Second)
    shutdownTimeout := parseDuration(os.Getenv("SHUTDOWN_TIMEOUT"), 30*time.Second)

    migrationsPath := os.Getenv("MIGRATIONS_PATH")
    if migrationsPath == "" {
        migrationsPath = "db/migrations"
    }

    return &Config{
        Port:            port,
        DatabaseURL:     dbURL,
        MigrationsPath:  migrationsPath,
        RedisURL:        os.Getenv("REDIS_URL"),
        JWTSecret:       jwtSecret,
        ReadTimeout:     readTimeout,
        WriteTimeout:    writeTimeout,
        ShutdownTimeout: shutdownTimeout,
        GinMode:         os.Getenv("GIN_MODE"), // "release" in production
    }, nil
}

func parseDuration(s string, fallback time.Duration) time.Duration {
    if s == "" {
        return fallback
    }
    d, err := time.ParseDuration(s)
    if err != nil {
        return fallback
    }
    return d
}
```

## .dockerignore

```dockerignore
# Version control
.git
.gitignore

# Go build artifacts
*.test
*.out
coverage.html

# Local development
.env
.env.*
docker-compose*.yml
air.toml

# Documentation and plans
*.md
docs/
plans/

# CI
.github/
```

**Why:** Excluding `.git` and `*.md` keeps the Docker build context small. Excluding `.env` prevents secrets from leaking into the image.

## Multi-Stage Dockerfile

Build a minimal production image from the `cmd/api/main.go` entry point (same structure as the **golang-gin-api** skill).

```dockerfile
# syntax=docker/dockerfile:1

# ── Stage 1: Builder ──────────────────────────────────────────────────────────
FROM golang:1.24-bookworm AS builder

WORKDIR /build

# Cache dependencies before copying source
COPY go.mod go.sum ./
RUN go mod download

COPY . .

# Build a statically linked binary; CGO_ENABLED=0 required for distroless
ARG TARGETARCH
RUN CGO_ENABLED=0 GOOS=linux GOARCH=${TARGETARCH} go build \
    -ldflags="-s -w" \
    -trimpath \
    -o /app/server \
    ./cmd/api

# ── Stage 2: Runtime ─────────────────────────────────────────────────────────
FROM gcr.io/distroless/static-debian12:nonroot

COPY --from=builder /app/server /server

# distroless nonroot runs as UID 65532 — no root needed
USER nonroot:nonroot

EXPOSE 8080

ENTRYPOINT ["/server"]
```

**Why distroless:** No shell, no package manager — drastically reduces attack surface. Final image is ~10 MB vs ~800 MB with golang base.

**Critical:** `CGO_ENABLED=0` is mandatory for distroless/scratch — it produces a statically linked binary with no libc dependency.

## docker-compose for Local Development

```yaml
# docker-compose.yml
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - PORT=8080
      - DATABASE_URL=postgres://myapp:myapp@postgres:5432/myapp?sslmode=disable
      - REDIS_URL=redis://redis:6379
      - JWT_SECRET=dev-secret-change-in-production
      - GIN_MODE=debug
      - MIGRATIONS_PATH=db/migrations
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      - .:/app  # for Air hot reload — remove in production

  postgres:
    image: postgres:17-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: myapp
      POSTGRES_PASSWORD: myapp
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myapp -d myapp"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

volumes:
  postgres_data:
```

For migration running in Docker, see the **golang-gin-database** skill (`references/migrations.md`).
