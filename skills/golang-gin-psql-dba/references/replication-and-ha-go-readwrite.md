# Replication and HA — Read/Write Splitting in Go

## Connection Struct

```go
// internal/db/connections.go
type Connections struct {
    primary *sqlx.DB
    replica *sqlx.DB
    logger  *slog.Logger
}

func (c *Connections) Primary() *sqlx.DB { return c.primary }
func (c *Connections) Replica() *sqlx.DB { return c.replica }
func (c *Connections) Close()            { c.primary.Close(); c.replica.Close() }

func NewConnections(primaryDSN, replicaDSN string, logger *slog.Logger) (*Connections, error) {
    primary, err := sqlx.Connect("postgres", primaryDSN)
    if err != nil { return nil, fmt.Errorf("primary connect: %w", err) }
    primary.SetMaxOpenConns(20); primary.SetMaxIdleConns(5)
    primary.SetConnMaxLifetime(5 * time.Minute); primary.SetConnMaxIdleTime(1 * time.Minute)

    replica, err := sqlx.Connect("postgres", replicaDSN)
    if err != nil { primary.Close(); return nil, fmt.Errorf("replica connect: %w", err) }
    replica.SetMaxOpenConns(40); replica.SetMaxIdleConns(10)  // replicas absorb more reads
    replica.SetConnMaxLifetime(5 * time.Minute); replica.SetConnMaxIdleTime(1 * time.Minute)

    return &Connections{primary: primary, replica: replica, logger: logger}, nil
}
```

## Repository Pattern: Routing by Operation

```go
// internal/repository/order_repo.go
package repository

import (
    "context"
    "fmt"

    "github.com/jmoiron/sqlx"
    appdb "myapp/internal/db"
)

type OrderRepo struct {
    conns *appdb.Connections
}

func NewOrderRepo(conns *appdb.Connections) *OrderRepo {
    return &OrderRepo{conns: conns}
}

// Create writes to primary — always consistent.
func (r *OrderRepo) Create(ctx context.Context, o *Order) error {
    const q = `
        INSERT INTO orders (id, user_id, total, status, created_at)
        VALUES (:id, :user_id, :total, :status, now())
        RETURNING created_at`
    rows, err := r.conns.Primary().NamedQueryContext(ctx, q, o)
    if err != nil {
        return fmt.Errorf("order create: %w", err)
    }
    defer rows.Close()
    if rows.Next() {
        return rows.Scan(&o.CreatedAt)
    }
    return nil
}

// List reads from replica — acceptable lag for list views.
func (r *OrderRepo) List(ctx context.Context, userID string, limit int) ([]Order, error) {
    const q = `
        SELECT id, user_id, total, status, created_at
        FROM orders
        WHERE user_id = $1
        ORDER BY created_at DESC
        LIMIT $2`
    var orders []Order
    if err := r.conns.Replica().SelectContext(ctx, &orders, q, userID, limit); err != nil {
        return nil, fmt.Errorf("order list: %w", err)
    }
    return orders, nil
}

// GetByID supports read-your-writes: pass strong=true after a write
// to ensure the caller sees what they just created.
func (r *OrderRepo) GetByID(ctx context.Context, id string, strong bool) (*Order, error) {
    db := r.conns.Replica()
    if strong {
        db = r.conns.Primary()
    }
    var o Order
    if err := db.GetContext(ctx, &o, `SELECT * FROM orders WHERE id = $1`, id); err != nil {
        return nil, fmt.Errorf("order get %s: %w", id, err)
    }
    return &o, nil
}
```

## Read-Your-Writes via Context Flag

Pass a context key so any layer can signal "use primary for this request":

```go
// internal/db/ctx.go
package db

import "context"

type ctxKey struct{}

// WithStrongRead marks the context to force primary reads.
func WithStrongRead(ctx context.Context) context.Context {
    return context.WithValue(ctx, ctxKey{}, true)
}

// IsStrongRead returns true if the context requires primary read.
func IsStrongRead(ctx context.Context) bool {
    v, _ := ctx.Value(ctxKey{}).(bool)
    return v
}
```

In the repository, replace the `strong bool` parameter:

```go
func (r *OrderRepo) pickDB(ctx context.Context) *sqlx.DB {
    if appdb.IsStrongRead(ctx) {
        return r.conns.Primary()
    }
    return r.conns.Replica()
}
```

In the handler, after a POST, tag the redirect context:

```go
ctx := appdb.WithStrongRead(c.Request.Context())
order, err := repo.GetByID(ctx, newOrderID, false)
```
