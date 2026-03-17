# API Design — Versioning and Pagination

Deep-dive reference for API versioning and pagination patterns in Go Gin APIs.

---

## Versioning Strategies

### URL Path Versioning (Recommended)

Simplest, most visible, easiest to test and debug.

```go
func registerRoutes(r *gin.Engine, v1 V1Handlers, v2 V2Handlers) {
    apiV1 := r.Group("/api/v1")
    {
        apiV1.GET("/users", v1.ListUsers)
        apiV1.GET("/users/:id", v1.GetUser)
        apiV1.POST("/users", v1.CreateUser)
    }

    apiV2 := r.Group("/api/v2")
    {
        apiV2.GET("/users", v2.ListUsers)   // new pagination format
        apiV2.GET("/users/:id", v2.GetUser) // expanded response
        apiV2.POST("/users", v2.CreateUser) // same as v1
    }
}
```

**When to bump the version:** removing a field, changing a field type, changing field meaning, changing pagination format, removing an endpoint.

**When NOT to bump:** adding a new field, adding a new endpoint, adding a new optional query param, fixing a bug.

Header versioning (`Accept: application/vnd.myapp.v2+json`) — harder to test, harder to cache. Avoid unless you have a strong reason. Query param versioning (`?version=2`) — pollutes query strings, caching nightmares. Don't use.

---

## Cursor-Based Pagination (Recommended for Large Sets)

Best for feeds, timelines, any list that changes frequently. No "page drift" problem.

```go
// GET /api/v1/orders?cursor=eyJpZCI6MTAwfQ&limit=20
type CursorQuery struct {
    Cursor string `form:"cursor"`
    Limit  int    `form:"limit" binding:"min=1,max=100"`
}

type CursorPage[T any] struct {
    Data       []T    `json:"data"`
    NextCursor string `json:"next_cursor,omitempty"` // empty = last page
    HasMore    bool   `json:"has_more"`
}

func (h *OrderHandler) List(c *gin.Context) {
    q := CursorQuery{Limit: 20}
    if err := c.ShouldBindQuery(&q); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    orders, nextCursor, err := h.svc.ListOrders(c.Request.Context(), q.Cursor, q.Limit)
    if err != nil {
        handleServiceError(c, err, h.logger)
        return
    }
    c.JSON(http.StatusOK, CursorPage[domain.Order]{
        Data: orders, NextCursor: nextCursor, HasMore: nextCursor != "",
    })
}
```

Cursor implementation in repository:

```go
func (r *OrderRepo) List(ctx context.Context, cursor string, limit int) ([]domain.Order, string, error) {
    var lastID int64
    if cursor != "" {
        decoded, err := base64.StdEncoding.DecodeString(cursor)
        if err != nil {
            return nil, "", fmt.Errorf("invalid cursor: %w", err)
        }
        lastID, _ = strconv.ParseInt(string(decoded), 10, 64)
    }

    var orders []domain.Order
    if err := r.db.SelectContext(ctx, &orders,
        `SELECT id, user_id, status, total, created_at FROM orders
         WHERE ($1 = 0 OR id < $1) ORDER BY id DESC LIMIT $2`,
        lastID, limit+1); err != nil {
        return nil, "", fmt.Errorf("list orders: %w", err)
    }

    var nextCursor string
    if len(orders) > limit {
        lastOrder := orders[limit]
        nextCursor = base64.StdEncoding.EncodeToString(
            []byte(strconv.FormatInt(lastOrder.ID, 10)))
        orders = orders[:limit]
    }
    return orders, nextCursor, nil
}
```

---

## Offset-Based Pagination (OK for Small Sets)

Best for admin dashboards, internal tools, data < 10K rows.

```go
// GET /api/v1/users?page=1&limit=20
type OffsetQuery struct {
    Page  int `form:"page"  binding:"min=1"`
    Limit int `form:"limit" binding:"min=1,max=100"`
}

type OffsetPage[T any] struct {
    Data       []T `json:"data"`
    Page       int `json:"page"`
    Limit      int `json:"limit"`
    TotalCount int `json:"total_count"`
    TotalPages int `json:"total_pages"`
}
```

**Warning:** `OFFSET 100000` scans 100K rows. Switch to cursor-based when data grows.

---

## Quick Decision: Which Pagination?

```
START: How many total records?
  ├── < 10K → Offset pagination is fine
  └── > 10K → Is the data frequently changing (inserts/deletes)?
      ├── No → Offset still OK (but monitor performance)
      └── Yes → Cursor-based pagination
```

---

## See Also

- `api-design-filtering-bulk-evolution.md` — filtering, sorting, bulk ops, partial updates, deprecation, backwards compatibility, error contract
