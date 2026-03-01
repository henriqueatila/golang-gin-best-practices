# Migrations Reference

This file covers database migration management for Go/Gin APIs using `golang-migrate/migrate`: CLI and library usage, file naming conventions, zero-downtime migration patterns, startup vs CI/CD strategies, data seeding, rollback strategies, and example migrations for the users and roles tables. Use when setting up or evolving a PostgreSQL schema in a Gin project.

> **Architectural recommendation:** golang-migrate is not part of the Gin framework. These are mainstream Go community patterns.

## Table of Contents

1. [golang-migrate Overview](#golang-migrate-overview)
2. [File Naming Convention](#file-naming-convention)
3. [CLI Usage](#cli-usage)
4. [Library Usage (Run on Startup)](#library-usage-run-on-startup)
5. [Startup vs CI/CD Strategy](#startup-vs-cicd-strategy)
6. [Zero-Downtime Migrations](#zero-downtime-migrations)
7. [Seeding Data](#seeding-data)
8. [Rollback Strategies](#rollback-strategies)
9. [Example Migrations — Users and Roles](#example-migrations--users-and-roles)

---

## golang-migrate Overview

`golang-migrate/migrate` runs versioned SQL migration files against a database. Each migration has an **up** file (apply) and a **down** file (rollback). Versions are tracked in a `schema_migrations` table.

Install:

```bash
# CLI
go install -tags 'postgres' github.com/golang-migrate/migrate/v4/cmd/migrate@latest

# Library (add to go.mod)
go get github.com/golang-migrate/migrate/v4
go get github.com/golang-migrate/migrate/v4/database/postgres
go get github.com/golang-migrate/migrate/v4/source/file
```

---

## File Naming Convention

```
db/migrations/
├── 000001_create_users_table.up.sql
├── 000001_create_users_table.down.sql
├── 000002_create_roles_table.up.sql
├── 000002_create_roles_table.down.sql
├── 000003_add_user_roles_junction.up.sql
├── 000003_add_user_roles_junction.down.sql
└── 000004_add_users_deleted_at.up.sql
    000004_add_users_deleted_at.down.sql
```

**Rules:**
- Prefix with a **zero-padded sequential integer** (`000001`, `000002`, …). golang-migrate sorts lexicographically — gaps are fine, but never reorder.
- Suffix: `.up.sql` for apply, `.down.sql` for rollback.
- Descriptive name in snake_case between version and suffix.
- **Never edit a migration once it has been applied to any environment.** Create a new migration to fix mistakes.

---

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
# Creates: 000005_add_profile_table.up.sql + 000005_add_profile_table.down.sql
```

**Critical:** If a migration fails halfway, the database is left in a "dirty" state (version N, dirty=true). Fix the schema manually, then run `force N` to clear the dirty flag before retrying.

---

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
// cmd/api/main.go
func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

    dsn := os.Getenv("DATABASE_URL")
    migrationsPath := os.Getenv("MIGRATIONS_PATH") // default: "db/migrations"
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

---

## Startup vs CI/CD Strategy

| Strategy | Best for | Pros | Cons |
|----------|----------|------|------|
| **Run on startup** | Dev, small teams, simple deploys | Zero-config, always up to date | Risk in multi-replica rollout (race) |
| **CI/CD step** | Production, Kubernetes, teams | Explicit, auditable, no race | Requires separate migration job |
| **Init container** (K8s) | Kubernetes deployments | Runs once before app pods | K8s-specific, adds complexity |

**Recommended production pattern (Kubernetes):**

```yaml
# k8s/migrate-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: migrate/migrate:v4
          args:
            - -path=/migrations
            - -database=$(DATABASE_URL)
            - up
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: url
          volumeMounts:
            - name: migrations
              mountPath: /migrations
      volumes:
        - name: migrations
          configMap:
            name: db-migrations
```

**GitHub Actions CI/CD:**

```yaml
# .github/workflows/deploy.yml (migration step)
- name: Run migrations
  env:
    DATABASE_URL: ${{ secrets.DATABASE_URL }}
  run: |
    go install -tags 'postgres' github.com/golang-migrate/migrate/v4/cmd/migrate@latest
    migrate -path db/migrations -database "$DATABASE_URL" up
```

---

## Zero-Downtime Migrations

Schema changes that require zero downtime (no table locks, backward-compatible for rolling deploys).

### Column Addition (Safe)

```sql
-- 000005_add_users_bio.up.sql
-- Adding a nullable column is instant — no table rewrite in PostgreSQL 11+
ALTER TABLE users ADD COLUMN bio TEXT;
```

```sql
-- 000005_add_users_bio.down.sql
ALTER TABLE users DROP COLUMN bio;
```

### Column Rename (Multi-Step)

Never rename directly — old app code still reads the old name during rollout.

```sql
-- Step 1: add new column (deploy v1 — reads old column)
-- 000006_add_users_full_name.up.sql
ALTER TABLE users ADD COLUMN full_name VARCHAR(200);
UPDATE users SET full_name = name WHERE full_name IS NULL;

-- Step 2: make old column obsolete (deploy v2 — reads new column)
-- 000007_drop_users_name.up.sql
ALTER TABLE users DROP COLUMN name;
ALTER TABLE users RENAME COLUMN full_name TO name; -- or keep as full_name
```

### Index Creation (Non-Blocking)

```sql
-- 000008_add_users_email_index.up.sql
-- CONCURRENTLY prevents table lock — safe in production
-- Note: cannot run inside a transaction block
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_users_email ON users(email);
```

```sql
-- 000008_add_users_email_index.down.sql
DROP INDEX CONCURRENTLY IF EXISTS idx_users_email;
```

**Warning:** golang-migrate wraps each migration in a transaction by default. `CREATE INDEX CONCURRENTLY` cannot run inside a transaction. Use the `--no-lock` pragma or split index creation into a dedicated migration with `BEGIN/COMMIT` omitted by using `-- migrate:notransaction` (depends on tool).

For golang-migrate, the workaround is using `database/sql` directly for the index migration:

```go
// Alternative: run CONCURRENTLY index outside migrate library
func createIndexConcurrently(db *sql.DB) error {
    _, err := db.Exec(`CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_users_email ON users(email)`)
    return err
}
```

### Backfill Large Tables

```sql
-- 000009_backfill_users_role.up.sql
-- Batch update to avoid long-running lock
DO $$
DECLARE
    batch_size INT := 1000;
    updated INT;
BEGIN
    LOOP
        UPDATE users
        SET role = 'user'
        WHERE role IS NULL
        AND id IN (
            SELECT id FROM users WHERE role IS NULL LIMIT batch_size
        );
        GET DIAGNOSTICS updated = ROW_COUNT;
        EXIT WHEN updated = 0;
        PERFORM pg_sleep(0.01); -- brief pause between batches
    END LOOP;
END $$;
```

---

## Seeding Data

Seed data belongs in separate files from schema migrations to keep concerns separated.

**Option A: Dedicated seed migration (simple, always runs)**

```sql
-- 000099_seed_admin_user.up.sql
INSERT INTO users (id, name, email, password_hash, role)
VALUES (
    'a0000000-0000-0000-0000-000000000001',
    'Admin',
    'admin@example.com',
    '$2a$12$...bcrypt-hash...',
    'admin'
)
ON CONFLICT (email) DO NOTHING; -- idempotent
```

**Option B: Go seed script (more control)**

```go
// cmd/seed/main.go
package main

import (
    "context"
    "log/slog"
    "os"

    "github.com/jmoiron/sqlx"
    _ "github.com/lib/pq"
)

func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
    db, err := sqlx.Connect("postgres", os.Getenv("DATABASE_URL"))
    if err != nil {
        logger.Error("connect failed", "error", err)
        os.Exit(1)
    }
    defer db.Close()

    ctx := context.Background()
    if err := seedAdminUser(ctx, db, logger); err != nil {
        logger.Error("seed failed", "error", err)
        os.Exit(1)
    }
    logger.Info("seed complete")
}

func seedAdminUser(ctx context.Context, db *sqlx.DB, logger *slog.Logger) error {
    _, err := db.ExecContext(ctx, `
        INSERT INTO users (id, name, email, password_hash, role)
        VALUES ($1, $2, $3, $4, $5)
        ON CONFLICT (email) DO NOTHING
    `,
        "a0000000-0000-0000-0000-000000000001",
        "Admin",
        os.Getenv("ADMIN_EMAIL"),
        os.Getenv("ADMIN_PASSWORD_HASH"),
        "admin",
    )
    if err != nil {
        return fmt.Errorf("seed admin: %w", err)
    }
    logger.Info("admin user seeded")
    return nil
}
```

Run with: `go run ./cmd/seed`

**Critical:** Never hardcode credentials in seed files. Read from environment variables. Never commit seed scripts with real passwords.

---

## Rollback Strategies

```bash
# Roll back exactly 1 migration (most common)
migrate -path db/migrations -database "$DATABASE_URL" down 1

# Roll back N migrations
migrate -path db/migrations -database "$DATABASE_URL" down 3

# Check what version you are on
migrate -path db/migrations -database "$DATABASE_URL" version

# Dirty state recovery: fix schema manually, then clear dirty flag
migrate -path db/migrations -database "$DATABASE_URL" force <version>
```

**When down migrations are impractical:**

Some operations (DROP TABLE, DROP COLUMN) destroy data. Write compensating up migrations instead of down migrations.

```sql
-- 000010_restore_users_bio.up.sql (compensating migration)
ALTER TABLE users ADD COLUMN bio TEXT; -- restore what 000009_drop_bio dropped
```

**Rollback policy recommendations:**

| Environment | Strategy |
|-------------|----------|
| Local dev | `down 1` freely — fast iteration |
| Staging | `down` only to recover from failed deploy |
| Production | Never roll back schema — deploy a fix migration |

---

## Example Migrations — Users and Roles

### Migration 000001: Create Users Table

```sql
-- db/migrations/000001_create_users_table.up.sql
CREATE EXTENSION IF NOT EXISTS "pgcrypto"; -- provides gen_random_uuid()

CREATE TABLE users (
    id            UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    name          VARCHAR(100) NOT NULL,
    email         VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role          VARCHAR(50)  NOT NULL DEFAULT 'user',
    created_at    TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at    TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    deleted_at    TIMESTAMPTZ  -- NULL = active, non-NULL = soft deleted
);

CREATE UNIQUE INDEX idx_users_email ON users(email) WHERE deleted_at IS NULL;
CREATE INDEX idx_users_role ON users(role);
CREATE INDEX idx_users_deleted_at ON users(deleted_at) WHERE deleted_at IS NOT NULL;
```

```sql
-- db/migrations/000001_create_users_table.down.sql
DROP TABLE IF EXISTS users;
DROP EXTENSION IF EXISTS "pgcrypto";
```

### Migration 000002: Create Roles Table

```sql
-- db/migrations/000002_create_roles_table.up.sql
CREATE TABLE roles (
    id          UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    name        VARCHAR(50)  NOT NULL UNIQUE,
    description TEXT,
    created_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

-- Seed built-in roles
INSERT INTO roles (name, description) VALUES
    ('admin',  'Full system access'),
    ('user',   'Standard user access'),
    ('viewer', 'Read-only access');
```

```sql
-- db/migrations/000002_create_roles_table.down.sql
DROP TABLE IF EXISTS roles;
```

### Migration 000003: User-Roles Junction Table

```sql
-- db/migrations/000003_add_user_roles_junction.up.sql
CREATE TABLE user_roles (
    user_id    UUID        NOT NULL REFERENCES users(id)  ON DELETE CASCADE,
    role_id    UUID        NOT NULL REFERENCES roles(id)  ON DELETE CASCADE,
    granted_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    granted_by UUID        REFERENCES users(id),

    PRIMARY KEY (user_id, role_id)
);

CREATE INDEX idx_user_roles_user_id ON user_roles(user_id);
CREATE INDEX idx_user_roles_role_id ON user_roles(role_id);
```

```sql
-- db/migrations/000003_add_user_roles_junction.down.sql
DROP TABLE IF EXISTS user_roles;
```

### Migration 000004: Add updated_at Trigger

```sql
-- db/migrations/000004_add_updated_at_trigger.up.sql
-- Automatically keeps updated_at in sync without application code
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER users_set_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW
    EXECUTE FUNCTION set_updated_at();
```

```sql
-- db/migrations/000004_add_updated_at_trigger.down.sql
DROP TRIGGER IF EXISTS users_set_updated_at ON users;
DROP FUNCTION IF EXISTS set_updated_at();
```
