# Data Ownership — Data Sync Strategies and Migration Path

Companion to `data-ownership-boundaries.md` (when to split, database-per-service, API composition).

---

## Data Sync Strategies

| Strategy | Use when | Consistency | Complexity |
|---|---|---|---|
| API call on demand | Low traffic, freshness required | Strong | LOW |
| Event-driven (message broker) | Eventually consistent is acceptable | Eventual | MEDIUM |
| CDC (Change Data Capture) | Real-time sync without app changes | Eventual | HIGH |
| Materialized view (CQRS) | Read-heavy, complex joins across services | Eventual | HIGH |

**Default:** API call on demand. Move to event-driven only when measured latency or coupling is a real problem.

### Event-driven example (order.created)

```go
// Order service — publish after successful insert
func (s *OrderService) Create(ctx context.Context, cmd CreateOrderCmd) (*Order, error) {
    order, err := s.repo.Insert(ctx, cmd)
    if err != nil {
        return nil, fmt.Errorf("OrderService.Create: %w", err)
    }
    evt := OrderCreatedEvent{OrderID: order.ID, UserID: order.UserID, Total: order.Total}
    if err := s.publisher.Publish(ctx, "order.created", evt); err != nil {
        // Log and continue — use outbox pattern for guaranteed delivery
        slog.ErrorContext(ctx, "publish order.created", "err", err, "order_id", order.ID)
    }
    return order, nil
}

// User service — consume order.created to update local count
func (c *OrderCountConsumer) Handle(ctx context.Context, msg OrderCreatedEvent) error {
    if err := c.repo.IncrementOrderCount(ctx, msg.UserID); err != nil {
        return fmt.Errorf("OrderCountConsumer.Handle: %w", err)
    }
    return nil
}
```

For guaranteed delivery, combine with the Transactional Outbox pattern (see `data-patterns-saga-outbox.md`).

---

## Shared Reference Data

Reference data: countries, currencies, product categories, feature flags. Low write, high read frequency.

| Option | When to use | Trade-off |
|---|---|---|
| Each service keeps a copy (synced via events) | High read volume, tolerates eventual consistency | Stale data window |
| Dedicated reference data service | Many services need it, central ownership matters | Extra hop; single point of failure |
| Config / static data in code | Data rarely changes (< once per release) | Redeploy to update; zero latency |

**Option C — embedded static data (prefer this first):**

```go
// internal/reference/currency.go
var Currencies = map[string]string{
    "BRL": "Brazilian Real",
    "USD": "US Dollar",
    "EUR": "Euro",
}

func ValidCurrency(code string) bool {
    _, ok := Currencies[code]
    return ok
}
```

**Option A — local copy synced via events:**

```go
type CurrencyCache struct {
    mu   sync.RWMutex
    data map[string]string
}

func (c *CurrencyCache) Set(code, name string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.data[code] = name
}

func (c *CurrencyCache) Get(ctx context.Context, code string) (string, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    name, ok := c.data[code]
    return name, ok
}
```

---

## Migration Path: Monolith to Split

Migrate incrementally. Never attempt a big-bang split.

**Step 1 — Single shared PostgreSQL** (start here)

**Step 2 — Separate schemas (same PostgreSQL instance)**

```sql
CREATE SCHEMA IF NOT EXISTS users;
CREATE SCHEMA IF NOT EXISTS orders;
ALTER TABLE users SET SCHEMA users;
ALTER TABLE orders SET SCHEMA orders;
```

Use schema prefix in repositories:

```go
const schema = "users"

func (r *UserRepository) GetByID(ctx context.Context, id int64) (*domain.User, error) {
    query := fmt.Sprintf(`SELECT id, email, name FROM %s.users WHERE id = $1`, schema)
    // ...
}
```

**Step 3 — Replace cross-schema JOINs with service calls**

Before: `SELECT o.id, u.email FROM orders.orders o JOIN users.users u ON u.id = o.user_id`

After:
```go
order, _ := orderRepo.GetByID(ctx, orderID)
user, _  := userClient.GetByID(ctx, order.UserID)
```

**Step 4 — Move schemas to separate databases**

```go
// cmd/user-service/main.go
db, err := sql.Open("pgx", os.Getenv("USER_DB_DSN"))

// cmd/order-service/main.go
db, err := sql.Open("pgx", os.Getenv("ORDER_DB_DSN"))
```

No application code changes if repositories use the schema constant.

**Step 5 — Deploy as separate services** with own binary, own database, communicate via HTTP/gRPC.
