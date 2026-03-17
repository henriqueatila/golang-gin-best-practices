# Database Setup — Connection and Configuration

Repository interface pattern, GORM connection setup, retry with backoff, TLS/sslmode, and dependency injection wiring.

## Repository Interface Pattern

```go
// internal/domain/user.go
package domain

import (
    "context"
    "time"
)

type User struct {
    ID           string
    Name         string
    Email        string
    Role         string
    PasswordHash string    // set by service layer before persisting; never serialized to API responses
    CreatedAt    time.Time
    UpdatedAt    time.Time
}

type ListOptions struct {
    Page  int
    Limit int
    Role  string
}

// UserRepository defines the data access contract.
// Implementations live in internal/repository — GORM or sqlx, interchangeable.
type UserRepository interface {
    Create(ctx context.Context, user *User) error
    GetByID(ctx context.Context, id string) (*User, error)
    GetByEmail(ctx context.Context, email string) (*User, error)
    List(ctx context.Context, opts ListOptions) ([]User, int64, error)
    Update(ctx context.Context, user *User) error
    Delete(ctx context.Context, id string) error
}
```

**Why define it at the consumer?** The domain package does not import `gorm.io/gorm` or `jmoiron/sqlx`. Only the repository package does. This is the Dependency Inversion Principle applied to data access.

> **Architecture note:** Domain entities should not carry `json` or `binding` tags. Use separate request/response DTOs in the delivery layer. See **golang-gin-architect skill (`references/clean-architecture.md`)** Golden Rule 4.

## Database Connection Setup (GORM)

```go
// internal/repository/db.go
package repository

import (
    "fmt"
    "log/slog"
    "strings"
    "time"

    "gorm.io/driver/postgres"
    "gorm.io/gorm"
    "gorm.io/gorm/logger"
)

type Config struct {
    DSN             string
    MaxOpenConns    int
    MaxIdleConns    int
    ConnMaxLifetime time.Duration
    LogLevel        string // "silent" | "error" | "warn" | "info"
}

// NewGORMDB opens a PostgreSQL connection with connection pool settings.
func NewGORMDB(cfg Config, appLogger *slog.Logger) (*gorm.DB, error) {
    gormCfg := &gorm.Config{
        Logger: logger.Default.LogMode(gormLogLevel(cfg.LogLevel)),
    }

    db, err := gorm.Open(postgres.Open(cfg.DSN), gormCfg)
    if err != nil {
        return nil, fmt.Errorf("gorm.Open: %w", err)
    }

    sqlDB, err := db.DB()
    if err != nil {
        return nil, fmt.Errorf("db.DB: %w", err)
    }

    sqlDB.SetMaxOpenConns(cfg.MaxOpenConns)      // e.g. 25
    sqlDB.SetMaxIdleConns(cfg.MaxIdleConns)      // e.g. 10
    sqlDB.SetConnMaxLifetime(cfg.ConnMaxLifetime) // e.g. 5*time.Minute

    if err := sqlDB.Ping(); err != nil {
        return nil, fmt.Errorf("db.Ping: %w", err)
    }

    appLogger.Info("database connected")
    return db, nil
}

func gormLogLevel(level string) logger.LogLevel {
    switch strings.ToLower(level) {
    case "info":
        return logger.Info
    case "warn":
        return logger.Warn
    case "error":
        return logger.Error
    default:
        return logger.Silent
    }
}
```

Recommended pool settings:

| Setting           | Value     | Why                               |
|-------------------|-----------|-----------------------------------|
| MaxOpenConns      | 25        | Limits PostgreSQL connections     |
| MaxIdleConns      | 10        | Keeps a warm pool without waste   |
| ConnMaxLifetime   | 5 minutes | Prevents stale connections        |

For retry with backoff, TLS/sslmode, and dependency injection wiring: see [setup-di.md](setup-di.md).
