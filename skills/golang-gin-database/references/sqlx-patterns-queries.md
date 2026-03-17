# sqlx Patterns — Get, Select, NamedExec, Null Handling, and Dynamic Queries

The three core sqlx query methods, null type handling, and safe dynamic WHERE clause construction.

> **Architectural recommendation:** These are mainstream Go/sqlx community patterns, not part of the Gin framework API.

## Get, Select, NamedExec

| Method | Returns | Use for |
|--------|---------|---------|
| `GetContext` | single row into struct | `SELECT ... WHERE id = $1` |
| `SelectContext` | slice of structs | `SELECT ...` many rows |
| `NamedExecContext` | `sql.Result` | INSERT/UPDATE with named params |

```go
// GetContext — single row; returns sql.ErrNoRows if not found
var u userRow
err := db.GetContext(ctx, &u, `
    SELECT id, name, email, role, created_at, updated_at
    FROM users WHERE id = $1 AND deleted_at IS NULL
`, id)

// SelectContext — multiple rows
var rows []userRow
err := db.SelectContext(ctx, &rows, `
    SELECT id, name, email, role, created_at, updated_at
    FROM users WHERE deleted_at IS NULL
    ORDER BY created_at DESC LIMIT $1 OFFSET $2
`, limit, offset)

// NamedExecContext — named parameters from struct or map
row := userInsertRow{ID: newUUID(), Name: req.Name, Email: req.Email,
    PasswordHash: hashPassword(req.Password), Role: req.Role}
_, err := db.NamedExecContext(ctx, `
    INSERT INTO users (id, name, email, password_hash, role)
    VALUES (:id, :name, :email, :password_hash, :role)
`, row)
```

**Critical:** Never use `fmt.Sprintf` to build SQL strings — use `$N` placeholders or `:name` named params. This prevents SQL injection.

## Null Handling

```go
import "database/sql"

type userRowNullable struct {
    ID        string         `db:"id"`
    Name      string         `db:"name"`
    DeletedAt sql.NullTime   `db:"deleted_at"` // NULL when not deleted
    Bio       sql.NullString `db:"bio"`         // NULL when profile incomplete
    CreatedAt time.Time      `db:"created_at"`
}

func (r *userRowNullable) toDomain() *domain.User {
    u := &domain.User{ID: r.ID, Name: r.Name}
    if r.Bio.Valid {
        u.Bio = r.Bio.String
    }
    return u
}

// Alternative: github.com/guregu/null for ergonomic null types
import "gopkg.in/guregu/null.v4"

type userRowNullV2 struct {
    Bio null.String `db:"bio"` // ValueOrZero() returns "" if null
}
```

## Safe Dynamic Queries

Build dynamic WHERE clauses safely. Never string-concatenate user input.

```go
// Option A: manual accumulation — no extra dependency
func buildListQuery(opts domain.ListOptions) (string, []any) {
    conditions := []string{"deleted_at IS NULL"}
    args := []any{}
    argIdx := 1

    if opts.Role != "" {
        conditions = append(conditions, fmt.Sprintf("role = $%d", argIdx))
        args = append(args, opts.Role)
        argIdx++
    }

    limit := opts.Limit
    if limit <= 0 || limit > 100 {
        limit = 20
    }
    offset := (opts.Page - 1) * limit

    query := fmt.Sprintf(`
        SELECT id, name, email, role, created_at, updated_at
        FROM users WHERE %s ORDER BY created_at DESC LIMIT $%d OFFSET $%d
    `, strings.Join(conditions, " AND "), argIdx, argIdx+1)

    args = append(args, limit, offset)
    return query, args
}

// Option B: squirrel query builder (github.com/Masterminds/squirrel)
import sq "github.com/Masterminds/squirrel"

psql := sq.StatementBuilder.PlaceholderFormat(sq.Dollar)
qb := psql.Select("id", "name", "email", "role", "created_at", "updated_at").
    From("users").Where("deleted_at IS NULL")

if opts.Role != "" {
    qb = qb.Where(sq.Eq{"role": opts.Role})
}

sql, args, err := qb.OrderBy("created_at DESC").
    Limit(uint64(limit)).Offset(uint64(offset)).ToSql()
```
