# GORM Patterns — Models and CRUD

Model definition, soft deletes, CRUD operations, and error mapping for GORM repositories.

> **Architectural recommendation:** These are mainstream Go/GORM community patterns, not part of the Gin framework API.

## Model Definition

GORM maps struct fields to columns via tags. Keep models in `internal/repository` (not `internal/domain`) to avoid leaking GORM into the domain layer.

```go
// internal/repository/models.go
package repository

import (
    "time"

    "github.com/google/uuid"
    "gorm.io/gorm"
    "myapp/internal/domain"
)

// UserModel is the GORM representation of the users table.
// It stays in the repository layer — convert to/from domain.User via methods.
type UserModel struct {
    ID           string         `gorm:"type:uuid;primaryKey;default:gen_random_uuid()"`
    Name         string         `gorm:"type:varchar(100);not null"`
    Email        string         `gorm:"type:varchar(255);uniqueIndex;not null"`
    PasswordHash string         `gorm:"type:varchar(255);not null"`
    Role         string         `gorm:"type:varchar(50);not null;default:'user'"`
    CreatedAt    time.Time      `gorm:"autoCreateTime"`
    UpdatedAt    time.Time      `gorm:"autoUpdateTime"`
    DeletedAt    gorm.DeletedAt `gorm:"index"` // enables soft delete
}

// TableName overrides the default table name.
func (UserModel) TableName() string { return "users" }

// ToDomain converts a GORM model to the domain entity.
func (m *UserModel) ToDomain() *domain.User {
    return &domain.User{
        ID:        m.ID,
        Name:      m.Name,
        Email:     m.Email,
        Role:      m.Role,
        CreatedAt: m.CreatedAt,
        UpdatedAt: m.UpdatedAt,
    }
}

// fromDomain converts a domain entity to a GORM model.
func fromDomain(u *domain.User) *UserModel {
    return &UserModel{
        ID:           u.ID,
        Name:         u.Name,
        Email:        u.Email,
        PasswordHash: u.PasswordHash, // set by service layer before calling repo
        Role:         u.Role,
    }
}
```

**Why separate model and domain entity?** The domain layer must not import `gorm.io/gorm`. Separation lets you change GORM tags, add database-only fields (like `PasswordHash`), or swap ORMs without touching domain logic.

## CRUD Operations

```go
// Create — sets CreatedAt/UpdatedAt automatically
func (r *gormUserRepository) Create(ctx context.Context, user *domain.User) error {
    m := fromDomain(user)
    if err := r.txFromCtx(ctx).WithContext(ctx).Create(m).Error; err != nil {
        return mapGORMError(err)
    }
    user.ID = m.ID // GORM sets the generated UUID back
    user.CreatedAt = m.CreatedAt
    user.UpdatedAt = m.UpdatedAt
    return nil
}

// GetByID — returns domain.ErrNotFound when row is missing
func (r *gormUserRepository) GetByID(ctx context.Context, id string) (*domain.User, error) {
    var m UserModel
    if err := r.txFromCtx(ctx).WithContext(ctx).First(&m, "id = ?", id).Error; err != nil {
        return nil, mapGORMError(err)
    }
    return m.ToDomain(), nil
}

// Update — only updates non-zero fields with Save; use Updates for partial
func (r *gormUserRepository) Update(ctx context.Context, user *domain.User) error {
    result := r.txFromCtx(ctx).WithContext(ctx).
        Model(&UserModel{}).
        Where("id = ?", user.ID).
        Updates(map[string]any{
            "name":       user.Name,
            "role":       user.Role,
            "updated_at": time.Now(),
        })
    if result.Error != nil {
        return mapGORMError(result.Error)
    }
    if result.RowsAffected == 0 {
        return domain.ErrNotFound.New(fmt.Errorf("user %s not found", user.ID))
    }
    return nil
}

// Delete — soft delete if DeletedAt is in the model, hard delete otherwise
func (r *gormUserRepository) Delete(ctx context.Context, id string) error {
    result := r.txFromCtx(ctx).WithContext(ctx).Delete(&UserModel{}, "id = ?", id)
    if result.Error != nil {
        return domain.ErrInternal.New(result.Error)
    }
    if result.RowsAffected == 0 {
        return domain.ErrNotFound.New(fmt.Errorf("user %s not found", id))
    }
    return nil
}
```

For soft deletes, error handling (mapGORMError), and hooks: see [gorm-patterns-errors.md](gorm-patterns-errors.md).
