# Resilience Patterns — Retry with Backoff and Rate Limiting

Companion to `resilience-circuit-breaker-bulkhead.md` (circuit breaker, bulkhead pattern).

---

## Retry with Exponential Backoff

**Cost: LOW (1/5).** Apply by default to any external HTTP or gRPC call. Not needed for local database calls (driver handles reconnects; adding retries can cause double-writes).

**Jitter** prevents retry storms when many goroutines fail simultaneously.

```go
// pkg/resilience/retry.go
type RetryConfig struct {
    MaxRetries  int           // number of retries (not total attempts)
    BaseBackoff time.Duration // base sleep before first retry (doubles each attempt)
    MaxBackoff  time.Duration // cap on sleep duration
}

var DefaultRetryConfig = RetryConfig{
    MaxRetries:  3,
    BaseBackoff: 100 * time.Millisecond,
    MaxBackoff:  2 * time.Second,
}

func Retry(ctx context.Context, cfg RetryConfig, fn func() error) error {
    var lastErr error
    for attempt := 0; attempt <= cfg.MaxRetries; attempt++ {
        if err := fn(); err != nil {
            lastErr = err
            if attempt == cfg.MaxRetries {
                break
            }
            backoff := cfg.BaseBackoff * (1 << uint(attempt))
            if backoff > cfg.MaxBackoff {
                backoff = cfg.MaxBackoff
            }
            jitter := time.Duration(rand.Int63n(int64(backoff / 2)))
            select {
            case <-time.After(backoff + jitter):
            case <-ctx.Done():
                return fmt.Errorf("retry cancelled: %w", ctx.Err())
            }
            continue
        }
        return nil
    }
    return fmt.Errorf("after %d retries: %w", cfg.MaxRetries, lastErr)
}
```

Usage:

```go
err := resilience.Retry(ctx, resilience.DefaultRetryConfig, func() error {
    return paymentClient.Charge(ctx, orderID)
})
```

**When NOT to retry:** database writes (risk double-insert), non-idempotent operations without idempotency keys, errors that are clearly permanent (4xx client errors).

---

## Rate Limiting at Architecture Level

Rate limiting in a single Gin instance (middleware) is covered in `golang-gin-api/references/rate-limiting.md`. This section covers architectural decisions.

### Decision Matrix

| Scenario | Approach |
|---|---|
| Single binary, simple protection | In-memory token bucket (no Redis) |
| Multiple instances, IP-based limits | Redis token bucket with Lua |
| Per-user billing accuracy | Redis sliding window, key = user ID |
| Global API quota across all routes | API gateway (Kong, Traefik, nginx) |
| Rate limiting for paid tiers | Redis + tier config loaded from env |

### Per-Service Limits

Each internal service should define its own limits independently. Do not use a shared rate limiter across services — failure domains must stay isolated.

```go
// Per-route group limits — different SLAs for different endpoints
public := r.Group("/api/v1")
public.Use(middleware.MemoryRateLimiter(middleware.MemoryRateLimiterConfig{
    Rate: 10, Burst: 20,
}, done))

authenticated := r.Group("/api/v1")
authenticated.Use(authMiddleware, middleware.TieredRateLimiter(rdb, tiers, "rl:auth:"))
```

**Key design choices:**
- Key by user ID (from JWT `sub` claim) over IP — survives NAT and mobile IP changes
- Fail open on Redis errors — blocking all traffic because a cache is down is worse than briefly allowing excess
- Set `X-RateLimit-Remaining` on every response, not only on 429

---

## Pattern Comparison Table

| Pattern | Complexity Cost | When to Use | Simple Alternative |
|---|---|---|---|
| Circuit Breaker | LOW (2/5) | Any external dependency that can be slow/down | Timeout + retry backoff |
| Bulkhead | LOW (2/5) | One slow dep saturating goroutines | `context.WithTimeout` on slow calls |
| Retry + Backoff | LOW (1/5) | Any external call (default: always use) | Single call with timeout |
| Rate Limiting (distributed) | MEDIUM (3/5) | Multiple instances + per-user accuracy | In-memory token bucket (single node) |

**Reading the cost scale:** 1 = one file, no new dependencies. 5 = new infrastructure, new failure modes, weeks of learning curve.
