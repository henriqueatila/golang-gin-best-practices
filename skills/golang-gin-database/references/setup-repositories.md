# Database Setup — Repository Snippets and Transaction Pattern

Thin GORM and sqlx repository stubs, context-based transaction pattern, and cursor pagination overview.

## GORM Repository Implementation (Stub)

```go
// internal/repository/user_repository_gorm.go
package repository

import (
    "context"
    "errors"

    "gorm.io/gorm"
    "myapp/internal/domain"
)

type gormUserRepository struct {
    db *gorm.DB
}

func NewUserRepository(db *gorm.DB) domain.UserRepository {
    return &gormUserRepository{db: db}
}

func (r *gormUserRepository) GetByID(ctx context.Context, id string) (*domain.User, error) {
    var m UserModel
    if err := r.txFromCtx(ctx).WithContext(ctx).First(&m, "id = ?", id).Error; err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, domain.ErrNotFound.New(err)
        }
        return nil, domain.ErrInternal.New(err)
    }
    return m.ToDomain(), nil
}
```

For the full GORM implementation (models, soft deletes, scopes, preloading, transactions, hooks):
see [references/gorm-patterns-models-crud.md](gorm-patterns-models-crud.md),
[references/gorm-patterns-queries.md](gorm-patterns-queries.md), and
[references/gorm-patterns-repository.md](gorm-patterns-repository.md).

## sqlx Repository Implementation (Stub)

```go
// internal/repository/user_repository_sqlx.go
package repository

import (
    "context"
    "database/sql"
    "errors"

    "github.com/jmoiron/sqlx"
    "myapp/internal/domain"
)

type sqlxUserRepository struct {
    db *sqlx.DB
}

func NewSqlxUserRepository(db *sqlx.DB) domain.UserRepository {
    return &sqlxUserRepository{db: db}
}

func (r *sqlxUserRepository) GetByID(ctx context.Context, id string) (*domain.User, error) {
    var u userRow
    err := r.db.GetContext(ctx, &u,
        `SELECT id, name, email, role, created_at, updated_at FROM users WHERE id = $1`, id)
    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, domain.ErrNotFound.New(err)
        }
        return nil, domain.ErrInternal.New(err)
    }
    return u.toDomain(), nil
}
```

For the full sqlx implementation (struct scanning, NamedExec, IN clauses, transactions):
see [references/sqlx-patterns-setup.md](sqlx-patterns-setup.md) and
[references/sqlx-patterns-repository.md](sqlx-patterns-repository.md).

## Transaction Pattern (Context-Based, GORM)

Pass `*gorm.DB` via context so repositories transparently participate in a transaction. The service orchestrates; repositories stay unaware.

```go
// internal/repository/tx.go
package repository

import (
    "context"
    "gorm.io/gorm"
)

type txKey struct{}

// WithTx stores a transaction in ctx so repositories can use it.
func WithTx(ctx context.Context, tx *gorm.DB) context.Context {
    return context.WithValue(ctx, txKey{}, tx)
}

// txFromCtx returns the transaction from ctx, or the default db.
func (r *gormUserRepository) txFromCtx(ctx context.Context) *gorm.DB {
    if tx, ok := ctx.Value(txKey{}).(*gorm.DB); ok {
        return tx
    }
    return r.db
}
```

Service-layer usage:

```go
// internal/service/user_service.go
func (s *userService) RegisterWithProfile(ctx context.Context, req domain.CreateUserRequest) error {
    return s.db.WithContext(ctx).Transaction(func(tx *gorm.DB) error {
        txCtx := repository.WithTx(ctx, tx)
        if err := s.userRepo.Create(txCtx, &user); err != nil {
            return err // tx rolled back automatically
        }
        return s.profileRepo.Create(txCtx, &profile)
    })
}
```

## Cursor / Keyset Pagination

Offset pagination (`LIMIT x OFFSET y`) degrades at large offsets because PostgreSQL must skip rows. **Keyset pagination** is O(log n) via an index seek — preferred for large or fast-growing tables.

```go
// domain/user.go
type CursorOptions struct {
    Cursor time.Time // created_at of last seen item; zero = first page
    Limit  int
}
```

See [gorm-patterns-queries.md](gorm-patterns-queries.md#cursor--keyset-pagination) for the full GORM implementation with index requirements and trade-off table.
