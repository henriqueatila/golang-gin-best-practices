# Redis Patterns

Redis integration for Go Gin APIs: caching, session/blacklist storage, distributed rate limiting, and health checks.

## Dependencies

```bash
go get github.com/redis/go-redis/v9
```

## Connection Setup

```go
// internal/infrastructure/redis.go
package infrastructure

import (
    "context"
    "crypto/tls"
    "fmt"
    "log/slog"
    "os"
    "time"

    "github.com/redis/go-redis/v9"
)

func NewRedisClient(ctx context.Context, logger *slog.Logger) (*redis.Client, error) {
    opts := &redis.Options{
        Addr:         os.Getenv("REDIS_URL"),      // e.g. "localhost:6379"
        Password:     os.Getenv("REDIS_PASSWORD"),
        DB:           0,
        DialTimeout:  5 * time.Second,
        ReadTimeout:  3 * time.Second,
        WriteTimeout: 3 * time.Second,
        MaxRetries:   3,
    }

    if os.Getenv("REDIS_TLS") == "true" {
        opts.TLSConfig = &tls.Config{MinVersion: tls.VersionTLS12}
    }

    rdb := redis.NewClient(opts)

    if err := rdb.Ping(ctx).Err(); err != nil {
        return nil, fmt.Errorf("redis ping failed: %w", err)
    }

    logger.Info("redis connected", "addr", opts.Addr)
    return rdb, nil
}
```

## Cache Repository Pattern

```go
// internal/domain/cache.go
package domain

import (
    "context"
    "time"
)

type CacheRepository interface {
    Get(ctx context.Context, key string) (string, error)
    Set(ctx context.Context, key string, value any, ttl time.Duration) error
    Delete(ctx context.Context, key string) error
    Exists(ctx context.Context, key string) (bool, error)
}
```

```go
// internal/repository/redis_cache.go
package repository

import (
    "context"
    "encoding/json"
    "errors"
    "time"

    "github.com/redis/go-redis/v9"
    "myapp/internal/domain"
)

type redisCacheRepo struct {
    rdb *redis.Client
}

func NewRedisCacheRepository(rdb *redis.Client) domain.CacheRepository {
    return &redisCacheRepo{rdb: rdb}
}

func (r *redisCacheRepo) Get(ctx context.Context, key string) (string, error) {
    val, err := r.rdb.Get(ctx, key).Result()
    if errors.Is(err, redis.Nil) {
        return "", domain.ErrCacheMiss
    }
    return val, err
}

func (r *redisCacheRepo) Set(ctx context.Context, key string, value any, ttl time.Duration) error {
    data, err := json.Marshal(value)
    if err != nil {
        return fmt.Errorf("marshal cache value: %w", err)
    }
    return r.rdb.Set(ctx, key, data, ttl).Err()
}

func (r *redisCacheRepo) Delete(ctx context.Context, key string) error {
    return r.rdb.Del(ctx, key).Err()
}

func (r *redisCacheRepo) Exists(ctx context.Context, key string) (bool, error) {
    n, err := r.rdb.Exists(ctx, key).Result()
    return n > 0, err
}
```

## Cache-Aside Pattern

Check cache first; on miss, query DB, store result, return.

```go
// internal/service/user_service.go
func (s *userService) GetByID(ctx context.Context, id uint) (*domain.User, error) {
    cacheKey := fmt.Sprintf("user:%d", id)

    // 1. Check cache
    cached, err := s.cache.Get(ctx, cacheKey)
    if err == nil {
        var user domain.User
        if jsonErr := json.Unmarshal([]byte(cached), &user); jsonErr == nil {
            return &user, nil
        }
    }

    // 2. Cache miss — query DB
    user, err := s.repo.GetByID(ctx, id)
    if err != nil {
        return nil, err
    }

    // 3. Populate cache (fire-and-forget acceptable for non-critical data)
    _ = s.cache.Set(ctx, cacheKey, user, 5*time.Minute)

    return user, nil
}
```

## Cache Invalidation — Delete on Write

```go
func (s *userService) Update(ctx context.Context, id uint, input domain.UpdateUserInput) (*domain.User, error) {
    user, err := s.repo.Update(ctx, id, input)
    if err != nil {
        return nil, err
    }
    // Invalidate stale entry immediately after write
    _ = s.cache.Delete(ctx, fmt.Sprintf("user:%d", id))
    return user, nil
}
```

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

## Dependency Injection in main.go

```go
// main.go (excerpt)
rdb, err := infrastructure.NewRedisClient(ctx, logger)
if err != nil {
    logger.Error("redis init failed", "err", err)
    os.Exit(1)
}
defer rdb.Close()

cacheRepo  := repository.NewRedisCacheRepository(rdb)
blacklist  := repository.NewTokenBlacklist(rdb)
userSvc    := service.NewUserService(userRepo, cacheRepo, logger)

r := gin.New()
r.Use(middleware.SlidingWindowRateLimiter(rdb, 100, time.Minute))
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
