# sqlx Patterns — Transactions, IN Clauses, and Query Constants

Transactions with sqlx.Tx, sqlx.In for IN clauses, and SQL query constants.

> **Architectural recommendation:** These are mainstream Go/sqlx community patterns, not part of the Gin framework API.

## Transactions with sqlx.Tx

```go
// internal/repository/tx_sqlx.go
package repository

import (
    "context"
    "fmt"

    "github.com/jmoiron/sqlx"
)

// WithTransaction runs fn inside a transaction, rolling back on error.
func WithTransaction(ctx context.Context, db *sqlx.DB, fn func(*sqlx.Tx) error) error {
    tx, err := db.BeginTxx(ctx, nil)
    if err != nil {
        return fmt.Errorf("begin tx: %w", err)
    }

    if err := fn(tx); err != nil {
        if rbErr := tx.Rollback(); rbErr != nil {
            return fmt.Errorf("tx rollback failed: %v (original: %w)", rbErr, err)
        }
        return err
    }

    return tx.Commit()
}

// Service-layer usage:
func (s *userService) RegisterWithProfile(ctx context.Context, req domain.CreateUserRequest) error {
    return repository.WithTransaction(ctx, s.db, func(tx *sqlx.Tx) error {
        if err := s.userRepo.CreateTx(ctx, tx, &user); err != nil {
            return err
        }
        return s.profileRepo.CreateTx(ctx, tx, &profile)
    })
}

// Repository method accepting a tx — dual interface pattern
func (r *sqlxUserRepository) CreateTx(ctx context.Context, tx *sqlx.Tx, user *domain.User) error {
    row := toInsertRow(user)
    _, err := tx.NamedExecContext(ctx, insertUserSQL, row)
    if err != nil {
        return mapSqlxError(err)
    }
    return nil
}
```

**Pattern choice:** Passing `*sqlx.Tx` explicitly is simpler than context-based tx propagation for sqlx. The context approach (storing tx in ctx) works but requires type assertions at every repository method.

## sqlx.In for IN Clauses

`database/sql` does not expand `IN (?)` for slices — sqlx.In does.

```go
func (r *sqlxUserRepository) GetByIDs(ctx context.Context, ids []string) ([]domain.User, error) {
    if len(ids) == 0 {
        return nil, nil
    }

    // sqlx.In rewrites "IN (?)" → "IN ($1,$2,$3,...)"
    query, args, err := sqlx.In(`
        SELECT id, name, email, role, created_at, updated_at
        FROM users WHERE id IN (?) AND deleted_at IS NULL
    `, ids)
    if err != nil {
        return nil, domain.ErrInternal.New(err)
    }

    // Rebind converts ? placeholders to $N for PostgreSQL
    query = r.db.Rebind(query)

    var rows []userRow
    if err := r.db.SelectContext(ctx, &rows, query, args...); err != nil {
        return nil, domain.ErrInternal.New(err)
    }

    users := make([]domain.User, len(rows))
    for i, row := range rows {
        users[i] = *row.toDomain()
    }
    return users, nil
}
```

**Warning:** Large IN clauses (1000+ IDs) can hurt performance. For bulk lookups, prefer a temporary table or `ANY($1::uuid[])` with a PostgreSQL array parameter.

## Query Constants

Keep SQL in constants for discoverability and grep-ability.

```go
// internal/repository/user_queries.go
package repository

const (
    insertUserSQL = `
        INSERT INTO users (id, name, email, password_hash, role, created_at, updated_at)
        VALUES (:id, :name, :email, :password_hash, :role, NOW(), NOW())
    `
    selectUserByIDSQL = `
        SELECT id, name, email, role, created_at, updated_at
        FROM users WHERE id = $1 AND deleted_at IS NULL
    `
    selectUserByEmailSQL = `
        SELECT id, name, email, role, created_at, updated_at
        FROM users WHERE email = $1 AND deleted_at IS NULL
    `
    updateUserSQL = `
        UPDATE users SET name = :name, role = :role, updated_at = NOW()
        WHERE id = :id AND deleted_at IS NULL
    `
    softDeleteUserSQL = `
        UPDATE users SET deleted_at = NOW() WHERE id = $1 AND deleted_at IS NULL
    `
)
```

**Why constants?** Grep-able, type-checkable at compile time, and easy to test in isolation.

For the complete repository struct implementation: see [sqlx-patterns-full-repository.md](sqlx-patterns-full-repository.md).
