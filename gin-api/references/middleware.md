# Middleware Reference

This file covers Gin middleware patterns: signature and chain execution order, CORS configuration, request logging with log/slog, rate limiting, request ID injection, recovery with custom JSON response, timeout middleware, and a custom middleware template.

## Table of Contents

1. [Middleware Signature & Chain Execution](#middleware-signature--chain-execution)
2. [CORS Configuration](#cors-configuration)
3. [Request Logging with log/slog](#request-logging-with-logslog)
4. [Rate Limiting](#rate-limiting)
5. [Request ID Middleware](#request-id-middleware)
6. [Recovery Middleware](#recovery-middleware)
7. [Timeout Middleware](#timeout-middleware)
8. [Custom Middleware Template](#custom-middleware-template)

---

## Middleware Signature & Chain Execution

All handlers and middleware share the same signature: `func(*gin.Context)`.

```go
// Execution order with two middleware and one handler:
// Middleware1-before → Middleware2-before → Handler → Middleware2-after → Middleware1-after

func Middleware1(c *gin.Context) {
    // runs BEFORE handler
    c.Next()
    // runs AFTER handler (and all subsequent middleware)
    status := c.Writer.Status()
    _ = status
}

func Middleware2(c *gin.Context) {
    c.Next()
}

func Handler(c *gin.Context) {
    c.JSON(200, gin.H{"ok": true})
}
```

Registration order matters. Register middleware before routes:

```go
r := gin.New()                    // no middleware
r.Use(middleware.RequestID())     // 1st: inject request ID (used by logger)
r.Use(middleware.Logger(logger))  // 2nd: logs with request ID available
r.Use(middleware.Recovery(logger)) // 3rd: catches panics from any handler

// Routes registered after middleware
r.GET("/health", healthHandler)
```

**Why `gin.New()` over `gin.Default()`:** `gin.Default()` adds Logger and Recovery with their default configuration. Use `gin.New()` + explicit middleware so you control format, destination, and behaviour of each middleware — critical for structured logging and custom error responses in production.

---

## CORS Configuration

Use `github.com/gin-contrib/cors` for CORS. Configure per environment.

```go
// pkg/middleware/cors.go
package middleware

import (
    "os"
    "time"

    "github.com/gin-contrib/cors"
    "github.com/gin-gonic/gin"
)

// CORS returns a configured CORS middleware.
// In development, allows all origins. In production, restricts to configured origins.
func CORS() gin.HandlerFunc {
    env := os.Getenv("APP_ENV")

    if env == "development" {
        return cors.New(cors.Config{
            AllowAllOrigins:  true,
            AllowMethods:     []string{"GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"},
            AllowHeaders:     []string{"Origin", "Content-Type", "Authorization", "X-Request-ID"},
            AllowCredentials: true,
            MaxAge:           12 * time.Hour,
        })
    }

    // Production: explicit origin allowlist
    origins := os.Getenv("ALLOWED_ORIGINS") // e.g. "https://app.example.com,https://admin.example.com"
    allowedOrigins := []string{}
    if origins != "" {
        for _, o := range strings.Split(origins, ",") {
            allowedOrigins = append(allowedOrigins, strings.TrimSpace(o))
        }
    }

    return cors.New(cors.Config{
        AllowOrigins:     allowedOrigins,
        AllowMethods:     []string{"GET", "POST", "PUT", "PATCH", "DELETE"},
        AllowHeaders:     []string{"Origin", "Content-Type", "Authorization", "X-Request-ID"},
        ExposeHeaders:    []string{"X-Request-ID"},
        AllowCredentials: true,
        MaxAge:           1 * time.Hour,
    })
}
```

Register before routes:

```go
r := gin.New()
r.Use(middleware.CORS())
r.Use(middleware.RequestID())
r.Use(middleware.Logger(logger))
r.Use(middleware.Recovery(logger))
```

**Warning:** `AllowAllOrigins: true` with `AllowCredentials: true` is rejected by browsers. Use one or the other, never both in production.

---

## Request Logging with log/slog

Log each request with structured fields: method, path, status, duration, request ID.

```go
// pkg/middleware/logger.go
package middleware

import (
    "log/slog"
    "time"

    "github.com/gin-gonic/gin"
)

// Logger returns a Gin middleware that logs each request using log/slog.
// Requires RequestID middleware to run first for request ID injection.
func Logger(logger *slog.Logger) gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()
        path := c.Request.URL.Path
        query := c.Request.URL.RawQuery

        c.Next() // process request

        duration := time.Since(start)
        status := c.Writer.Status()

        attrs := []slog.Attr{
            slog.String("method", c.Request.Method),
            slog.String("path", path),
            slog.Int("status", status),
            slog.Duration("duration", duration),
            slog.String("ip", c.ClientIP()),
            slog.String("request_id", c.GetString("request_id")),
        }
        if query != "" {
            attrs = append(attrs, slog.String("query", query))
        }
        if len(c.Errors) > 0 {
            attrs = append(attrs, slog.String("errors", c.Errors.String()))
        }

        level := slog.LevelInfo
        if status >= 500 {
            level = slog.LevelError
        } else if status >= 400 {
            level = slog.LevelWarn
        }

        logger.LogAttrs(c.Request.Context(), level, "http request", attrs...)
    }
}
```

Sample output (JSON handler):

```json
{"time":"2026-03-01T10:00:00Z","level":"INFO","msg":"http request","method":"POST","path":"/api/v1/users","status":201,"duration":"2.3ms","ip":"192.168.1.1","request_id":"a1b2c3d4"}
```

---

## Rate Limiting

Use `golang.org/x/time/rate` (token bucket) per client IP.

```go
// pkg/middleware/rate_limiter.go
package middleware

import (
    "net/http"
    "sync"
    "time"

    "github.com/gin-gonic/gin"
    "golang.org/x/time/rate"
)

type ipLimiter struct {
    limiter  *rate.Limiter
    lastSeen time.Time
}

// RateLimiter returns middleware that limits requests per IP.
// r: requests per second allowed; burst: maximum burst size.
func RateLimiter(r rate.Limit, burst int) gin.HandlerFunc {
    var (
        mu       sync.Mutex
        limiters = make(map[string]*ipLimiter)
    )

    // Cleanup stale entries every minute
    go func() {
        for range time.Tick(time.Minute) {
            mu.Lock()
            for ip, l := range limiters {
                if time.Since(l.lastSeen) > 3*time.Minute {
                    delete(limiters, ip)
                }
            }
            mu.Unlock()
        }
    }()

    getLimiter := func(ip string) *rate.Limiter {
        mu.Lock()
        defer mu.Unlock()
        l, ok := limiters[ip]
        if !ok {
            l = &ipLimiter{limiter: rate.NewLimiter(r, burst)}
            limiters[ip] = l
        }
        l.lastSeen = time.Now()
        return l.limiter
    }

    return func(c *gin.Context) {
        limiter := getLimiter(c.ClientIP())
        if !limiter.Allow() {
            c.AbortWithStatusJSON(http.StatusTooManyRequests, gin.H{
                "error": "rate limit exceeded",
            })
            return
        }
        c.Next()
    }
}
```

Apply globally or to specific groups:

```go
// Global: 10 req/s, burst of 20
r.Use(middleware.RateLimiter(10, 20))

// Stricter limit on auth endpoints
auth := r.Group("/api/v1/auth")
auth.Use(middleware.RateLimiter(2, 5)) // 2 req/s, burst of 5
```

---

## Request ID Middleware

Inject a unique request ID per request. Propagate it in the response header and store in context for logging.

```go
// pkg/middleware/request_id.go
package middleware

import (
    "github.com/gin-gonic/gin"
    "github.com/google/uuid"
)

const RequestIDKey = "request_id"
const RequestIDHeader = "X-Request-ID"

// RequestID injects a unique request ID into the context and response headers.
// Reuses the incoming X-Request-ID header if present (for distributed tracing).
func RequestID() gin.HandlerFunc {
    return func(c *gin.Context) {
        requestID := c.GetHeader(RequestIDHeader)
        if requestID == "" {
            requestID = uuid.New().String()
        }

        c.Set(RequestIDKey, requestID)
        c.Header(RequestIDHeader, requestID)
        c.Next()
    }
}
```

Retrieve in any handler or downstream service:

```go
requestID := c.GetString(middleware.RequestIDKey)

// Pass to slog context for structured logging
logger := slog.With("request_id", requestID)
```

---

## Recovery Middleware

Return JSON on panic instead of Gin's default plain-text 500.

```go
// pkg/middleware/recovery.go
package middleware

import (
    "log/slog"
    "net/http"
    "runtime/debug"

    "github.com/gin-gonic/gin"
)

// Recovery returns middleware that recovers from panics, logs the stack trace,
// and returns a JSON 500 response. Use instead of gin.Recovery().
func Recovery(logger *slog.Logger) gin.HandlerFunc {
    return func(c *gin.Context) {
        defer func() {
            if rec := recover(); rec != nil {
                logger.ErrorContext(c.Request.Context(), "panic recovered",
                    "panic", rec,
                    "stack", string(debug.Stack()),
                    "method", c.Request.Method,
                    "path", c.FullPath(),
                    "request_id", c.GetString(RequestIDKey),
                )
                c.AbortWithStatusJSON(http.StatusInternalServerError, gin.H{
                    "error": "internal server error",
                })
            }
        }()
        c.Next()
    }
}
```

---

## Timeout Middleware

Cancel the request context after a deadline. Handlers must respect `c.Request.Context()` for this to work.

```go
// pkg/middleware/timeout.go
package middleware

import (
    "context"
    "net/http"
    "time"

    "github.com/gin-gonic/gin"
)

// Timeout returns middleware that cancels the request context after d.
// Handlers must pass c.Request.Context() to all blocking calls.
func Timeout(d time.Duration) gin.HandlerFunc {
    return func(c *gin.Context) {
        ctx, cancel := context.WithTimeout(c.Request.Context(), d)
        defer cancel()

        c.Request = c.Request.WithContext(ctx)

        done := make(chan struct{})
        go func() {
            c.Next()
            close(done)
        }()

        select {
        case <-done:
            // Handler completed within timeout
        case <-ctx.Done():
            c.AbortWithStatusJSON(http.StatusGatewayTimeout, gin.H{
                "error": "request timeout",
            })
        }
    }
}
```

Apply per route group, not globally (health check should not time out):

```go
api := r.Group("/api/v1")
api.Use(middleware.Timeout(30 * time.Second))
```

**Critical:** This middleware launches `c.Next()` in a goroutine. If your handlers write to `c` after the timeout fires and `AbortWithStatusJSON` has run, Gin will ignore the duplicate write (first write wins). This is safe but handlers should always check `ctx.Done()` in long loops.

---

## Custom Middleware Template

Use this as a starting point for any new middleware.

```go
// pkg/middleware/my_middleware.go
package middleware

import (
    "log/slog"
    "net/http"

    "github.com/gin-gonic/gin"
)

// MyMiddleware does X. It requires Y to run before it.
// Apply at: r.Use() for global, group.Use() for group-scoped, or inline for per-route.
func MyMiddleware(logger *slog.Logger) gin.HandlerFunc {
    // One-time setup (runs at registration, not per request)
    // e.g., compile regexps, open connections

    return func(c *gin.Context) {
        // --- PRE-HANDLER ---
        // Read request data: c.GetHeader(), c.Query(), c.Param()
        // Short-circuit with c.AbortWithStatusJSON() if validation fails

        val := c.GetHeader("X-My-Header")
        if val == "" {
            c.AbortWithStatusJSON(http.StatusBadRequest, gin.H{
                "error": "X-My-Header is required",
            })
            return
        }

        // Store data for downstream handlers
        c.Set("my_value", val)

        // --- CALL NEXT ---
        c.Next()

        // --- POST-HANDLER ---
        // Read response data: c.Writer.Status(), c.Writer.Size()
        // Add response headers, log, metrics
        status := c.Writer.Status()
        logger.InfoContext(c.Request.Context(), "my middleware post",
            "status", status,
            "value", val,
        )
    }
}
```
