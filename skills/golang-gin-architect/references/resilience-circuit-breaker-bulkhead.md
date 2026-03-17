# Resilience Patterns — Circuit Breaker and Bulkhead

Low-cost patterns for making Go Gin APIs resilient to external dependency failures.

**Default stance:** Add these to any service that calls external APIs. They cost little and prevent cascading failures.

---

## Circuit Breaker

### Gate — you need this when

- [ ] Your service calls an external API or service that can be slow or unavailable
- [ ] Timeouts alone would block goroutines too long under cascading failure

**Cost: LOW (2/5).** A 50-line wrapper on your HTTP client is worth it for any external dependency.

**Simpler alternative:** A generous `http.Client` timeout + retry with backoff. Add the circuit breaker once you observe that retries amplify load on the failing upstream.

### Implementation (`github.com/sony/gobreaker/v2`)

```go
// pkg/resilience/circuit_breaker.go
func NewCircuitBreaker(name string) *gobreaker.CircuitBreaker[[]byte] {
    return gobreaker.NewCircuitBreaker[[]byte](gobreaker.Settings{
        Name:        name,
        MaxRequests: 3,                // allow 3 requests in half-open state
        Interval:    60 * time.Second, // reset counts every 60s in closed state
        Timeout:     30 * time.Second, // wait 30s in open state before half-open
        ReadyToTrip: func(counts gobreaker.Counts) bool {
            return counts.ConsecutiveFailures > 5
        },
        OnStateChange: func(name string, from, to gobreaker.State) {
            slog.Warn("circuit breaker state changed",
                "name", name, "from", from.String(), "to", to.String())
        },
    })
}

func CallWithBreaker(ctx context.Context, cb *gobreaker.CircuitBreaker[[]byte], fn func() ([]byte, error)) ([]byte, error) {
    result, err := cb.Execute(func() ([]byte, error) {
        return fn()
    })
    if err != nil {
        return nil, fmt.Errorf("circuit breaker %s: %w", cb.Name(), err)
    }
    return result, nil
}
```

### Usage in a service

```go
type Client struct {
    http *http.Client
    cb   *gobreaker.CircuitBreaker[[]byte]
}

func NewClient() *Client {
    return &Client{
        http: &http.Client{Timeout: 5 * time.Second},
        cb:   resilience.NewCircuitBreaker("payment-api"),
    }
}

func (c *Client) Charge(ctx context.Context, orderID string) error {
    body, err := resilience.CallWithBreaker(ctx, c.cb, func() ([]byte, error) {
        req, _ := http.NewRequestWithContext(ctx, "POST", "https://payment.example.com/charge", nil)
        resp, err := c.http.Do(req)
        if err != nil {
            return nil, err
        }
        defer resp.Body.Close()
        if resp.StatusCode >= 500 {
            return nil, fmt.Errorf("upstream error: %d", resp.StatusCode)
        }
        return io.ReadAll(resp.Body)
    })
    _ = body
    return err
}
```

---

## Bulkhead Pattern

### Gate — you need this when

- [ ] One slow dependency is causing goroutine saturation affecting unrelated parts of your service
- [ ] You can identify 2+ distinct dependency groups with different SLA expectations

**Cost: LOW (2/5).** A semaphore is 5 lines of Go.

**Simpler alternative:** Set an aggressive `context.WithTimeout` on the slow call. Add bulkhead when you need concurrent capacity control, not just timeout protection.

### Implementation

```go
// pkg/resilience/bulkhead.go
type Bulkhead struct {
    name string
    sem  chan struct{}
}

func NewBulkhead(name string, maxConcurrent int) *Bulkhead {
    return &Bulkhead{name: name, sem: make(chan struct{}, maxConcurrent)}
}

func (b *Bulkhead) Execute(ctx context.Context, fn func() error) error {
    select {
    case b.sem <- struct{}{}:
        defer func() { <-b.sem }()
        return fn()
    case <-ctx.Done():
        return fmt.Errorf("bulkhead %s: %w", b.name, ctx.Err())
    }
}
```

### Usage — isolate database from external API calls

```go
var (
    dbBulkhead  = resilience.NewBulkhead("database", 50)
    apiBulkhead = resilience.NewBulkhead("payment-api", 10)
)

func (s *OrderService) CreateOrder(ctx context.Context, cmd order.CreateOrderCommand) (string, error) {
    var id string
    err := dbBulkhead.Execute(ctx, func() error {
        var e error
        id, e = s.repo.Create(ctx, cmd)
        return e
    })
    return id, err
}
```

---

## See Also

- `resilience-retry-rate-limiting.md` — retry with exponential backoff, rate limiting architecture, pattern comparison table
