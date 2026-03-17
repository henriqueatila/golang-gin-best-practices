# Redis Patterns — Setup and Cache Repository

Connection setup, cache repository interface, cache-aside pattern, and cache invalidation.

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
    "fmt"
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

For cache-aside pattern, cache invalidation, and DI wiring: see [redis-patterns-cache.md](redis-patterns-cache.md).
