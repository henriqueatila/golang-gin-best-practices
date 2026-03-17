# sqlx Patterns — Complete UserRepository Implementation

Full `UserRepository` satisfying the `domain.UserRepository` interface. Drop-in replacement for the GORM version — swap in `main.go` without changing service code.

```go
// internal/repository/user_repository_sqlx.go
package repository

import (
    "context"
    "database/sql"
    "errors"
    "fmt"
    "strings"

    "github.com/google/uuid"
    "github.com/jmoiron/sqlx"
    "github.com/lib/pq"
    "myapp/internal/domain"
)

type sqlxUserRepository struct {
    db *sqlx.DB
}

// NewSqlxUserRepository returns a domain.UserRepository backed by sqlx.
func NewSqlxUserRepository(db *sqlx.DB) domain.UserRepository {
    return &sqlxUserRepository{db: db}
}

func (r *sqlxUserRepository) Create(ctx context.Context, user *domain.User) error {
    if user.ID == "" {
        user.ID = uuid.Must(uuid.NewV7()).String()
    }
    if user.Role == "" {
        user.Role = "user"
    }
    row := userInsertRow{
        ID: user.ID, Name: user.Name, Email: user.Email,
        PasswordHash: user.PasswordHash, Role: user.Role,
    }
    _, err := r.db.NamedExecContext(ctx, insertUserSQL, row)
    return mapSqlxError(err)
}

func (r *sqlxUserRepository) GetByID(ctx context.Context, id string) (*domain.User, error) {
    var row userRow
    if err := r.db.GetContext(ctx, &row, selectUserByIDSQL, id); err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, domain.ErrNotFound.New(err)
        }
        return nil, domain.ErrInternal.New(err)
    }
    return row.toDomain(), nil
}

func (r *sqlxUserRepository) GetByEmail(ctx context.Context, email string) (*domain.User, error) {
    var row userRow
    if err := r.db.GetContext(ctx, &row, selectUserByEmailSQL, email); err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, domain.ErrNotFound.New(err)
        }
        return nil, domain.ErrInternal.New(err)
    }
    return row.toDomain(), nil
}

func (r *sqlxUserRepository) List(ctx context.Context, opts domain.ListOptions) ([]domain.User, int64, error) {
    var total int64
    countQuery, countArgs := buildCountQuery(opts)
    if err := r.db.GetContext(ctx, &total, countQuery, countArgs...); err != nil {
        return nil, 0, domain.ErrInternal.New(err)
    }

    listQuery, listArgs := buildListQuery(opts)
    var rows []userRow
    if err := r.db.SelectContext(ctx, &rows, listQuery, listArgs...); err != nil {
        return nil, 0, domain.ErrInternal.New(err)
    }

    users := make([]domain.User, len(rows))
    for i, row := range rows {
        users[i] = *row.toDomain()
    }
    return users, total, nil
}

func (r *sqlxUserRepository) Update(ctx context.Context, user *domain.User) error {
    result, err := r.db.NamedExecContext(ctx, updateUserSQL, map[string]any{
        "id": user.ID, "name": user.Name, "role": user.Role,
    })
    if err != nil {
        return mapSqlxError(err)
    }
    rows, err := result.RowsAffected()
    if err != nil {
        return domain.ErrInternal.New(err)
    }
    if rows == 0 {
        return domain.ErrNotFound.New(fmt.Errorf("user %s not found", user.ID))
    }
    return nil
}

func (r *sqlxUserRepository) Delete(ctx context.Context, id string) error {
    result, err := r.db.ExecContext(ctx, softDeleteUserSQL, id)
    if err != nil {
        return domain.ErrInternal.New(err)
    }
    rows, err := result.RowsAffected()
    if err != nil {
        return domain.ErrInternal.New(err)
    }
    if rows == 0 {
        return domain.ErrNotFound.New(fmt.Errorf("user %s not found", id))
    }
    return nil
}

// mapSqlxError, buildCountQuery, buildListQuery: see sqlx-patterns-repository.md
```
