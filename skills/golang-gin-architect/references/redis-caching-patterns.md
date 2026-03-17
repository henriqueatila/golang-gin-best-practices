# Redis Caching — Cache-Aside, Key Design, and Stampede Prevention

Core caching patterns for Go Gin APIs using Redis. For basic get/set see `cross-cutting-security-config.md`.

---

## What to Cache (Decision Matrix)

| Data Type | Cache? | Strategy | TTL |
|---|---|---|---|
| User profile | Yes (read-heavy) | Cache-aside, invalidate on update | 10 min |
| Product catalog | Yes (read-heavy, shared) | Cache-aside + warming | 1 h |
| Session / auth tokens | Yes (every request) | Write-through to Redis | Token lifetime |
| Search results | Maybe (if query > 100ms) | Cache-aside | 5 min |
| Real-time inventory | No (stale = oversell) | Always query DB | — |
| Configuration / feature flags | Yes | Cache-aside + pub/sub invalidation | 5 min |
| One-time codes (OTP, CSRF) | Yes | Write-through, single-use delete | 5–15 min |

**Do NOT cache:** Data that must be consistent (payments, inventory counts), user-specific write-heavy data, tiny datasets (<10 rows).

---

## Cache-Aside (Recommended Default)

Read: check cache → miss → fetch DB → store → return. Write: update DB → invalidate cache.

```go
func (s *ProductService) GetByID(ctx context.Context, id string) (*Product, error) {
    key := fmt.Sprintf("myapp:product:%s", id)

    data, err := s.redis.Get(ctx, key).Bytes()
    if err == nil {
        var p Product
        if err := json.Unmarshal(data, &p); err == nil {
            return &p, nil
        }
    }
    if err != nil && err != redis.Nil {
        s.logger.Warn("redis get failed", "key", key, "err", err)
    }

    p, err := s.repo.GetByID(ctx, id)
    if err != nil {
        return nil, fmt.Errorf("get product: %w", err)
    }
    if b, err := json.Marshal(p); err == nil {
        _ = s.redis.Set(ctx, key, b, time.Hour).Err()
    }
    return p, nil
}

func (s *ProductService) Update(ctx context.Context, p *Product) error {
    if err := s.repo.Update(ctx, p); err != nil {
        return fmt.Errorf("update product: %w", err)
    }
    _ = s.redis.Del(ctx, fmt.Sprintf("myapp:product:%s", p.ID)).Err()
    return nil
}
```

---

## Cache Key Design

**Pattern:** `{app}:{version}:{entity}:{id}`

```go
const (
    KeyUser     = "myapp:v1:user:%s"
    KeyUserList = "myapp:v1:user:list:%d:%d"
    KeyProduct  = "myapp:v1:product:%s"
    KeySession  = "myapp:session:%s"
)

func userKey(id string) string         { return fmt.Sprintf(KeyUser, id) }
func userListKey(page, size int) string { return fmt.Sprintf(KeyUserList, page, size) }
```

**Rules:** Namespace prevents collision. Version (`v1`) allows schema changes without flush. Never interpolate raw user input. Use `SCAN` + prefix for bulk-delete.

```go
func (s *UserService) InvalidateListCache(ctx context.Context) error {
    var cursor uint64
    for {
        keys, next, err := s.redis.Scan(ctx, cursor, "myapp:v1:user:list:*", 100).Result()
        if err != nil {
            return fmt.Errorf("scan keys: %w", err)
        }
        if len(keys) > 0 {
            _ = s.redis.Del(ctx, keys...).Err()
        }
        cursor = next
        if cursor == 0 {
            break
        }
    }
    return nil
}
```

---

## Cache Stampede Prevention (singleflight)

**Problem:** Cache TTL expires — concurrent misses all hit DB simultaneously. **Solution:** `singleflight` collapses them — only ONE goroutine fetches; others share the result.

```go
import "golang.org/x/sync/singleflight"

type ProductService struct {
    redis  *redis.Client
    repo   ProductRepository
    sf     singleflight.Group
    logger *slog.Logger
}

func (s *ProductService) GetByID(ctx context.Context, id string) (*Product, error) {
    key := fmt.Sprintf("myapp:v1:product:%s", id)

    data, err := s.redis.Get(ctx, key).Bytes()
    if err == nil {
        var p Product
        if jsonErr := json.Unmarshal(data, &p); jsonErr == nil {
            return &p, nil
        }
    }

    v, err, _ := s.sf.Do(key, func() (any, error) {
        p, err := s.repo.GetByID(ctx, id)
        if err != nil {
            return nil, fmt.Errorf("fetch product %s: %w", id, err)
        }
        b, _ := json.Marshal(p)
        ttl := time.Hour + time.Duration(rand.Intn(300))*time.Second // jitter
        _ = s.redis.Set(ctx, key, b, ttl).Err()
        return p, nil
    })
    if err != nil {
        return nil, err
    }
    return v.(*Product), nil
}
```

**TTL jitter** prevents synchronized stampedes: `base + rand.Intn(maxSec)*time.Second`.
