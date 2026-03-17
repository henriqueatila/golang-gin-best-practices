# GORM Patterns — Soft Deletes, Error Handling, and Hooks

Soft delete mechanics, domain error mapping (mapGORMError), and GORM hook trade-offs.

> **Architectural recommendation:** These are mainstream Go/GORM community patterns, not part of the Gin framework API.

## Soft Deletes

Adding `gorm.DeletedAt` to the model activates soft deletes automatically. GORM adds `WHERE deleted_at IS NULL` to all queries.

```go
type UserModel struct {
    // ...
    DeletedAt gorm.DeletedAt `gorm:"index"` // soft delete column
}

// Delete sets deleted_at = NOW() instead of removing the row
r.db.Delete(&UserModel{}, "id = ?", id)

// Query ignores soft-deleted rows by default
r.db.Find(&users) // WHERE deleted_at IS NULL

// Include soft-deleted rows
r.db.Unscoped().Find(&users)

// Hard delete (permanently remove)
r.db.Unscoped().Delete(&UserModel{}, "id = ?", id)
```

**Trade-off:** Soft deletes keep audit history but require `Unscoped()` everywhere you intentionally query deleted records. Consider a dedicated `archived_users` table for large datasets to avoid index bloat.

## Error Handling

Map GORM errors to domain errors. Never leak raw GORM errors to the handler layer.

```go
// internal/repository/errors.go
package repository

import (
    "errors"

    "github.com/jackc/pgx/v5/pgconn"
    "gorm.io/gorm"
    "myapp/internal/domain"
)

// mapGORMError converts a GORM error to a domain error.
func mapGORMError(err error) error {
    if errors.Is(err, gorm.ErrRecordNotFound) {
        return domain.ErrNotFound.New(err)
    }
    // PostgreSQL unique violation (SQLSTATE 23505) — typed assertion, not string matching
    var pgErr *pgconn.PgError
    if errors.As(err, &pgErr) && pgErr.Code == "23505" {
        return domain.ErrConflict.New(err)
    }
    return domain.ErrInternal.New(err)
}
```

**Critical:** Log the raw GORM error at the service layer (it includes the SQL state). Return only the domain error to the handler.

## Hooks — Trade-offs

GORM hooks (`BeforeCreate`, `AfterCreate`, etc.) run automatically but introduce hidden side effects.

```go
// Use hooks for truly cross-cutting concerns (e.g., UUID generation)
func (m *UserModel) BeforeCreate(tx *gorm.DB) error {
    if m.ID == "" {
        m.ID = uuid.Must(uuid.NewV7()).String() // UUIDv7: time-sortable, better B-tree index performance
    }
    return nil
}
```

| Pros | Cons |
|------|------|
| Zero boilerplate for UUID gen | Hidden logic — hard to trace |
| Consistent across all create paths | Can't be disabled per call |
| Works with transactions | Tested separately from service logic |

**Recommendation:** Use hooks for `ID` generation and `UpdatedAt` only. Move business logic (hashing passwords, sending emails) to the service layer where it's explicit and testable.
