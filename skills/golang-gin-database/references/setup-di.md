# Database Setup — Retry, TLS, and Dependency Injection

ConnectWithRetry with exponential backoff, TLS/sslmode configuration, and dependency injection wiring in main.go.

## Connection with Retry (Startup)

```go
// ConnectWithRetry retries the connection with exponential backoff.
// Use during startup when the database container may not be ready yet.
func ConnectWithRetry(dsn string, maxRetries int) (*gorm.DB, error) {
    var db *gorm.DB
    var err error
    for i := 0; i < maxRetries; i++ {
        db, err = gorm.Open(postgres.Open(dsn), &gorm.Config{
            Logger: logger.Default.LogMode(logger.Warn),
        })
        if err == nil {
            sqlDB, err := db.DB()
            if err != nil {
                return nil, fmt.Errorf("failed to get underlying sql.DB: %w", err)
            }
            sqlDB.SetMaxOpenConns(25)
            sqlDB.SetMaxIdleConns(10)
            sqlDB.SetConnMaxLifetime(5 * time.Minute)
            sqlDB.SetConnMaxIdleTime(1 * time.Minute)
            if err := sqlDB.Ping(); err != nil {
                return nil, fmt.Errorf("database ping failed: %w", err)
            }
            return db, nil
        }
        backoff := time.Duration(1<<uint(i)) * time.Second
        slog.Warn("database connection failed, retrying",
            "attempt", i+1, "backoff", backoff, "error", err)
        time.Sleep(backoff)
    }
    return nil, fmt.Errorf("failed to connect after %d retries: %w", maxRetries, err)
}
```

## TLS / sslmode

```go
// Development (local Docker)
dsn := "host=localhost user=app password=secret dbname=myapp sslmode=disable"

// Production: verify the server certificate
dsn := "host=db.example.com user=app password=*** dbname=myapp sslmode=verify-full sslrootcert=/etc/ssl/certs/rds-ca.pem"
```

Always use `sslmode=verify-full` in production to prevent MITM attacks.

## Dependency Injection in main.go

Wire repositories → services → handlers. Nothing creates its own dependencies.

```go
// cmd/api/main.go
package main

import (
    "log/slog"
    "os"
    "time"

    "myapp/internal/handler"
    "myapp/internal/repository"
    "myapp/internal/service"
)

func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

    dbCfg := repository.Config{
        DSN:             os.Getenv("DATABASE_URL"),
        MaxOpenConns:    25,
        MaxIdleConns:    10,
        ConnMaxLifetime: 5 * time.Minute,
    }

    db, err := repository.NewGORMDB(dbCfg, logger)
    if err != nil {
        logger.Error("failed to connect to database", "error", err)
        os.Exit(1)
    }

    // Wire the dependency graph: repo → service → handler
    userRepo    := repository.NewUserRepository(db)
    userService := service.NewUserService(userRepo, logger)
    userHandler := handler.NewUserHandler(userService, logger)

    _ = userHandler // ... router setup, see gin-api skill
}
```

**Critical:** Read `DATABASE_URL` from environment, never hardcode credentials. See the **golang-gin-deploy** skill for Docker/Kubernetes secrets.
