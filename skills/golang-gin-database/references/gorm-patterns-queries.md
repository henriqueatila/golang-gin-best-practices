# GORM Patterns — Scopes and Pagination

Scopes, cursor/keyset pagination. For preloading, raw SQL, batch ops, and PostgreSQL features: see [gorm-patterns-advanced.md](gorm-patterns-advanced.md).

> **Architectural recommendation:** These are mainstream Go/GORM community patterns, not part of the Gin framework API.

## Scopes

Scopes encapsulate reusable query fragments.

```go
// internal/repository/scopes.go
package repository

import "gorm.io/gorm"

// ByRole filters users by role.
func ByRole(role string) func(*gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        if role == "" {
            return db
        }
        return db.Where("role = ?", role)
    }
}

// Paginate applies offset/limit pagination.
func Paginate(page, limit int) func(*gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        if page < 1 {
            page = 1
        }
        if limit < 1 || limit > 100 {
            limit = 20
        }
        return db.Offset((page - 1) * limit).Limit(limit)
    }
}

// Usage in List:
func (r *gormUserRepository) List(ctx context.Context, opts domain.ListOptions) ([]domain.User, int64, error) {
    var models []UserModel
    var total int64

    query := r.db.WithContext(ctx).Model(&UserModel{}).Scopes(ByRole(opts.Role))

    if err := query.Count(&total).Error; err != nil {
        return nil, 0, domain.ErrInternal.New(err)
    }

    if err := query.Scopes(Paginate(opts.Page, opts.Limit)).Find(&models).Error; err != nil {
        return nil, 0, domain.ErrInternal.New(err)
    }

    users := make([]domain.User, len(models))
    for i, m := range models {
        users[i] = *m.ToDomain()
    }
    return users, total, nil
}
```

## Cursor / Keyset Pagination

Offset pagination (`LIMIT x OFFSET y`) performs an O(n) skip. For large or append-heavy tables, use keyset pagination — a single index seek per page.

```go
// domain/user.go — add alongside existing ListOptions
type CursorOptions struct {
    Cursor time.Time // created_at of last item on previous page; zero = first page
    Limit  int
}

// internal/repository/user_repository_gorm.go
func (r *gormUserRepository) ListAfterCursor(ctx context.Context, opts domain.CursorOptions) ([]domain.User, error) {
    limit := opts.Limit
    if limit <= 0 || limit > 100 {
        limit = 20
    }

    var models []UserModel
    q := r.txFromCtx(ctx).WithContext(ctx).Order("created_at ASC").Limit(limit)
    if !opts.Cursor.IsZero() {
        q = q.Where("created_at > ?", opts.Cursor)
    }
    if err := q.Find(&models).Error; err != nil {
        return nil, domain.ErrInternal.New(err)
    }

    users := make([]domain.User, len(models))
    for i, m := range models {
        users[i] = *m.ToDomain()
    }
    return users, nil
}
```

**Index requirement:** `CREATE INDEX idx_users_created_at ON users(created_at ASC);` — the cursor column must be indexed for O(log n) performance.

**Returning the next cursor:** In the handler or service, take `CreatedAt` of the last returned item and encode it (ISO-8601 or Unix timestamp) as `next_cursor` in the response. The client sends it back on the next request.

| | Offset | Keyset |
|---|---|---|
| Complexity | Simple | Slightly more work |
| Performance at depth | O(offset) | O(log n) |
| Row stability | Drifts on insert/delete | Stable |
| Arbitrary page jump | Yes | No |

**Recommendation:** Use offset for small tables and admin UIs; keyset for feeds, audit logs, and any table that grows quickly.

For preloading associations, raw SQL, batch operations, and PostgreSQL-specific features (ON CONFLICT, RETURNING, arrays, JSONB): see [gorm-patterns-advanced.md](gorm-patterns-advanced.md).
