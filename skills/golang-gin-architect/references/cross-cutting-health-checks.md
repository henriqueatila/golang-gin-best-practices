# Cross-Cutting Concerns — Health Checks

Liveness and readiness endpoints for Go Gin APIs.

Companion to `cross-cutting-observability.md` (logging, metrics, tracing).

---

## Health Check Handler

```go
type HealthChecker struct {
    db    *sqlx.DB
    redis *redis.Client // optional
}

func (h *HealthChecker) Liveness(c *gin.Context) {
    c.JSON(http.StatusOK, gin.H{"status": "alive"})
}

func (h *HealthChecker) Readiness(c *gin.Context) {
    ctx, cancel := context.WithTimeout(c.Request.Context(), 2*time.Second)
    defer cancel()

    checks := gin.H{"status": "ready"}
    if err := h.db.PingContext(ctx); err != nil {
        checks["status"] = "not_ready"
        checks["db"] = err.Error()
        c.JSON(http.StatusServiceUnavailable, checks)
        return
    }
    checks["db"] = "ok"

    if h.redis != nil {
        if err := h.redis.Ping(ctx).Err(); err != nil {
            checks["status"] = "not_ready"
            checks["redis"] = err.Error()
            c.JSON(http.StatusServiceUnavailable, checks)
            return
        }
        checks["redis"] = "ok"
    }
    c.JSON(http.StatusOK, checks)
}

r.GET("/health/live", health.Liveness)
r.GET("/health/ready", health.Readiness)
```

---

## Liveness vs Readiness

| Endpoint | Purpose | Kubernetes probe |
|---|---|---|
| `/health/live` | Is the process alive? Restart if fails. | `livenessProbe` |
| `/health/ready` | Can the service handle traffic? Remove from load balancer if fails. | `readinessProbe` |

**Liveness:** Never check external dependencies (DB, Redis). Only check if the process itself is healthy (not deadlocked).

**Readiness:** Check critical dependencies. If DB is unreachable, return 503 — Kubernetes stops routing traffic until DB recovers.
