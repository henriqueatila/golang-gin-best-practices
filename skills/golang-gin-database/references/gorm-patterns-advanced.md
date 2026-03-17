# GORM Patterns — Preloading, Raw SQL, Batches, and PostgreSQL Features

Preloading associations, raw SQL, batch operations, and PostgreSQL-specific GORM features.

> **Architectural recommendation:** These are mainstream Go/GORM community patterns, not part of the Gin framework API.

## Preloading Associations

```go
// ProfileModel demonstrates a 1:1 association
type ProfileModel struct {
    ID     string `gorm:"type:uuid;primaryKey;default:gen_random_uuid()"`
    UserID string `gorm:"type:uuid;not null;uniqueIndex"`
    Bio    string `gorm:"type:text"`
}

type UserWithProfile struct {
    UserModel
    Profile ProfileModel `gorm:"foreignKey:UserID"`
}

// Preload loads Profile in a single additional query (N+1 safe)
func (r *gormUserRepository) GetWithProfile(ctx context.Context, id string) (*UserWithProfile, error) {
    var u UserWithProfile
    if err := r.db.WithContext(ctx).
        Preload("Profile").
        First(&u.UserModel, "id = ?", id).Error; err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, domain.ErrNotFound.New(err)
        }
        return nil, domain.ErrInternal.New(err)
    }
    return &u, nil
}

// Preload with condition
r.db.Preload("Orders", "status = ?", "active").Find(&users)
```

**Warning:** `Preload` issues one extra query per association, not one per row (GORM batches it). Avoid preloading large datasets — use `Joins` for filtering.

## Raw SQL

Use raw SQL for complex queries that GORM can't express cleanly.

```go
type UserStats struct {
    Role  string `gorm:"column:role"`
    Count int64  `gorm:"column:count"`
}

func (r *gormUserRepository) StatsByRole(ctx context.Context) ([]UserStats, error) {
    var stats []UserStats
    if err := r.db.WithContext(ctx).Raw(`
        SELECT role, COUNT(*) AS count FROM users
        WHERE deleted_at IS NULL GROUP BY role ORDER BY count DESC
    `).Scan(&stats).Error; err != nil {
        return nil, domain.ErrInternal.New(err)
    }
    return stats, nil
}

// Exec for mutations
func (r *gormUserRepository) BanUser(ctx context.Context, id string) error {
    result := r.db.WithContext(ctx).Exec(
        `UPDATE users SET role = 'banned', updated_at = NOW() WHERE id = ?`, id,
    )
    if result.Error != nil {
        return domain.ErrInternal.New(result.Error)
    }
    if result.RowsAffected == 0 {
        return domain.ErrNotFound.New(fmt.Errorf("user %s not found", id))
    }
    return nil
}
```

## Batch Operations

```go
// CreateInBatches — inserts in chunks to avoid parameter limit
func (r *gormUserRepository) BulkCreate(ctx context.Context, users []domain.User) error {
    models := make([]UserModel, len(users))
    for i, u := range users {
        models[i] = *fromDomain(&u)
    }
    if err := r.db.WithContext(ctx).CreateInBatches(models, 100).Error; err != nil {
        return domain.ErrInternal.New(err)
    }
    return nil
}

// FindInBatches — process large result sets without loading all rows into memory
func (r *gormUserRepository) ExportAll(ctx context.Context, process func([]domain.User) error) error {
    var models []UserModel
    return r.db.WithContext(ctx).Model(&UserModel{}).
        FindInBatches(&models, 200, func(tx *gorm.DB, batch int) error {
            users := make([]domain.User, len(models))
            for i, m := range models {
                users[i] = *m.ToDomain()
            }
            return process(users)
        }).Error
}
```

## PostgreSQL-Specific Features

```go
import (
    "gorm.io/gorm/clause"
    "github.com/lib/pq"   // pq.StringArray for array columns
    "gorm.io/datatypes"   // datatypes.JSON for JSONB columns
)

// ON CONFLICT DO NOTHING — idempotent insert
r.db.WithContext(ctx).Clauses(clause.OnConflict{DoNothing: true}).Create(&m)

// ON CONFLICT DO UPDATE (upsert)
r.db.WithContext(ctx).Clauses(clause.OnConflict{
    Columns:   []clause.Column{{Name: "email"}},
    DoUpdates: clause.AssignmentColumns([]string{"name", "updated_at"}),
}).Create(&m)

// RETURNING — get generated values after insert
r.db.WithContext(ctx).
    Clauses(clause.Returning{Columns: []clause.Column{{Name: "id"}, {Name: "created_at"}}}).
    Create(&m)

// Array and JSONB column types
type UserModel struct {
    Tags     pq.StringArray `gorm:"type:text[]"`
    Metadata datatypes.JSON `gorm:"type:jsonb"`
}
```
