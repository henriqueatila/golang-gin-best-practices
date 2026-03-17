# Data Patterns — CQRS (Command Query Responsibility Segregation)

High-complexity pattern for Go Gin APIs with divergent read/write models.

Companion to `data-patterns-read-replicas.md` (read replica pool, replication lag handling).

**Default stance:** PostgreSQL + clean repositories handles more than you think.

---

## Gate — all must be true

- [ ] Read and write models are fundamentally different shapes (not just different subsets of the same row)
- [ ] Read volume is 10x+ write volume AND the read path is the bottleneck
- [ ] A single PostgreSQL primary + read replica is not enough
- [ ] Your team has at least 3 engineers who understand the pattern

**Simpler alternative first:** Separate `ReadUser()` / `WriteUser()` methods in the same repository. Add a PostgreSQL materialized view for denormalized reads.

```sql
-- Read-optimized view — refresh on schedule or via trigger
CREATE MATERIALIZED VIEW order_summary AS
SELECT o.id, o.user_id, u.name AS user_name,
       COUNT(oi.id) AS item_count,
       SUM(oi.price * oi.quantity) AS total_amount,
       o.status, o.created_at
FROM orders o
JOIN users u ON u.id = o.user_id
JOIN order_items oi ON oi.order_id = o.id
GROUP BY o.id, o.user_id, u.name, o.status, o.created_at;

CREATE UNIQUE INDEX ON order_summary(id);
```

---

## Go Implementation

```go
// internal/order/command.go — Write side: normalized domain model
package order

type CreateOrderCommand struct {
    UserID string
    Items  []OrderItemInput
}

type OrderWriteRepo interface {
    Create(ctx context.Context, userID string, items []OrderItemInput) (string, error)
}

type OrderCommandHandler struct{ repo OrderWriteRepo }

func (h *OrderCommandHandler) CreateOrder(ctx context.Context, cmd CreateOrderCommand) (string, error) {
    if len(cmd.Items) == 0 {
        return "", fmt.Errorf("order must have at least one item")
    }
    id, err := h.repo.Create(ctx, cmd.UserID, cmd.Items)
    if err != nil {
        return "", fmt.Errorf("create order: %w", err)
    }
    return id, nil
}
```

```go
// internal/order/query.go — Read side: denormalized view
package order

type OrderView struct {
    ID          string    `db:"id"`
    UserName    string    `db:"user_name"`
    ItemCount   int       `db:"item_count"`
    TotalAmount int64     `db:"total_amount"`
    Status      string    `db:"status"`
    CreatedAt   time.Time `db:"created_at"`
}

type OrderQueryHandler struct{ db *sqlx.DB }

func (h *OrderQueryHandler) ListOrders(ctx context.Context, userID string, page, limit int) ([]OrderView, error) {
    var views []OrderView
    q := `SELECT * FROM order_summary WHERE user_id = $1 ORDER BY created_at DESC LIMIT $2 OFFSET $3`
    if err := h.db.SelectContext(ctx, &views, q, userID, limit, (page-1)*limit); err != nil {
        return nil, fmt.Errorf("list orders: %w", err)
    }
    return views, nil
}
```

```go
// internal/order/handler.go — Wire both sides
type Handler struct {
    cmd   *OrderCommandHandler
    query *OrderQueryHandler
}

func (h *Handler) Create(c *gin.Context) {
    var cmd CreateOrderCommand
    if err := c.ShouldBindJSON(&cmd); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    id, err := h.cmd.CreateOrder(c.Request.Context(), cmd)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
        return
    }
    c.JSON(http.StatusCreated, gin.H{"id": id})
}
```

---

## Pattern Comparison

| Pattern | Complexity | When Prerequisites Are Met | Simple Alternative |
|---|---|---|---|
| CQRS | HIGH (4/5) | Read/write shapes diverge + 10x read volume + replica not enough | Separate read/write methods + materialized view |
| Read Replicas | MEDIUM (3/5) | Read/write ratio > 5:1 AND primary is measured bottleneck | Add indexes; run EXPLAIN ANALYZE |
