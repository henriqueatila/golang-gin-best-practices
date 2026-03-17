# sqlx Patterns — Setup and Basic Queries

Connection setup, struct scanning with db tags, Get/Select/NamedExec, safe dynamic queries, and null handling.

> **Architectural recommendation:** These are mainstream Go/sqlx community patterns, not part of the Gin framework API.

**sqlx vs GORM:** sqlx wraps `database/sql` with struct scanning and named queries. It executes raw SQL — no ORM magic, no hidden queries, full control. Choose sqlx when you need complex queries or existing DBA-crafted SQL.

## Connection Setup

```go
// internal/repository/db_sqlx.go
package repository

import (
    "fmt"
    "log/slog"
    "time"

    "github.com/jmoiron/sqlx"
    _ "github.com/lib/pq" // PostgreSQL driver — blank import registers it
)

type SqlxConfig struct {
    DSN             string
    MaxOpenConns    int
    MaxIdleConns    int
    ConnMaxLifetime time.Duration
}

// NewSqlxDB opens a PostgreSQL connection via sqlx.
func NewSqlxDB(cfg SqlxConfig, logger *slog.Logger) (*sqlx.DB, error) {
    db, err := sqlx.Connect("postgres", cfg.DSN)
    if err != nil {
        return nil, fmt.Errorf("sqlx.Connect: %w", err)
    }

    db.SetMaxOpenConns(cfg.MaxOpenConns)      // e.g. 25
    db.SetMaxIdleConns(cfg.MaxIdleConns)      // e.g. 10
    db.SetConnMaxLifetime(cfg.ConnMaxLifetime) // e.g. 5*time.Minute

    if err := db.Ping(); err != nil {
        return nil, fmt.Errorf("db.Ping: %w", err)
    }

    logger.Info("database connected (sqlx)")
    return db, nil
}
```

## Connection Pooling

Same recommendations as GORM (both use `database/sql` under the hood):

```go
db.SetMaxOpenConns(25)                    // max concurrent connections
db.SetMaxIdleConns(10)                    // warm idle pool
db.SetConnMaxLifetime(5 * time.Minute)    // recycle stale connections
db.SetConnMaxIdleTime(1 * time.Minute)    // close idle connections sooner
```

**Health check endpoint:**

```go
// internal/handler/health_handler.go
func (h *HealthHandler) Check(c *gin.Context) {
    if err := h.db.PingContext(c.Request.Context()); err != nil {
        c.JSON(http.StatusServiceUnavailable, gin.H{
            "status": "degraded",
            "db":     "unreachable",
        })
        return
    }
    c.JSON(http.StatusOK, gin.H{"status": "ok"})
}
```

## Struct Scanning with db Tags

Map SQL columns to struct fields using `db` tags. No GORM, no magic.

```go
// internal/repository/user_row.go
package repository

import (
    "time"
    "myapp/internal/domain"
)

// userRow is the sqlx scan target for the users table.
type userRow struct {
    ID        string    `db:"id"`
    Name      string    `db:"name"`
    Email     string    `db:"email"`
    Role      string    `db:"role"`
    CreatedAt time.Time `db:"created_at"`
    UpdatedAt time.Time `db:"updated_at"`
}

func (r *userRow) toDomain() *domain.User {
    return &domain.User{
        ID: r.ID, Name: r.Name, Email: r.Email,
        Role: r.Role, CreatedAt: r.CreatedAt, UpdatedAt: r.UpdatedAt,
    }
}

// userInsertRow includes fields only used on INSERT.
type userInsertRow struct {
    ID           string `db:"id"`
    Name         string `db:"name"`
    Email        string `db:"email"`
    PasswordHash string `db:"password_hash"`
    Role         string `db:"role"`
}
```

**Rule:** Column names in SQL must exactly match `db` tags. Mismatches produce silent zero values — always verify with a test query.

For Get/Select/NamedExec, null handling, and safe dynamic queries: see [sqlx-patterns-queries.md](sqlx-patterns-queries.md).
