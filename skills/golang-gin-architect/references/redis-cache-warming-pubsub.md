# Redis — Cache Warming and Pub/Sub Invalidation

Advanced Redis patterns beyond basic caching for Go Gin APIs.

---

## Cache Warming

**Pattern A — Startup warming** for critical, bounded datasets:

```go
func (s *ProductService) WarmCache(ctx context.Context) error {
    s.logger.Info("warming product cache")
    products, err := s.repo.ListAll(ctx)
    if err != nil {
        return fmt.Errorf("warm cache: %w", err)
    }
    pipe := s.redis.Pipeline()
    for _, p := range products {
        b, _ := json.Marshal(p)
        key := fmt.Sprintf("myapp:v1:product:%s", p.ID)
        pipe.Set(ctx, key, b, withJitter(time.Hour, 300))
    }
    _, err = pipe.Exec(ctx)
    return err
}

// Call in main.go after Redis connection established
if err := productService.WarmCache(context.Background()); err != nil {
    logger.Warn("cache warm failed, degrading gracefully", "err", err)
}
```

**Pattern B — Background refresh** (serve stale, refresh before expiry):

```go
type CachedItem[T any] struct {
    Data      T         `json:"data"`
    ExpiresAt time.Time `json:"expires_at"`
    RefreshAt time.Time `json:"refresh_at"` // 80% of TTL
}

func (s *ProductService) GetWithEarlyRefresh(ctx context.Context, id string) (*Product, error) {
    key := fmt.Sprintf("myapp:v1:product:%s", id)
    data, err := s.redis.Get(ctx, key).Bytes()
    if err == nil {
        var item CachedItem[Product]
        if json.Unmarshal(data, &item) == nil {
            if time.Now().After(item.RefreshAt) {
                go s.refreshAsync(context.Background(), id, key)
            }
            return &item.Data, nil
        }
    }
    return s.fetchAndCache(ctx, id, key)
}
```

---

## Pub/Sub Invalidation (Multi-Instance)

Use when multiple app instances each have local in-memory caches. When instance A updates data, it tells others to invalidate.

```go
const invalidationChannel = "myapp:cache:invalidations"

type InvalidationMessage struct {
    Entity string `json:"entity"`
    ID     string `json:"id"`
}

func (s *ProductService) publishInvalidation(ctx context.Context, id string) {
    msg := InvalidationMessage{Entity: "product", ID: id}
    b, _ := json.Marshal(msg)
    if err := s.redis.Publish(ctx, invalidationChannel, b).Err(); err != nil {
        s.logger.Warn("publish invalidation failed", "err", err)
    }
}

func (s *ProductService) SubscribeInvalidations(ctx context.Context) {
    sub := s.redis.Subscribe(ctx, invalidationChannel)
    defer sub.Close()
    ch := sub.Channel()
    for {
        select {
        case <-ctx.Done():
            return
        case msg, ok := <-ch:
            if !ok {
                return
            }
            var inv InvalidationMessage
            if err := json.Unmarshal([]byte(msg.Payload), &inv); err != nil {
                continue
            }
            if inv.Entity == "product" {
                s.localCache.Delete(inv.ID)
            }
        }
    }
}

// Wire in main.go
go productService.SubscribeInvalidations(ctx)
```

---

## See Also

- `redis-session-distributed-lock.md` — session storage, distributed locking, Redis connection setup
- `redis-caching-patterns.md` — cache-aside, key design, singleflight stampede prevention
