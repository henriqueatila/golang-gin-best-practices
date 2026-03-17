# Redis Patterns — Session Storage, Rate Limiting, and Health

JWT blacklist storage, distributed sliding-window rate limiting, health checks, and docker-compose config.

## Session Storage — JWT Blacklist

Store revoked JWT IDs (`jti`) in Redis to support token invalidation. Ties to `golang-gin-auth`.

```go
// internal/repository/token_blacklist.go
package repository

import (
    "context"
    "time"

    "github.com/redis/go-redis/v9"
)

type TokenBlacklist struct {
    rdb *redis.Client
}

func NewTokenBlacklist(rdb *redis.Client) *TokenBlacklist {
    return &TokenBlacklist{rdb: rdb}
}

// Revoke stores the jti until the token's natural expiry.
func (b *TokenBlacklist) Revoke(ctx context.Context, jti string, ttl time.Duration) error {
    return b.rdb.Set(ctx, "blacklist:"+jti, "1", ttl).Err()
}

// IsRevoked returns true if the jti has been blacklisted.
func (b *TokenBlacklist) IsRevoked(ctx context.Context, jti string) (bool, error) {
    n, err := b.rdb.Exists(ctx, "blacklist:"+jti).Result()
    return n > 0, err
}
```

Use `IsRevoked` inside your JWT auth middleware after validating the token signature.

## Distributed Rate Limiting — Sliding Window

Redis-backed rate limiter safe for multi-instance deployments.

```go
// internal/middleware/rate_limiter.go
package middleware

import (
    "context"
    "fmt"
    "net/http"
    "time"

    "github.com/gin-gonic/gin"
    "github.com/redis/go-redis/v9"
)

// SlidingWindowRateLimiter allows `limit` requests per `window` per IP.
func SlidingWindowRateLimiter(rdb *redis.Client, limit int, window time.Duration) gin.HandlerFunc {
    return func(c *gin.Context) {
        ctx := c.Request.Context()
        key := fmt.Sprintf("rate:%s", c.ClientIP())
        now := time.Now().UnixMilli()
        windowStart := now - window.Milliseconds()

        pipe := rdb.Pipeline()
        pipe.ZRemRangeByScore(ctx, key, "0", fmt.Sprintf("%d", windowStart))
        pipe.ZAdd(ctx, key, redis.Z{Score: float64(now), Member: now})
        pipe.ZCard(ctx, key)
        pipe.Expire(ctx, key, window)

        results, err := pipe.Exec(ctx)
        if err != nil {
            c.Next() // fail open; log err in production
            return
        }

        count := results[2].(*redis.IntCmd).Val()
        if count > int64(limit) {
            c.AbortWithStatusJSON(http.StatusTooManyRequests, gin.H{"error": "rate limit exceeded"})
            return
        }
        c.Next()
    }
}
```

## Health Check

```go
// internal/handler/health.go
func (h *HealthHandler) Check(c *gin.Context) {
    ctx := c.Request.Context()
    status := gin.H{"db": "ok", "redis": "ok"}
    code := http.StatusOK

    if err := h.rdb.Ping(ctx).Err(); err != nil {
        status["redis"] = "unavailable"
        code = http.StatusServiceUnavailable
    }

    c.JSON(code, status)
}
```

## docker-compose Redis Service

```yaml
# docker-compose.yml (excerpt) — ties to golang-gin-deploy skill
services:
  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: redis-server --requirepass ${REDIS_PASSWORD}
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  redis_data:
```

Set `REDIS_URL=redis:6379` and `REDIS_PASSWORD=...` in your `.env` file.
