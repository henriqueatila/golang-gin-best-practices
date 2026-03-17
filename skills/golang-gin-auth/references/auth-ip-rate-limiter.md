# Auth — IP Rate Limiter Middleware

See also: `auth-implementation-handlers.md`

## IPRateLimiter

Per-IP token bucket limiter using `golang.org/x/time/rate`. Apply to auth routes to defend against brute-force attacks.

```go
// pkg/middleware/rate_limiter.go
package middleware

import (
    "net/http"
    "sync"

    "github.com/gin-gonic/gin"
    "golang.org/x/time/rate"
)

func IPRateLimiter(r rate.Limit, b int) gin.HandlerFunc {
    limiters := make(map[string]*rate.Limiter)
    var mu sync.Mutex

    getLimiter := func(ip string) *rate.Limiter {
        mu.Lock()
        defer mu.Unlock()
        if lim, ok := limiters[ip]; ok {
            return lim
        }
        lim := rate.NewLimiter(r, b)
        limiters[ip] = lim
        return lim
    }

    return func(c *gin.Context) {
        if !getLimiter(c.ClientIP()).Allow() {
            c.AbortWithStatusJSON(http.StatusTooManyRequests, gin.H{"error": "too many requests"})
            return
        }
        c.Next()
    }
}
```

## Usage on Auth Routes

```go
authRoutes := api.Group("/auth")
authRoutes.Use(middleware.IPRateLimiter(rate.Every(12*time.Second), 5)) // 5 req/min
{
    authRoutes.POST("/login", authHandler.Login)
    authRoutes.POST("/register", authHandler.Register)
    authRoutes.POST("/refresh", authHandler.Refresh)
}
```

> **Production note:** The in-process map works for single-instance deployments. For multi-instance, use a Redis-backed limiter so limits are shared across all pods.
