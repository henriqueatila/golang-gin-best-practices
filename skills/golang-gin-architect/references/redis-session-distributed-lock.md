# Redis — Session Storage and Distributed Locking

Companion to `redis-cache-warming-pubsub.md` (cache warming, pub/sub invalidation).

---

## Session Storage in Redis

Use for server-side sessions: JWT revocation, large session payloads, multi-device logout.

```go
type SessionStore struct {
    redis  *redis.Client
    logger *slog.Logger
}

type UserSession struct {
    UserID    string    `json:"user_id"`
    Email     string    `json:"email"`
    Roles     []string  `json:"roles"`
    CreatedAt time.Time `json:"created_at"`
}

func sessionKey(sessionID string) string {
    return fmt.Sprintf("myapp:session:%s", sessionID)
}

func (s *SessionStore) Save(ctx context.Context, sessionID string, sess *UserSession, ttl time.Duration) error {
    b, err := json.Marshal(sess)
    if err != nil {
        return fmt.Errorf("marshal session: %w", err)
    }
    return s.redis.Set(ctx, sessionKey(sessionID), b, ttl).Err()
}

func (s *SessionStore) Get(ctx context.Context, sessionID string) (*UserSession, error) {
    data, err := s.redis.Get(ctx, sessionKey(sessionID)).Bytes()
    if err == redis.Nil {
        return nil, nil // session not found / expired
    }
    if err != nil {
        return nil, fmt.Errorf("get session: %w", err)
    }
    var sess UserSession
    if err := json.Unmarshal(data, &sess); err != nil {
        return nil, fmt.Errorf("unmarshal session: %w", err)
    }
    return &sess, nil
}

func (s *SessionStore) Delete(ctx context.Context, sessionID string) error {
    return s.redis.Del(ctx, sessionKey(sessionID)).Err()
}
```

---

## Distributed Locking

Use to prevent duplicate processing in multi-instance deployments.

```go
type DistributedLock struct{ redis *redis.Client }

func (l *DistributedLock) TryLock(ctx context.Context, resource string, ttl time.Duration) (acquired bool, unlock func(), err error) {
    lockKey := fmt.Sprintf("myapp:lock:%s", resource)
    lockVal := fmt.Sprintf("%d", time.Now().UnixNano())

    ok, err := l.redis.SetNX(ctx, lockKey, lockVal, ttl).Result()
    if err != nil {
        return false, nil, fmt.Errorf("acquire lock %s: %w", resource, err)
    }
    if !ok {
        return false, nil, nil
    }

    unlock = func() {
        script := redis.NewScript(`
            if redis.call("get", KEYS[1]) == ARGV[1] then
                return redis.call("del", KEYS[1])
            else
                return 0
            end
        `)
        _ = script.Run(ctx, l.redis, []string{lockKey}, lockVal).Err()
    }
    return true, unlock, nil
}

// Usage — prevent duplicate payment processing
func (s *PaymentService) ProcessPayment(ctx context.Context, orderID string) error {
    acquired, unlock, err := s.lock.TryLock(ctx, "payment:"+orderID, 30*time.Second)
    if err != nil {
        return fmt.Errorf("lock check: %w", err)
    }
    if !acquired {
        return fmt.Errorf("payment for order %s already in progress", orderID)
    }
    defer unlock()
    return s.doProcessPayment(ctx, orderID)
}
```

**Notes:** TTL must exceed max expected operation duration. Lua script ensures atomic check-and-delete. For stronger guarantees, use `github.com/go-redsync/redsync`.

---

## Redis Connection Setup

```go
func NewRedisClient(url string, logger *slog.Logger) (*redis.Client, error) {
    opts, err := redis.ParseURL(url)
    if err != nil {
        return nil, fmt.Errorf("parse redis url: %w", err)
    }
    opts.PoolSize = 20
    opts.MinIdleConns = 5
    opts.ConnMaxIdleTime = 5 * time.Minute
    opts.DialTimeout = 3 * time.Second
    opts.ReadTimeout = 2 * time.Second
    opts.WriteTimeout = 2 * time.Second

    client := redis.NewClient(opts)
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    if err := client.Ping(ctx).Err(); err != nil {
        return nil, fmt.Errorf("redis ping: %w", err)
    }
    logger.Info("redis connected", "addr", opts.Addr)
    return client, nil
}
```

**Required modules:** `go get github.com/redis/go-redis/v9`
