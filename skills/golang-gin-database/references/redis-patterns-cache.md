# Redis Patterns — Cache-Aside, Invalidation, and DI Wiring

Cache-aside pattern, cache invalidation on write, and dependency injection wiring.

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

## Dependency Injection in main.go

```go
// main.go (excerpt)
rdb, err := infrastructure.NewRedisClient(ctx, logger)
if err != nil {
    logger.Error("redis init failed", "err", err)
    os.Exit(1)
}
defer rdb.Close()

cacheRepo := repository.NewRedisCacheRepository(rdb)
blacklist  := repository.NewTokenBlacklist(rdb)
userSvc    := service.NewUserService(userRepo, cacheRepo, logger)

r := gin.New()
r.Use(middleware.SlidingWindowRateLimiter(rdb, 100, time.Minute))
```

For JWT blacklist, sliding-window rate limiter, health check, and docker-compose config: see [redis-patterns-advanced.md](redis-patterns-advanced.md).
