# GORM Patterns — Transaction and Repository Implementation

Context-based transaction propagation and the complete `UserRepository` implementation.

> **Architectural recommendation:** These are mainstream Go/GORM community patterns, not part of the Gin framework API.

## Transaction Context Helpers

Pass `*gorm.DB` via context so repositories transparently participate in a service-layer transaction. The service orchestrates; repositories just call `txFromCtx`.

```go
// internal/repository/tx.go
package repository

import (
    "context"

    "gorm.io/gorm"
)

type txKey struct{}

// WithTx stores a *gorm.DB transaction in ctx.
// Call this in the service layer before passing ctx to repositories.
func WithTx(ctx context.Context, tx *gorm.DB) context.Context {
    return context.WithValue(ctx, txKey{}, tx)
}

// txFromCtx returns the transaction stored in ctx, or the repository's default db.
// Every repository write method should call this instead of using r.db directly.
func (r *gormUserRepository) txFromCtx(ctx context.Context) *gorm.DB {
    if tx, ok := ctx.Value(txKey{}).(*gorm.DB); ok {
        return tx
    }
    return r.db
}
```

## Complete Repository Implementation

Full `UserRepository` satisfying the `domain.UserRepository` interface. All methods use `txFromCtx` so they transparently participate in a service-layer transaction when one is present in context.

```go
// internal/repository/user_repository_gorm.go
package repository

import (
    "context"
    "fmt"
    "time"

    "gorm.io/gorm"
    "myapp/internal/domain"
)

type gormUserRepository struct {
    db *gorm.DB
}

// NewUserRepository returns a domain.UserRepository backed by GORM.
func NewUserRepository(db *gorm.DB) domain.UserRepository {
    return &gormUserRepository{db: db}
}

func (r *gormUserRepository) Create(ctx context.Context, user *domain.User) error {
    m := fromDomain(user)
    if err := r.txFromCtx(ctx).WithContext(ctx).Create(m).Error; err != nil {
        return mapGORMError(err)
    }
    user.ID = m.ID
    user.CreatedAt = m.CreatedAt
    user.UpdatedAt = m.UpdatedAt
    return nil
}

func (r *gormUserRepository) GetByID(ctx context.Context, id string) (*domain.User, error) {
    var m UserModel
    if err := r.txFromCtx(ctx).WithContext(ctx).First(&m, "id = ?", id).Error; err != nil {
        return nil, mapGORMError(err)
    }
    return m.ToDomain(), nil
}

func (r *gormUserRepository) GetByEmail(ctx context.Context, email string) (*domain.User, error) {
    var m UserModel
    if err := r.txFromCtx(ctx).WithContext(ctx).First(&m, "email = ?", email).Error; err != nil {
        return nil, mapGORMError(err)
    }
    return m.ToDomain(), nil
}

func (r *gormUserRepository) List(ctx context.Context, opts domain.ListOptions) ([]domain.User, int64, error) {
    var models []UserModel
    var total int64

    q := r.txFromCtx(ctx).WithContext(ctx).Model(&UserModel{}).Scopes(ByRole(opts.Role))

    if err := q.Count(&total).Error; err != nil {
        return nil, 0, domain.ErrInternal.New(err)
    }
    if err := q.Scopes(Paginate(opts.Page, opts.Limit)).Find(&models).Error; err != nil {
        return nil, 0, domain.ErrInternal.New(err)
    }

    users := make([]domain.User, len(models))
    for i, m := range models {
        users[i] = *m.ToDomain()
    }
    return users, total, nil
}

    // Update and Delete methods follow the same txFromCtx pattern.
    // Service-layer transaction usage: s.db.WithContext(ctx).Transaction(func(tx *gorm.DB) error {
    //     txCtx := repository.WithTx(ctx, tx)
    //     return s.userRepo.Create(txCtx, &user) // automatic rollback on error
    // })
}
