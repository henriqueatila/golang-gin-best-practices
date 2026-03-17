# Migrations — Setup, CLI, and Library Usage

golang-migrate overview, file naming convention, CLI commands, library usage (run on startup), and startup vs CI/CD strategy.

> **Architectural recommendation:** golang-migrate is not part of the Gin framework. These are mainstream Go community patterns.

## golang-migrate Overview

`golang-migrate/migrate` runs versioned SQL migration files against a database. Each migration has an **up** file (apply) and a **down** file (rollback). Versions are tracked in a `schema_migrations` table.

```bash
# CLI
go install -tags 'postgres' github.com/golang-migrate/migrate/v4/cmd/migrate@latest

# Library (add to go.mod)
go get github.com/golang-migrate/migrate/v4
go get github.com/golang-migrate/migrate/v4/database/postgres
go get github.com/golang-migrate/migrate/v4/source/file
```

## File Naming Convention

```
db/migrations/
├── 000001_create_users_table.up.sql
├── 000001_create_users_table.down.sql
├── 000002_create_roles_table.up.sql
├── 000002_create_roles_table.down.sql
├── 000003_add_user_roles_junction.up.sql
└── 000003_add_user_roles_junction.down.sql
```

**Rules:**
- Prefix with a **zero-padded sequential integer** (`000001`, `000002`, …). golang-migrate sorts lexicographically — gaps are fine, but never reorder.
- Suffix: `.up.sql` for apply, `.down.sql` for rollback.
- Descriptive name in snake_case between version and suffix.
- **Never edit a migration once it has been applied to any environment.** Create a new migration to fix mistakes.

## CLI Usage

```bash
# Apply all pending migrations
migrate -path db/migrations -database "$DATABASE_URL" up

# Apply exactly N migrations
migrate -path db/migrations -database "$DATABASE_URL" up 2

# Roll back the most recent migration
migrate -path db/migrations -database "$DATABASE_URL" down 1

# Roll back all migrations (destructive — dev only)
migrate -path db/migrations -database "$DATABASE_URL" down

# Show current version and dirty state
migrate -path db/migrations -database "$DATABASE_URL" version

# Force a specific version (use when dirty=true after a failed migration)
migrate -path db/migrations -database "$DATABASE_URL" force 3

# Create a new migration file pair
migrate create -ext sql -dir db/migrations -seq add_profile_table
# Creates: 000004_add_profile_table.up.sql + 000004_add_profile_table.down.sql
```

**Critical:** If a migration fails halfway, the database is left in a "dirty" state (version N, dirty=true). Fix the schema manually, then run `force N` to clear the dirty flag before retrying.

## Library Usage (Run on Startup)

Run migrations automatically when the server starts. Suitable for simple deployments and development.

```go
// internal/repository/migrate.go
package repository

import (
    "errors"
    "fmt"
    "log/slog"

    "github.com/golang-migrate/migrate/v4"
    _ "github.com/golang-migrate/migrate/v4/database/postgres"
    _ "github.com/golang-migrate/migrate/v4/source/file"
)

// RunMigrations applies all pending up migrations.
// Call before starting the HTTP server.
func RunMigrations(dsn, migrationsPath string, logger *slog.Logger) error {
    m, err := migrate.New(
        fmt.Sprintf("file://%s", migrationsPath), // e.g. "file://db/migrations"
        dsn,
    )
    if err != nil {
        return fmt.Errorf("migrate.New: %w", err)
    }
    defer m.Close()

    if err := m.Up(); err != nil {
        if errors.Is(err, migrate.ErrNoChange) {
            logger.Info("migrations: no changes")
            return nil
        }
        return fmt.Errorf("migrate.Up: %w", err)
    }

    version, dirty, _ := m.Version()
    logger.Info("migrations applied", "version", version, "dirty", dirty)
    return nil
}
```

Wire into `main.go` before starting the router:

```go
func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
    dsn := os.Getenv("DATABASE_URL")
    migrationsPath := os.Getenv("MIGRATIONS_PATH")
    if migrationsPath == "" {
        migrationsPath = "db/migrations"
    }
    if err := repository.RunMigrations(dsn, migrationsPath, logger); err != nil {
        logger.Error("migration failed", "error", err)
        os.Exit(1)
    }
    // ... router setup
}
```

## Startup vs CI/CD Strategy

| Strategy | Best for | Pros | Cons |
|----------|----------|------|------|
| **Run on startup** | Dev, small teams | Zero-config, always up to date | Risk in multi-replica rollout (race) |
| **CI/CD step** | Production, teams | Explicit, auditable, no race | Requires separate migration job |
| **Init container** (K8s) | Kubernetes | Runs once before app pods | K8s-specific, adds complexity |

For Kubernetes Job YAML and GitHub Actions step: see [migrations-advanced.md](migrations-advanced.md).
