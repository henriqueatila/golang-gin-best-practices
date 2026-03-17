# Data Ownership — Service Boundaries and API Composition

How services own and share data in Go architectures. Load when evaluating service extraction or deciding on data boundaries.

> **Default stance:** Start with a single database. Split ONLY when you have a genuine reason. Premature data splitting creates distributed transaction nightmares.

---

## When to Split Data

### Gate — VERY HIGH cost (5/5)

```
START: Should this module have its own database?
  ├── Same team deploys everything?
  │     └── Shared DB is fine. STOP.
  └── Different teams, different deploy cycles?
        ├── Data is tightly coupled (JOINs between modules)?
        │     └── Shared DB. Splitting will hurt more than help.
        └── Data has clear boundaries (no cross-module JOINs)?
              ├── Different scaling needs?
              │     └── Split → Database-per-service.
              └── Same scaling needs?
                    └── Schema-per-service (same PostgreSQL, different schemas).
```

**Premature split signals:** you need `JOIN` between modules; one team owns both; both deploy together; no profiled bottleneck.

**Justified split signals:** teams blocked on each other's migrations; one module needs different storage technology; 100x write load differential; regulatory data isolation requirement.

---

## Database-Per-Service Pattern

```
┌──────────────────┐     ┌──────────────────┐
│   User Service   │     │  Order Service   │
└────────┬─────────┘     └────────┬─────────┘
         │                        │
┌────────▼─────────┐     ┌────────▼─────────┐
│    user_db       │     │    order_db      │
└──────────────────┘     └──────────────────┘
```

Rules: Service A NEVER queries Service B's database. All cross-service data access goes through APIs. Each service owns its schema and migrations. No foreign keys across service boundaries.

```go
type UserRepository struct{ db *sql.DB }

func (r *UserRepository) GetByID(ctx context.Context, id int64) (*domain.User, error) {
    const q = `SELECT id, email, name FROM users WHERE id = $1`
    row := r.db.QueryRowContext(ctx, q, id)
    var u domain.User
    if err := row.Scan(&u.ID, &u.Email, &u.Name); err != nil {
        return nil, fmt.Errorf("UserRepository.GetByID: %w", err)
    }
    return &u, nil
}
```

---

## API Composition (Fan-out with errgroup)

When a client needs data from multiple services, use an API gateway or BFF. Never let the client call multiple services directly.

```go
func (h *OrderDetailsHandler) Get(c *gin.Context) {
    orderID := c.Param("id")

    order, err := h.orders.GetByID(c.Request.Context(), orderID)
    if err != nil {
        c.JSON(http.StatusBadGateway, gin.H{"error": "order service unavailable"})
        return
    }

    g, ctx := errgroup.WithContext(c.Request.Context())
    var user *client.User
    g.Go(func() error {
        var err error
        user, err = h.users.GetByID(ctx, order.UserID)
        return err
    })

    if err := g.Wait(); err != nil {
        c.JSON(http.StatusBadGateway, gin.H{"error": "enrichment failed"})
        return
    }

    c.JSON(http.StatusOK, gin.H{"order": order, "user": user})
}
```

Typed HTTP client interface (testable):

```go
type OrderClient interface {
    GetByID(ctx context.Context, id string) (*Order, error)
}

func (c *httpOrderClient) GetByID(ctx context.Context, id string) (*Order, error) {
    req, err := http.NewRequestWithContext(ctx, http.MethodGet, c.base+"/orders/"+id, nil)
    if err != nil {
        return nil, fmt.Errorf("OrderClient.GetByID build request: %w", err)
    }
    resp, err := c.http.Do(req)
    if err != nil {
        return nil, fmt.Errorf("OrderClient.GetByID: %w", err)
    }
    defer resp.Body.Close()
    if resp.StatusCode != http.StatusOK {
        return nil, fmt.Errorf("OrderClient.GetByID: upstream returned %d", resp.StatusCode)
    }
    var o Order
    if err := json.NewDecoder(resp.Body).Decode(&o); err != nil {
        return nil, fmt.Errorf("OrderClient.GetByID decode: %w", err)
    }
    return &o, nil
}
```

---

## See Also

- `data-ownership-sync-migration.md` — data sync strategies, shared reference data, migration path from monolith to split
