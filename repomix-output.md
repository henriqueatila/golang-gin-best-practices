This file is a merged representation of a subset of the codebase, containing files not matching ignore patterns, combined into a single document by Repomix.

<file_summary>
This section contains a summary of this file.

<purpose>
This file contains a packed representation of a subset of the repository's contents that is considered the most important context.
It is designed to be easily consumable by AI systems for analysis, code review,
or other automated processes.
</purpose>

<file_format>
The content is organized as follows:
1. This summary section
2. Repository information
3. Directory structure
4. Repository files (if enabled)
5. Multiple file entries, each consisting of:
  - File path as an attribute
  - Full contents of the file
</file_format>

<usage_guidelines>
- This file should be treated as read-only. Any changes should be made to the
  original repository files, not this packed version.
- When processing this file, use the file path to distinguish
  between different files in the repository.
- Be aware that this file may contain sensitive information. Handle it with
  the same level of security as you would the original repository.
</usage_guidelines>

<notes>
- Some files may have been excluded based on .gitignore rules and Repomix's configuration
- Binary files are not included in this packed representation. Please refer to the Repository Structure section for a complete list of file paths, including binary files
- Files matching these patterns are excluded: plans/**, *.zip, repomix-output.md
- Files matching patterns in .gitignore are excluded
- Files matching default ignore patterns are excluded
- Files are sorted by Git change count (files with more changes are at the bottom)
</notes>

</file_summary>

<directory_structure>
skills/
  golang-gin-api/
    references/
      error-handling.md
      middleware.md
      rate-limiting.md
      routing.md
      websocket.md
    metadata.json
    README.md
    SKILL.md
  golang-gin-auth/
    references/
      jwt-patterns.md
      rbac.md
    metadata.json
    README.md
    SKILL.md
  golang-gin-database/
    references/
      gorm-patterns.md
      migrations.md
      sqlx-patterns.md
    metadata.json
    README.md
    SKILL.md
  golang-gin-deploy/
    references/
      docker-compose.md
      dockerfile.md
      kubernetes.md
      observability.md
    metadata.json
    README.md
    SKILL.md
  golang-gin-swagger/
    references/
      annotations.md
      ci-cd.md
    metadata.json
    README.md
    SKILL.md
  golang-gin-testing/
    references/
      e2e.md
      integration-tests.md
      unit-tests.md
    metadata.json
    README.md
    SKILL.md
  golang-gin-api.zip
  golang-gin-auth.zip
  golang-gin-database.zip
  golang-gin-deploy.zip
  golang-gin-swagger.zip
  golang-gin-testing.zip
.gitignore
AGENTS.md
CLAUDE.md
LICENSE
README.md
</directory_structure>

<files>
This section contains the contents of the repository's files.

<file path="skills/golang-gin-api/README.md">
# golang-gin-api

Core REST API patterns for the Go Gin framework.

## What's Included

- **SKILL.md** — Server setup, routing, handler patterns, request binding/validation, error handling, and layered project structure
- **references/routing.md** — Route groups, versioning, pagination, file uploads
- **references/middleware.md** — CORS, rate limiting, request ID, timeout, recovery
- **references/error-handling.md** — AppError system, validation errors, panic recovery

## Categories

`go` `gin` `rest-api` `web-framework` `routing` `middleware` `error-handling`

## Install

```bash
npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-api
```
</file>

<file path="skills/golang-gin-auth/README.md">
# golang-gin-auth

JWT authentication and authorization for Go Gin APIs.

## What's Included

- **SKILL.md** — JWT middleware, login/register handlers, RBAC, token lifecycle, protected routes
- **references/jwt-patterns.md** — Token refresh, blacklisting (Redis), RS256 vs HS256
- **references/rbac.md** — RequireRole, permissions, multi-tenant authorization

## Categories

`go` `gin` `jwt` `authentication` `authorization` `rbac` `middleware`

## Install

```bash
npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-auth
```
</file>

<file path="skills/golang-gin-database/references/migrations.md">
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
    "fmt"
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
</file>

<file path="skills/golang-gin-database/README.md">
# golang-gin-database

PostgreSQL integration for Go Gin APIs using GORM and sqlx.

## What's Included

- **SKILL.md** — Repository pattern, GORM/sqlx setup, connection pooling, transactions, DI
- **references/gorm-patterns.md** — Models, CRUD, soft deletes, transactions, hooks
- **references/sqlx-patterns.md** — Struct scanning, NamedExec, IN clauses, transactions
- **references/migrations.md** — golang-migrate CLI, zero-downtime, seeding, rollback

## Categories

`go` `gin` `postgresql` `gorm` `sqlx` `database` `repository-pattern` `migrations`

## Install

```bash
npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-database
```
</file>

<file path="skills/golang-gin-deploy/README.md">
# golang-gin-deploy

Containerization and deployment for Go Gin APIs.

## What's Included

- **SKILL.md** — Multi-stage Dockerfile, docker-compose, health checks, graceful shutdown
- **references/dockerfile.md** — Distroless, build args, layer caching, image size
- **references/docker-compose.md** — Air hot reload, pgadmin, networking, integration tests
- **references/kubernetes.md** — Deployment, Service, ConfigMap, HPA, Ingress, Helm

## Categories

`go` `gin` `docker` `kubernetes` `ci-cd` `deployment` `docker-compose`

## Install

```bash
npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-deploy
```
</file>

<file path="skills/golang-gin-swagger/references/ci-cd.md">
# ci-cd.md — CI/CD, Tooling & Advanced Configuration

GitHub Actions workflows, PR validation, pre-commit hooks, OpenAPI 3.0 conversion, multiple swagger instances, and the complete `swag init` flags reference.

## Table of Contents

1. [GitHub Actions — Generate & Commit](#1-github-actions--generate--commit)
2. [GitHub Actions — PR Validation](#2-github-actions--pr-validation)
3. [Makefile Integration](#3-makefile-integration)
4. [Pre-commit Hook](#4-pre-commit-hook)
5. [OpenAPI 3.0 Conversion](#5-openapi-30-conversion)
6. [Multiple Swagger Instances](#6-multiple-swagger-instances)
7. [swag init Flags Reference](#7-swag-init-flags-reference)
8. [Docker Integration](#8-docker-integration)
9. [Troubleshooting CI Failures](#9-troubleshooting-ci-failures)

---

## 1. GitHub Actions — Generate & Commit

Auto-generate and commit docs on push to `main`:

```yaml
name: swagger-docs

on:
  push:
    branches: [main]
    paths: ['**/*.go']

jobs:
  generate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}

      - uses: actions/setup-go@v5
        with:
          go-version: '1.24'
          cache: true

      - name: Install swag
        run: go install github.com/swaggo/swag/cmd/swag@latest

      - name: Generate docs
        run: swag init -g cmd/api/main.go -d ./,./internal/... --exclude ./vendor

      - name: Commit if changed
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add docs/
          git diff --staged --quiet || git commit -m "docs: regenerate swagger"
          git push
```

---

## 2. GitHub Actions — PR Validation

Fail the PR if swagger docs are stale:

```yaml
name: swagger-check

on:
  pull_request:
    paths: ['**/*.go']

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version: '1.24'
          cache: true

      - name: Install swag
        run: go install github.com/swaggo/swag/cmd/swag@latest

      - name: Check docs are up to date
        run: |
          swag init -g cmd/api/main.go -d ./,./internal/... --exclude ./vendor
          git diff --exit-code docs/ || {
            echo "::error::Swagger docs are stale. Run 'make docs' and commit the changes."
            exit 1
          }
```

---

## 3. Makefile Integration

```makefile
.PHONY: docs docs-check docs-serve

# Generate swagger docs
docs:
	swag fmt
	swag init -g cmd/api/main.go -d ./,./internal/... --exclude ./vendor

# Validate docs are committed (used in CI)
docs-check:
	swag init -g cmd/api/main.go -d ./,./internal/... --exclude ./vendor
	git diff --exit-code docs/

# Serve swagger UI locally (requires the app to be running)
docs-serve:
	@echo "Swagger UI: http://localhost:8080/swagger/index.html"
```

---

## 4. Pre-commit Hook

Auto-regenerate docs before each commit:

```bash
#!/bin/sh
# .git/hooks/pre-commit

# Check if any Go files changed
STAGED_GO=$(git diff --cached --name-only --diff-filter=ACM | grep '\.go$')
if [ -z "$STAGED_GO" ]; then
    exit 0
fi

# Regenerate docs
swag fmt
swag init -g cmd/api/main.go -d ./,./internal/... --exclude ./vendor

# Stage updated docs
git add docs/
```

Make executable: `chmod +x .git/hooks/pre-commit`

---

## 5. OpenAPI 3.0 Conversion

swag v1.x generates Swagger 2.0. If OpenAPI 3.0 is required (e.g., for external API gateways), convert as a post-processing step:

```bash
# Install converter
npm install -g swagger2openapi

# Convert after swag init
swag init -g cmd/api/main.go
swagger2openapi docs/swagger.json -o docs/openapi3.json
swagger2openapi docs/swagger.yaml -o docs/openapi3.yaml
```

Add to Makefile:

```makefile
docs-openapi3: docs
	swagger2openapi docs/swagger.json -o docs/openapi3.json
	swagger2openapi docs/swagger.yaml -o docs/openapi3.yaml
```

**Note:** swag v2 (OpenAPI 3.1 native) is still RC and not production-ready. Use the conversion approach until v2 reaches stable.

---

## 6. Multiple Swagger Instances

Serve separate docs for API v1 and v2 on the same server:

```bash
# Generate separate docs
swag init --instanceName v1 -g cmd/api/main.go -o ./docs/v1
swag init --instanceName v2 -g cmd/api/v2/main.go -o ./docs/v2
```

```go
import (
    _ "myapp/docs/v1"
    _ "myapp/docs/v2"
    swaggerFiles "github.com/swaggo/files"
    ginSwagger   "github.com/swaggo/gin-swagger"
)

// Serve each version at a different path
r.GET("/swagger/v1/*any", ginSwagger.WrapHandler(
    swaggerFiles.NewHandler(),
    ginSwagger.InstanceName("v1"),
))
r.GET("/swagger/v2/*any", ginSwagger.WrapHandler(
    swaggerFiles.NewHandler(),
    ginSwagger.InstanceName("v2"),
))
```

---

## 7. swag init Flags Reference

```
swag init [flags]
```

| Flag | Short | Default | Description |
|------|-------|---------|-------------|
| `--generalInfo` | `-g` | `main.go` | Go file with general API annotations |
| `--dir` | `-d` | `./` | Directories to parse (comma-separated) |
| `--exclude` | | | Paths to exclude (comma-separated) |
| `--output` | `-o` | `./docs` | Output directory |
| `--outputTypes` | `--ot` | `go,json,yaml` | File types to generate |
| `--parseDependency` | `--pd` | `false` | Parse types in dependency modules |
| `--parseDependencyLevel` | `--pdl` | `0` | Depth: 0=off, 1=models, 2=ops, 3=all |
| `--parseInternal` | | `false` | Parse `internal/` packages |
| `--parseVendor` | | `false` | Parse `vendor/` directory |
| `--parseGoList` | | `true` | Resolve deps via `go list` |
| `--propertyStrategy` | `-p` | `CamelCase` | Field naming: SnakeCase, CamelCase, PascalCase |
| `--instanceName` | | | Unique name for multiple swagger instances |
| `--tags` | `-t` | | Filter by tag (prefix `!` to exclude) |
| `--markdownFiles` | `--md` | | Folder with markdown for `@Description` |
| `--packageName` | | | Override `docs.go` package name |
| `--requiredByDefault` | | `false` | Mark all fields required unless optional |
| `--quiet` | `-q` | `false` | Suppress logging |
| `--generatedTime` | | `false` | Add timestamp to `docs.go` |
| `--overridesFile` | | `.swaggo` | File for global type overrides |
| `--collectionFormat` | `--cf` | `csv` | Default array collection format |
| `--useStructName` | | `false` | Use struct name only (fixes `internal_` prefix) |

### Common Combos

```bash
# Standard cmd/ layout
swag init -g cmd/api/main.go -d ./,./internal/handler,./internal/domain

# Fast — Go file only (skip JSON/YAML)
swag init -g cmd/api/main.go --outputTypes go

# Parse internal + external types
swag init -g cmd/api/main.go --parseInternal --parseDependency --parseDependencyLevel 1

# Exclude test and vendor dirs
swag init -g cmd/api/main.go --exclude ./vendor,./test

# Filter to specific tag group
swag init -g cmd/api/main.go -t users
swag init -g cmd/api/main.go -t "!internal"  # exclude internal tag
```

---

## 8. Docker Integration

Add `swag init` to the build stage of a multi-stage Dockerfile:

```dockerfile
FROM golang:1.24-alpine AS builder

RUN go install github.com/swaggo/swag/cmd/swag@latest

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN swag init -g cmd/api/main.go -d ./,./internal/...
RUN CGO_ENABLED=0 go build -o /app/server ./cmd/api

FROM gcr.io/distroless/static-debian12
COPY --from=builder /app/server /server
COPY --from=builder /app/docs /docs
ENTRYPOINT ["/server"]
```

For production builds without Swagger UI (using build tags):

```dockerfile
# Dev/staging — includes Swagger UI
RUN CGO_ENABLED=0 go build -tags swagger -o /app/server ./cmd/api

# Production — excludes Swagger UI
RUN CGO_ENABLED=0 go build -o /app/server ./cmd/api
```

---

## 9. Troubleshooting CI Failures

| Symptom | Cause | Fix |
|---------|-------|-----|
| `swag: command not found` | `GOPATH/bin` not in PATH | Add `export PATH=$(go env GOPATH)/bin:$PATH` |
| `cannot find type definition` | Type in `internal/` | Add `--parseInternal` flag |
| `cannot find type definition` | Type in external dep | Add `--parseDependency --parseDependencyLevel 1` |
| `git diff --exit-code docs/` fails | Dev forgot to run `swag init` | Run `make docs` and commit |
| `docs/docs.go` has different timestamp | `--generatedTime` enabled | Remove `--generatedTime` flag |
| Slow CI step (>30s) | Parsing too many directories | Narrow `-d` to only handler/domain dirs |
| `internal_domain_User` in docs | Known bug with `--parseInternal` | Add `--useStructName` flag |
</file>

<file path="skills/golang-gin-swagger/README.md">
# golang-gin-swagger

Swagger/OpenAPI documentation for Go Gin APIs using swaggo/swag.

## What's Included

- **SKILL.md** — Setup, handler annotations, model tags, Swagger UI, doc generation, common gotchas
- **references/annotations.md** — Complete annotation reference: all @Param types, file uploads, response headers, enums, model renaming, security schemes
- **references/ci-cd.md** — GitHub Actions workflows, PR validation, pre-commit hooks, OpenAPI 3.0 conversion, Docker integration, swag init flags

## Categories

`go` `gin` `swagger` `openapi` `api-documentation` `swaggo` `rest-api`

## Install

```bash
npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-swagger
```
</file>

<file path="skills/golang-gin-testing/README.md">
# golang-gin-testing

Testing patterns for Go Gin REST APIs.

## What's Included

- **SKILL.md** — httptest patterns, table-driven tests, mock repositories, test setup
- **references/unit-tests.md** — Handler tests, middleware isolation, mock generation
- **references/integration-tests.md** — testcontainers, TestMain, DB lifecycle, cleanup
- **references/e2e.md** — Full flows, docker-compose tests, GitHub Actions CI

## Categories

`go` `gin` `testing` `httptest` `testcontainers` `unit-tests` `integration-tests`

## Install

```bash
npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-testing
```
</file>

<file path="LICENSE">
MIT License

Copyright (c) 2026 gin-skills contributors

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
</file>

<file path="skills/golang-gin-database/references/sqlx-patterns.md">
# sqlx Patterns Reference

This file covers sqlx-specific patterns for PostgreSQL integration in Gin APIs: connection setup, struct scanning, Get/Select/NamedExec, safe dynamic queries, transactions, sqlx.In for IN clauses, null handling, query builder patterns, connection pooling, and a complete repository implementation satisfying the same `domain.UserRepository` interface as the GORM version. Use when building the repository layer with sqlx instead of GORM.

> **Architectural recommendation:** These are mainstream Go/sqlx community patterns, not part of the Gin framework API.

## Table of Contents

1. [Connection Setup](#connection-setup)
2. [Struct Scanning with db Tags](#struct-scanning-with-db-tags)
3. [Get, Select, NamedExec](#get-select-namedexec)
4. [Safe Dynamic Queries](#safe-dynamic-queries)
5. [Transactions with sqlx.Tx](#transactions-with-sqlxtx)
6. [sqlx.In for IN Clauses](#sqlxin-for-in-clauses)
7. [Null Handling](#null-handling)
8. [Query Builder Patterns](#query-builder-patterns)
9. [Connection Pooling](#connection-pooling)
10. [Complete Repository Implementation](#complete-repository-implementation)

---

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
// Architectural recommendation — not part of the Gin API.
func NewSqlxDB(cfg SqlxConfig, logger *slog.Logger) (*sqlx.DB, error) {
    db, err := sqlx.Connect("postgres", cfg.DSN)
    if err != nil {
        return nil, fmt.Errorf("sqlx.Connect: %w", err)
    }

    db.SetMaxOpenConns(cfg.MaxOpenConns)       // e.g. 25
    db.SetMaxIdleConns(cfg.MaxIdleConns)       // e.g. 5
    db.SetConnMaxLifetime(cfg.ConnMaxLifetime)  // e.g. 5*time.Minute

    // sqlx.Connect already pings — this confirms pool is healthy
    if err := db.Ping(); err != nil {
        return nil, fmt.Errorf("db.Ping: %w", err)
    }

    logger.Info("database connected (sqlx)")
    return db, nil
}
```

**sqlx vs GORM:** sqlx wraps `database/sql` with struct scanning and named queries. It executes raw SQL — no ORM magic, no hidden queries, full control. Choose sqlx when you need complex queries or existing DBA-crafted SQL.

---

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
// Kept in the repository package — never exposed to domain or handlers.
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
        ID:        r.ID,
        Name:      r.Name,
        Email:     r.Email,
        Role:      r.Role,
        CreatedAt: r.CreatedAt,
        UpdatedAt: r.UpdatedAt,
    }
}

// userInsertRow includes fields only used on INSERT (e.g. password_hash).
type userInsertRow struct {
    ID           string `db:"id"`
    Name         string `db:"name"`
    Email        string `db:"email"`
    PasswordHash string `db:"password_hash"`
    Role         string `db:"role"`
}
```

**Rule:** Column names in SQL must exactly match `db` tags. Mismatches produce silent zero values — always verify with a test query.

---

## Get, Select, NamedExec

sqlx adds three key methods on top of `database/sql`:

| Method | Returns | Use for |
|--------|---------|---------|
| `GetContext` | single row into struct | `SELECT ... WHERE id = $1` |
| `SelectContext` | slice of structs | `SELECT ... WHERE ...` many rows |
| `NamedExecContext` | `sql.Result` | INSERT/UPDATE with named params |

```go
// GetContext — single row; returns sql.ErrNoRows if not found
var u userRow
err := db.GetContext(ctx, &u, `
    SELECT id, name, email, role, created_at, updated_at
    FROM users
    WHERE id = $1 AND deleted_at IS NULL
`, id)

// SelectContext — multiple rows
var rows []userRow
err := db.SelectContext(ctx, &rows, `
    SELECT id, name, email, role, created_at, updated_at
    FROM users
    WHERE deleted_at IS NULL
    ORDER BY created_at DESC
    LIMIT $1 OFFSET $2
`, limit, offset)

// NamedExecContext — named parameters from struct or map
row := userInsertRow{
    ID:           newUUID(),
    Name:         req.Name,
    Email:        req.Email,
    PasswordHash: hashPassword(req.Password),
    Role:         req.Role,
}
_, err := db.NamedExecContext(ctx, `
    INSERT INTO users (id, name, email, password_hash, role)
    VALUES (:id, :name, :email, :password_hash, :role)
`, row)
```

**Critical:** Never use `fmt.Sprintf` to build SQL strings — use `$N` placeholders for positional params or `:name` for named params. This prevents SQL injection.

---

## Safe Dynamic Queries

Build dynamic WHERE clauses safely using `squirrel` or manual slice accumulation. Never string-concatenate user input.

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

    where := strings.Join(conditions, " AND ")
    limit := opts.Limit
    if limit <= 0 || limit > 100 {
        limit = 20
    }
    offset := (opts.Page - 1) * limit

    query := fmt.Sprintf(`
        SELECT id, name, email, role, created_at, updated_at
        FROM users
        WHERE %s
        ORDER BY created_at DESC
        LIMIT $%d OFFSET $%d
    `, where, argIdx, argIdx+1)

    args = append(args, limit, offset)
    return query, args
}

// Option B: squirrel query builder (github.com/Masterminds/squirrel)
import sq "github.com/Masterminds/squirrel"

psql := sq.StatementBuilder.PlaceholderFormat(sq.Dollar)

qb := psql.Select("id", "name", "email", "role", "created_at", "updated_at").
    From("users").
    Where("deleted_at IS NULL")

if opts.Role != "" {
    qb = qb.Where(sq.Eq{"role": opts.Role})
}

sql, args, err := qb.OrderBy("created_at DESC").
    Limit(uint64(limit)).
    Offset(uint64(offset)).
    ToSql()
```

---

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

// Service-layer usage — the service orchestrates the transaction:
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

---

## sqlx.In for IN Clauses

`database/sql` does not expand `IN (?)` for slices — sqlx.In does.

```go
import "github.com/jmoiron/sqlx"

// sqlx.In expands the slice into positional placeholders
func (r *sqlxUserRepository) GetByIDs(ctx context.Context, ids []string) ([]domain.User, error) {
    if len(ids) == 0 {
        return nil, nil
    }

    // sqlx.In rewrites "IN (?)" → "IN ($1,$2,$3,...)"
    query, args, err := sqlx.In(`
        SELECT id, name, email, role, created_at, updated_at
        FROM users
        WHERE id IN (?) AND deleted_at IS NULL
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

---

## Null Handling

PostgreSQL NULLs map to Go's zero values unless you use `database/sql` null types.

```go
import "database/sql"

// userRowNullable handles columns that can be NULL in the database
type userRowNullable struct {
    ID        string         `db:"id"`
    Name      string         `db:"name"`
    Email     string         `db:"email"`
    Role      string         `db:"role"`
    DeletedAt sql.NullTime   `db:"deleted_at"` // NULL when not deleted
    Bio       sql.NullString `db:"bio"`         // NULL when profile incomplete
    CreatedAt time.Time      `db:"created_at"`
    UpdatedAt time.Time      `db:"updated_at"`
}

func (r *userRowNullable) toDomain() *domain.User {
    u := &domain.User{
        ID:        r.ID,
        Name:      r.Name,
        Email:     r.Email,
        Role:      r.Role,
        CreatedAt: r.CreatedAt,
        UpdatedAt: r.UpdatedAt,
    }
    if r.Bio.Valid {
        u.Bio = r.Bio.String
    }
    return u
}

// Alternative: use github.com/guregu/null for ergonomic null types
import "gopkg.in/guregu/null.v4"

type userRowNullV2 struct {
    Bio null.String `db:"bio"` // null.String.ValueOrZero() returns "" if null
}
```

---

## Query Builder Patterns

Keep SQL in constants or a dedicated file for discoverability.

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
        FROM users
        WHERE id = $1 AND deleted_at IS NULL
    `

    selectUserByEmailSQL = `
        SELECT id, name, email, role, created_at, updated_at
        FROM users
        WHERE email = $1 AND deleted_at IS NULL
    `

    updateUserSQL = `
        UPDATE users
        SET name = :name, role = :role, updated_at = NOW()
        WHERE id = :id AND deleted_at IS NULL
    `

    softDeleteUserSQL = `
        UPDATE users SET deleted_at = NOW() WHERE id = $1 AND deleted_at IS NULL
    `

    countUsersByRoleSQL = `
        SELECT COUNT(*) FROM users WHERE deleted_at IS NULL
    `
)
```

**Why constants?** Grep-able, type-checkable at compile time, and easy to test in isolation with `sqlx.NamedExec` dry-run.

---

## Connection Pooling

Same recommendations as GORM (both use `database/sql` under the hood):

```go
db.SetMaxOpenConns(25)                     // max concurrent connections
db.SetMaxIdleConns(5)                      // warm idle pool
db.SetConnMaxLifetime(5 * time.Minute)     // recycle stale connections
db.SetConnMaxIdleTime(1 * time.Minute)     // close idle connections sooner
```

**Health check endpoint** — ping the database to confirm connectivity:

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

---

## Complete Repository Implementation

Full `UserRepository` satisfying the same `domain.UserRepository` interface as the GORM version — swap implementations in `main.go` without changing service code.

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
// Drop-in replacement for NewUserRepository (GORM version).
func NewSqlxUserRepository(db *sqlx.DB) domain.UserRepository {
    return &sqlxUserRepository{db: db}
}

func (r *sqlxUserRepository) Create(ctx context.Context, user *domain.User) error {
    if user.ID == "" {
        user.ID = newUUID()
    }
    if user.Role == "" {
        user.Role = "user"
    }

    // PasswordHash must be set on user before calling Create.
    // The service layer hashes the password and stores it in user.PasswordHash.
    row := userInsertRow{
        ID:           user.ID,
        Name:         user.Name,
        Email:        user.Email,
        PasswordHash: user.PasswordHash, // set by service before calling repo
        Role:         user.Role,
    }
    _, err := r.db.NamedExecContext(ctx, insertUserSQL, row)
    if err != nil {
        return mapSqlxError(err)
    }
    return nil
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
    // Count
    var total int64
    countQuery, countArgs := buildCountQuery(opts)
    if err := r.db.GetContext(ctx, &total, countQuery, countArgs...); err != nil {
        return nil, 0, domain.ErrInternal.New(err)
    }

    // Fetch page
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
        "id":   user.ID,
        "name": user.Name,
        "role": user.Role,
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

// mapSqlxError converts database/sql errors to domain errors.
func mapSqlxError(err error) error {
    if errors.Is(err, sql.ErrNoRows) {
        return domain.ErrNotFound.New(err)
    }
    // PostgreSQL unique violation (SQLSTATE 23505) — typed assertion via lib/pq
    var pqErr *pq.Error
    if errors.As(err, &pqErr) && pqErr.Code == "23505" {
        return domain.ErrConflict.New(err)
    }
    return domain.ErrInternal.New(err)
}

// newUUID generates a new UUID string.
// Use github.com/google/uuid in production.
func newUUID() string {
    return uuid.Must(uuid.NewV7()).String() // UUIDv7: time-sortable, better B-tree index performance
}

// buildCountQuery builds a parameterized COUNT query for the given options.
func buildCountQuery(opts domain.ListOptions) (string, []any) {
    conds := []string{"deleted_at IS NULL"}
    args := []any{}
    idx := 1
    if opts.Role != "" {
        conds = append(conds, fmt.Sprintf("role = $%d", idx))
        args = append(args, opts.Role)
    }
    return fmt.Sprintf("SELECT COUNT(*) FROM users WHERE %s", strings.Join(conds, " AND ")), args
}

// buildListQuery builds a parameterized SELECT with pagination.
func buildListQuery(opts domain.ListOptions) (string, []any) {
    conds := []string{"deleted_at IS NULL"}
    args := []any{}
    idx := 1
    if opts.Role != "" {
        conds = append(conds, fmt.Sprintf("role = $%d", idx))
        args = append(args, opts.Role)
        idx++
    }
    limit := opts.Limit
    if limit <= 0 || limit > 100 {
        limit = 20
    }
    offset := (opts.Page - 1) * limit
    args = append(args, limit, offset)
    query := fmt.Sprintf(`
        SELECT id, name, email, role, created_at, updated_at
        FROM users
        WHERE %s
        ORDER BY created_at DESC
        LIMIT $%d OFFSET $%d
    `, strings.Join(conds, " AND "), idx, idx+1)
    return query, args
}
```
</file>

<file path=".gitignore">
# See https://help.github.com/articles/ignoring-files/ for more about ignoring files.

# dependencies
/node_modules
/.pnp
.pnp.*
.yarn/*
!.yarn/patches
!.yarn/plugins
!.yarn/releases
!.yarn/versions

# testing
/coverage

# next.js
/.next/
/out/
dist/
bin/

# production
/build

# misc
.DS_Store
*.pem

# debug
npm-debug.log*
yarn-debug.log*
yarn-error.log*
.pnpm-debug.log*

# package manager
package-lock.json
yarn.lock
pnpm-lock.yaml

# semantic-release
.nyc_output

# env files (can opt-in for committing if needed)
.env*
!.env.example

# vercel
.vercel

# typescript
*.tsbuildinfo
next-env.d.ts

# flutter
.dart_tool
build
GoogleService-Info.plist

repomix-output.xml
.serena/cache
plans/**/*
!plans/templates/*
screenshots/*
docs/screenshots/*
logs.txt
test-ck
__pycache__

# CK meta commands
prompt.md
ck.md

# Agent memory (project scope)
.claude/agent-memory/

# Hook diagnostics logs
.claude/hooks/.logs/

# Gemini CLI settings (symlink to .claude/.mcp.json)
.gemini/settings.json
.claude/settings.bak.json
# External repos for study/reference
external/

# Git worktrees (local development only)
worktrees/

# AI agent / IDE config directories
.agent/
.agents/
.codebuddy/
.commandcode/
.continue/
.crush/
.factory/
.goose/
.junie/
.kilocode/
.kiro/
.kode/
.mcpjam/
.mux/
.neovate/
.opencode/
.openhands/
.pi/
.pochi/
.qoder/
.qwen/
.roo/
.trae/
.windsurf/
.zencoder/
# AGENTS.md — tracked for this repo (agent guidance)

# Local skill lock files and release manifest
skills-lock.json
release-manifest.json
home/

# Repomix
.repomixignore
</file>

<file path="skills/golang-gin-api/references/routing.md">
# Routing Reference

This file covers advanced routing patterns for Gin: route groups and nesting, API versioning, path and query parameter binding, wildcard routes, NoRoute handler, custom validators, request size limits, and multipart file upload handling.

## Table of Contents

1. [Route Groups & Nesting](#route-groups--nesting)
2. [API Versioning](#api-versioning)
3. [Path Parameter Patterns](#path-parameter-patterns)
4. [Query Parameter Binding with Pagination](#query-parameter-binding-with-pagination)
5. [Wildcard Routes & NoRoute Handler](#wildcard-routes--noroute-handler)
6. [Custom Validators](#custom-validators)
7. [Request Size Limits](#request-size-limits)
8. [Multipart File Upload](#multipart-file-upload)

---

## Route Groups & Nesting

Groups share a path prefix and optional middleware. Use `{}` blocks for readability — they are cosmetic only (Go scope, not Gin scope).

```go
func registerRoutes(r *gin.Engine, h *handler.Handlers) {
    // Health outside versioning — no auth, no version prefix
    r.GET("/health", h.Health.Check)

    api := r.Group("/api")
    {
        v1 := api.Group("/v1")
        {
            // Public: no middleware beyond global
            auth := v1.Group("/auth")
            {
                auth.POST("/login", h.Auth.Login)
                auth.POST("/register", h.Auth.Register)
                auth.POST("/refresh", h.Auth.Refresh)
            }

            // Protected: JWT required
            users := v1.Group("/users")
            users.Use(middleware.Auth(tokenCfg, logger))
            {
                users.GET("", h.User.List)
                users.POST("", h.User.Create)
                users.GET("/:id", h.User.GetByID)
                users.PUT("/:id", h.User.Update)
                users.DELETE("/:id", h.User.Delete)
            }

            // Admin: JWT + role check
            admin := v1.Group("/admin")
            admin.Use(middleware.Auth(tokenCfg, logger), middleware.RequireRole("admin"))
            {
                admin.GET("/users", h.Admin.ListAllUsers)
                admin.DELETE("/users/:id", h.Admin.DeleteUser)
            }
        }
    }
}
```

**Why group nesting:** Each level applies its middleware to all routes below. This avoids repeating `middleware.Auth(tokenCfg, logger)` on every route and makes the security model explicit in the structure.

---

## API Versioning

### URL Path Versioning (recommended)

Path versioning is explicit, cache-friendly, and easy to route at the load-balancer level.

```go
v1 := r.Group("/api/v1")
v2 := r.Group("/api/v2")

// v1 handler
v1.GET("/users", listUsersV1)

// v2 handler — different response shape
v2.GET("/users", listUsersV2)
```

### Header-Based Versioning

Use when you want a stable URL but need to evolve the API contract.

```go
// pkg/middleware/version.go
func APIVersion() gin.HandlerFunc {
    return func(c *gin.Context) {
        version := c.GetHeader("API-Version")
        if version == "" {
            version = "v1" // default
        }
        c.Set("api_version", version)
        c.Next()
    }
}

// Handler reads version from context
func listUsers(c *gin.Context) {
    version := c.GetString("api_version")
    switch version {
    case "v2":
        // return v2 shape
    default:
        // return v1 shape
    }
}
```

```go
// Register with single path
r.Use(middleware.APIVersion())
r.GET("/api/users", listUsers)
```

---

## Path Parameter Patterns

> **Security note:** In production, never expose raw `err.Error()` to clients. Return generic messages and log the error server-side. See **golang-gin-clean-arch** error handling patterns.

Use `ShouldBindURI` with a struct to bind multiple path parameters at once.

```go
// Single parameter
r.GET("/users/:id", func(c *gin.Context) {
    id := c.Param("id") // returns "/uuid-here" — note leading slash for wildcard, not for :param
    // For :id, c.Param returns the value without slash
})

// Multiple parameters with struct binding
r.GET("/orgs/:orgID/users/:userID", func(c *gin.Context) {
    type params struct {
        OrgID  string `uri:"orgID"  binding:"required"`
        UserID string `uri:"userID" binding:"required"`
    }
    var p params
    if err := c.ShouldBindURI(&p); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    // p.OrgID, p.UserID are bound and validated
})
```

**Critical:** URI parameter names in the `uri:""` tag must match the `:name` in the route path exactly.

---

## Query Parameter Binding with Pagination

Bind all query parameters at once using `ShouldBindQuery`. Define defaults via struct field initialisation before binding.

```go
// internal/domain/pagination.go
package domain

// ListOptions is the standard pagination + filter struct.
type ListOptions struct {
    Page   int    `form:"page"   binding:"min=1"`
    Limit  int    `form:"limit"  binding:"min=1,max=100"`
    Sort   string `form:"sort"   binding:"omitempty,oneof=created_at updated_at name"`
    Order  string `form:"order"  binding:"omitempty,oneof=asc desc"`
    Search string `form:"search" binding:"omitempty,max=100"`
    Role   string `form:"role"   binding:"omitempty,oneof=admin user"`
}

func (o *ListOptions) SetDefaults() {
    if o.Page == 0 {
        o.Page = 1
    }
    if o.Limit == 0 {
        o.Limit = 20
    }
    if o.Sort == "" {
        o.Sort = "created_at"
    }
    if o.Order == "" {
        o.Order = "desc"
    }
}
```

```go
// Handler
func (h *UserHandler) List(c *gin.Context) {
    var opts domain.ListOptions
    opts.SetDefaults()

    if err := c.ShouldBindQuery(&opts); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    users, total, err := h.svc.List(c.Request.Context(), opts)
    if err != nil {
        handleServiceError(c, err, h.logger)
        return
    }

    c.JSON(http.StatusOK, gin.H{
        "data":  users,
        "total": total,
        "page":  opts.Page,
        "limit": opts.Limit,
    })
}
```

---

## Wildcard Routes & NoRoute Handler

### Wildcard Parameters

`*param` matches everything after the prefix, including slashes.

```go
// Matches: /files/docs/report.pdf, /files/images/logo.png
r.GET("/files/*filepath", func(c *gin.Context) {
    filepath := c.Param("filepath") // includes leading slash: "/docs/report.pdf"
    // Serve file from storage...
})
```

### NoRoute Handler (Custom 404)

Replace Gin's default 404 with a JSON response.

```go
r.NoRoute(func(c *gin.Context) {
    c.JSON(http.StatusNotFound, gin.H{
        "error": "route not found",
        "path":  c.Request.URL.Path,
    })
})

// Required — NoMethod handler only fires when this is true (default: false)
r.HandleMethodNotAllowed = true

r.NoMethod(func(c *gin.Context) {
    c.JSON(http.StatusMethodNotAllowed, gin.H{
        "error":  "method not allowed",
        "method": c.Request.Method,
    })
})
```

---

## Custom Validators

Register custom validators via the `go-playground/validator/v10` engine. Do this once at startup.

```go
// pkg/validation/validators.go
package validation

import (
    "fmt"
    "time"

    "github.com/gin-gonic/gin/binding"
    "github.com/go-playground/validator/v10"
    "github.com/google/uuid"
)

func Register() error {
    v, ok := binding.Validator.Engine().(*validator.Validate)
    if !ok {
        return fmt.Errorf("unexpected validator type")
    }

    // Custom: validate that string is a valid UUID
    if err := v.RegisterValidation("uuid", validateUUID); err != nil {
        return fmt.Errorf("register uuid validator: %w", err)
    }

    // Custom: validate that int is a valid Unix timestamp (not in the past)
    if err := v.RegisterValidation("future_ts", validateFutureTimestamp); err != nil {
        return fmt.Errorf("register future_ts validator: %w", err)
    }

    return nil
}

func validateUUID(fl validator.FieldLevel) bool {
    val := fl.Field().String()
    _, err := uuid.Parse(val)
    return err == nil
}

func validateFutureTimestamp(fl validator.FieldLevel) bool {
    ts := fl.Field().Int()
    return time.Unix(ts, 0).After(time.Now())
}
```

Register in main.go before starting the server:

```go
if err := validation.Register(); err != nil {
    logger.Error("failed to register validators", "error", err)
    os.Exit(1)
}
```

Usage in request struct:

```go
type ScheduleRequest struct {
    UserID    string `json:"user_id"    binding:"required,uuid"`
    StartTime int64  `json:"start_time" binding:"required,future_ts"`
}
```

---

## Request Size Limits

Limit request body size to protect against large payload attacks.

```go
// pkg/middleware/limits.go
package middleware

import (
    "net/http"
    "github.com/gin-gonic/gin"
)

// RequestSizeLimit returns middleware that limits request body size.
// maxBytes: maximum allowed body size in bytes.
func RequestSizeLimit(maxBytes int64) gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Request.Body = http.MaxBytesReader(c.Writer, c.Request.Body, maxBytes)
        c.Next()
    }
}
```

```go
// Apply globally (1 MB limit for all routes)
r.Use(middleware.RequestSizeLimit(1 << 20)) // 1 MB

// Or per-route (10 MB for file upload route)
r.POST("/upload", middleware.RequestSizeLimit(10<<20), uploadHandler)
```

For multipart forms, also configure `engine.MaxMultipartMemory`:

```go
r := gin.New()
r.MaxMultipartMemory = 8 << 20 // 8 MB in-memory; rest spills to disk
```

---

## Multipart File Upload

```go
// internal/handler/upload_handler.go
package handler

import (
    "fmt"
    "net/http"
    "path/filepath"

    "github.com/gin-gonic/gin"
)

func (h *UploadHandler) UploadAvatar(c *gin.Context) {
    // ShouldBindURI for any path params first
    type uriParams struct {
        UserID string `uri:"id" binding:"required"`
    }
    var p uriParams
    if err := c.ShouldBindURI(&p); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    // Get single file
    file, err := c.FormFile("avatar")
    if err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "avatar file required"})
        return
    }

    // Validate file type by extension
    ext := filepath.Ext(file.Filename)
    allowed := map[string]bool{".jpg": true, ".jpeg": true, ".png": true, ".webp": true}
    if !allowed[ext] {
        c.JSON(http.StatusBadRequest, gin.H{"error": "only jpg, png, webp files allowed"})
        return
    }

    // Validate file size (2 MB max)
    if file.Size > 2<<20 {
        c.JSON(http.StatusBadRequest, gin.H{"error": "file must be under 2 MB"})
        return
    }

    // Sanitize path: strip directory components to prevent path traversal
    safeID := filepath.Base(p.UserID)
    dst := filepath.Join("uploads/avatars", safeID+ext)
    if err := c.SaveUploadedFile(file, dst); err != nil {
        handleServiceError(c, domain.ErrInternal.New(fmt.Errorf("save file: %w", err)), h.logger)
        return
    }

    c.JSON(http.StatusOK, gin.H{"path": dst})
}

// Multiple file upload
func (h *UploadHandler) UploadDocuments(c *gin.Context) {
    form, err := c.MultipartForm()
    if err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "multipart form required"})
        return
    }

    files := form.File["documents"]
    if len(files) == 0 {
        c.JSON(http.StatusBadRequest, gin.H{"error": "at least one document required"})
        return
    }

    saved := make([]string, 0, len(files))
    for _, file := range files {
        // Sanitize filename: strip directory components to prevent path traversal
        safeName := filepath.Base(file.Filename)
        if safeName == "." || safeName == "/" {
            c.JSON(http.StatusBadRequest, gin.H{"error": "invalid filename"})
            return
        }
        dst := filepath.Join("uploads/docs", safeName)
        if err := c.SaveUploadedFile(file, dst); err != nil {
            c.JSON(http.StatusInternalServerError, gin.H{"error": "failed to save file"})
            return
        }
        saved = append(saved, dst)
    }

    c.JSON(http.StatusOK, gin.H{"saved": saved, "count": len(saved)})
}
```

Route registration:

```go
r.MaxMultipartMemory = 8 << 20

protected := r.Group("/api/v1")
protected.Use(middleware.Auth(tokenCfg, logger))
{
    protected.POST("/users/:id/avatar",    uploadHandler.UploadAvatar)
    protected.POST("/users/:id/documents", uploadHandler.UploadDocuments)
}
```
</file>

<file path="skills/golang-gin-api/references/websocket.md">
# WebSocket Reference

This file covers WebSocket integration with Gin using `gorilla/websocket`. Gin handles the HTTP upgrade handshake through its normal routing and middleware chain; after upgrade, all communication is raw WebSocket — Gin is no longer involved. Topics: upgrader setup, echo handler, hub pattern for broadcasting, authentication before upgrade, ping/pong keepalive, graceful shutdown, JSON messages, and testing.

## Table of Contents

1. [Upgrader Setup](#upgrader-setup)
2. [Basic Echo Handler](#basic-echo-handler)
3. [Hub Pattern](#hub-pattern)
4. [Client Struct with readPump / writePump](#client-struct-with-readpump--writepump)
5. [Auth Before Upgrade](#auth-before-upgrade)
6. [Ping/Pong Keepalive](#pingpong-keepalive)
7. [Graceful Shutdown](#graceful-shutdown)
8. [JSON Messages](#json-messages)
9. [Testing](#testing)

---

## Upgrader Setup

`websocket.Upgrader` converts an HTTP connection to a WebSocket connection. Configure it once at the package level (or inject it) — it is safe to use concurrently.

```go
// internal/ws/upgrader.go
package ws

import (
    "net/http"

    "github.com/gorilla/websocket"
)

// upgrader is the shared upgrader for all WebSocket endpoints.
// ReadBufferSize / WriteBufferSize tune the internal I/O buffers — not message
// size limits. 1024 bytes is a safe default for typical JSON payloads.
var upgrader = websocket.Upgrader{
    ReadBufferSize:  1024,
    WriteBufferSize: 1024,

    // CheckOrigin gates which origins may open a WebSocket connection.
    // Never return true unconditionally in production — that allows
    // cross-site WebSocket hijacking (CSWSH).
    CheckOrigin: func(r *http.Request) bool {
        origin := r.Header.Get("Origin")
        // Allow requests with no origin header (e.g., native clients, tests).
        if origin == "" {
            return true
        }
        allowed := map[string]bool{
            "https://example.com":     true,
            "https://app.example.com": true,
        }
        return allowed[origin]
    },
}
```

**Why `CheckOrigin` matters:** Browsers automatically send the `Origin` header on WebSocket requests. Without validation, any website can open a WebSocket to your server using the visitor's credentials (cookies/sessions).

**`SetReadLimit`** — set a per-connection message size limit to prevent memory exhaustion from malicious clients:

```go
// After upgrading, before the read loop:
conn.SetReadLimit(512 * 1024) // 512 KB max per message
```

---

## Basic Echo Handler

The simplest handler: upgrade the connection, read messages in a loop, write each one back. This pattern demonstrates the full lifecycle.

```go
// internal/ws/echo.go
package ws

import (
    "log/slog"

    "github.com/gin-gonic/gin"
    "github.com/gorilla/websocket"
)

// EchoHandler upgrades the HTTP connection and echoes every message back.
func EchoHandler(c *gin.Context) {
    conn, err := upgrader.Upgrade(c.Writer, c.Request, nil)
    if err != nil {
        // Upgrade already wrote a 400 response on failure; just log.
        slog.Error("websocket upgrade failed", "err", err)
        return
    }
    defer conn.Close()

    conn.SetReadLimit(512 * 1024)

    for {
        msgType, msg, err := conn.ReadMessage()
        if err != nil {
            if websocket.IsUnexpectedCloseError(err,
                websocket.CloseGoingAway,
                websocket.CloseNormalClosure,
            ) {
                slog.Warn("websocket read error", "err", err)
            }
            return // client disconnected or error — exit loop
        }

        if err := conn.WriteMessage(msgType, msg); err != nil {
            slog.Warn("websocket write error", "err", err)
            return
        }
    }
}
```

Register the handler like any Gin route — middleware runs during the HTTP upgrade request:

```go
r := gin.New()
r.Use(middleware.Logger(), middleware.Recovery())

r.GET("/ws/echo", ws.EchoHandler)
```

**Why `IsUnexpectedCloseError`:** When a client disconnects cleanly (browser tab closed), gorilla/websocket returns `CloseGoingAway`. Logging that as an error is noise. `IsUnexpectedCloseError` filters out expected close codes so you only log genuine errors.

---

## Hub Pattern

For broadcasting to multiple clients (chat rooms, live feeds), a central hub serializes all register/unregister/broadcast operations through a single goroutine — avoiding mutex-protected maps.

```go
// internal/ws/hub.go
package ws

// Hub maintains active client connections and routes messages.
type Hub struct {
    clients    map[*Client]bool
    broadcast  chan []byte
    register   chan *Client
    unregister chan *Client
}

func NewHub() *Hub {
    return &Hub{
        clients:    make(map[*Client]bool),
        broadcast:  make(chan []byte, 256),
        register:   make(chan *Client),
        unregister: make(chan *Client),
    }
}

// Run processes hub events. Call this in a dedicated goroutine.
func (h *Hub) Run() {
    for {
        select {
        case client := <-h.register:
            h.clients[client] = true

        case client := <-h.unregister:
            if _, ok := h.clients[client]; ok {
                delete(h.clients, client)
                close(client.send)
            }

        case message := <-h.broadcast:
            for client := range h.clients {
                select {
                case client.send <- message:
                default:
                    // send buffer full — client is too slow; drop and disconnect.
                    close(client.send)
                    delete(h.clients, client)
                }
            }
        }
    }
}
```

**Why channel-based (not mutex):** The hub goroutine owns the `clients` map exclusively. No locking needed. The `default` branch in broadcast prevents a slow client from blocking the entire broadcast loop.

Wire the hub into your Gin server:

```go
// cmd/server/main.go
hub := ws.NewHub()
go hub.Run()

r.GET("/ws", ws.ChatHandler(hub))
```

---

## Client Struct with readPump / writePump

Each connected client gets two goroutines: `readPump` reads from the WebSocket, `writePump` writes to it. Separating read and write avoids concurrent writes to `*websocket.Conn` (which gorilla/websocket does not allow).

```go
// internal/ws/client.go
package ws

import (
    "log/slog"
    "time"

    "github.com/gin-gonic/gin"
    "github.com/gorilla/websocket"
)

const (
    writeWait      = 10 * time.Second
    pongWait       = 60 * time.Second
    pingPeriod     = (pongWait * 9) / 10 // must be less than pongWait
    maxMessageSize = 512 * 1024           // 512 KB
)

// Client represents a single WebSocket connection.
type Client struct {
    hub  *Hub
    conn *websocket.Conn
    send chan []byte // buffered channel of outbound messages
}

// readPump reads messages from the WebSocket and forwards them to the hub.
// One goroutine per connection.
func (cl *Client) readPump() {
    defer func() {
        cl.hub.unregister <- cl
        cl.conn.Close()
    }()

    cl.conn.SetReadLimit(maxMessageSize)
    cl.conn.SetReadDeadline(time.Now().Add(pongWait))
    cl.conn.SetPongHandler(func(string) error {
        cl.conn.SetReadDeadline(time.Now().Add(pongWait))
        return nil
    })

    for {
        _, msg, err := cl.conn.ReadMessage()
        if err != nil {
            if websocket.IsUnexpectedCloseError(err,
                websocket.CloseGoingAway,
                websocket.CloseNormalClosure,
            ) {
                slog.Warn("ws read error", "err", err)
            }
            return
        }
        cl.hub.broadcast <- msg
    }
}

// writePump writes messages from the send channel to the WebSocket.
// One goroutine per connection — gorilla/websocket requires that writes
// happen from a single goroutine.
func (cl *Client) writePump() {
    ticker := time.NewTicker(pingPeriod)
    defer func() {
        ticker.Stop()
        cl.conn.Close()
    }()

    for {
        select {
        case msg, ok := <-cl.send:
            cl.conn.SetWriteDeadline(time.Now().Add(writeWait))
            if !ok {
                // Hub closed the channel — send a close frame.
                cl.conn.WriteMessage(websocket.CloseMessage, []byte{})
                return
            }
            if err := cl.conn.WriteMessage(websocket.TextMessage, msg); err != nil {
                slog.Warn("ws write error", "err", err)
                return
            }

        case <-ticker.C:
            cl.conn.SetWriteDeadline(time.Now().Add(writeWait))
            if err := cl.conn.WriteMessage(websocket.PingMessage, nil); err != nil {
                return
            }
        }
    }
}

// ChatHandler upgrades the connection and registers the client with the hub.
func ChatHandler(hub *Hub) gin.HandlerFunc {
    return func(c *gin.Context) {
        conn, err := upgrader.Upgrade(c.Writer, c.Request, nil)
        if err != nil {
            slog.Error("ws upgrade failed", "err", err)
            return
        }

        // c.Copy() is required when passing *gin.Context to goroutines.
        // The original context is recycled by Gin after the handler returns.
        // However, after upgrade we don't use the Gin context anymore —
        // only the raw *websocket.Conn. Still, capture request-scoped values
        // (e.g., user ID set by auth middleware) before the handler returns.
        cpy := c.Copy()
        _ = cpy // use cpy.GetString("userID") etc. if needed

        client := &Client{
            hub:  hub,
            conn: conn,
            send: make(chan []byte, 256),
        }
        hub.register <- client

        go client.writePump()
        go client.readPump()
        // Handler returns immediately. readPump/writePump own the connection.
    }
}
```

**Why `c.Copy()` before goroutines:** Gin recycles `*gin.Context` objects via `sync.Pool` after the handler returns. If a goroutine holds the original context, it may read corrupted data from a different request. `c.Copy()` creates a snapshot safe to use after the handler returns.

---

## Auth Before Upgrade

WebSocket has no standard mechanism for sending auth headers after the handshake. Authenticate during the HTTP upgrade request — before calling `upgrader.Upgrade()`. If auth fails, return a normal HTTP error response.

```go
// internal/ws/auth_handler.go
package ws

import (
    "log/slog"
    "net/http"

    "github.com/gin-gonic/gin"
    "github.com/gorilla/websocket"
)

// AuthChatHandler validates a JWT from the query param before upgrading.
// Browsers cannot set custom headers on WebSocket connections — query params
// are the standard workaround. Keep tokens short-lived (< 60 s) when using
// them in URLs to reduce exposure in server logs.
func AuthChatHandler(hub *Hub, validateToken func(string) (string, error)) gin.HandlerFunc {
    return func(c *gin.Context) {
        token := c.Query("token")
        if token == "" {
            c.JSON(http.StatusUnauthorized, gin.H{"error": "missing token"})
            return
        }

        userID, err := validateToken(token)
        if err != nil {
            slog.Warn("ws auth failed", "err", err)
            c.JSON(http.StatusUnauthorized, gin.H{"error": "invalid token"})
            return
        }

        conn, err := upgrader.Upgrade(c.Writer, c.Request, nil)
        if err != nil {
            slog.Error("ws upgrade failed", "err", err)
            return
        }

        // Auth middleware may also set userID via c.Set — capture it here
        // before the handler returns and Gin recycles the context.
        _ = userID // attach to Client struct as needed

        client := &Client{
            hub:  hub,
            conn: conn,
            send: make(chan []byte, 256),
        }
        hub.register <- client

        go client.writePump()
        go client.readPump()
    }
}
```

**Route registration — auth middleware still runs:**

```go
wsGroup := r.Group("/ws")
wsGroup.Use(middleware.RateLimit()) // rate limiting applies to upgrade request
{
    // Token in query param — no JWT middleware needed on this route
    wsGroup.GET("/chat", ws.AuthChatHandler(hub, tokenSvc.ValidateToken))
}
```

**Alternative — `Sec-WebSocket-Protocol` header:** Some clients send the token as a subprotocol name. This avoids URL exposure but requires echoing the subprotocol in the upgrade response header, which is more complex and less portable. Query param is simpler for most use cases.

---

## Ping/Pong Keepalive

Without keepalive, idle connections time out silently at the load balancer or OS level. gorilla/websocket supports the WebSocket ping/pong control frames natively.

The `writePump` ticker above already sends pings. The `readPump` deadline is reset on every pong. Here is the pattern isolated for clarity:

```go
// In readPump — reset read deadline when a pong arrives
conn.SetReadDeadline(time.Now().Add(pongWait)) // initial deadline
conn.SetPongHandler(func(appData string) error {
    // Extend deadline: client is alive
    conn.SetReadDeadline(time.Now().Add(pongWait))
    return nil
})

// In writePump ticker case — send a ping
conn.SetWriteDeadline(time.Now().Add(writeWait))
if err := conn.WriteMessage(websocket.PingMessage, nil); err != nil {
    return // connection broken; writePump exits, triggering cleanup
}
```

**Deadline chain:** writePump sends ping every `pingPeriod` (54 s). Client must reply with pong before `pongWait` (60 s) expires. If pong never arrives, `ReadMessage` returns a deadline error and `readPump` exits, which triggers `hub.unregister` and `conn.Close()`.

**Why `pingPeriod < pongWait`:** The ping must be sent before the read deadline fires. Using `(pongWait * 9) / 10` gives a 10% safety margin.

---

## Graceful Shutdown

On server shutdown, close all WebSocket connections cleanly so clients can reconnect rather than hang.

```go
// internal/ws/hub.go — add shutdown support
type Hub struct {
    clients    map[*Client]bool
    broadcast  chan []byte
    register   chan *Client
    unregister chan *Client
    shutdown   chan struct{}
}

func NewHub() *Hub {
    return &Hub{
        clients:    make(map[*Client]bool),
        broadcast:  make(chan []byte, 256),
        register:   make(chan *Client),
        unregister: make(chan *Client),
        shutdown:   make(chan struct{}),
    }
}

func (h *Hub) Run() {
    for {
        select {
        case <-h.shutdown:
            for client := range h.clients {
                // Send a close frame so the client knows the server is shutting down.
                client.conn.WriteMessage(
                    websocket.CloseMessage,
                    websocket.FormatCloseMessage(websocket.CloseServiceRestart, "server shutdown"),
                )
                close(client.send)
                delete(h.clients, client)
            }
            return

        case client := <-h.register:
            h.clients[client] = true

        case client := <-h.unregister:
            if _, ok := h.clients[client]; ok {
                delete(h.clients, client)
                close(client.send)
            }

        case message := <-h.broadcast:
            for client := range h.clients {
                select {
                case client.send <- message:
                default:
                    close(client.send)
                    delete(h.clients, client)
                }
            }
        }
    }
}

// Shutdown signals the hub to close all connections and stop.
func (h *Hub) Shutdown() {
    close(h.shutdown)
}
```

Wire into server shutdown:

```go
// cmd/server/main.go
hub := ws.NewHub()
go hub.Run()

srv := &http.Server{Addr: ":8080", Handler: r}

quit := make(chan os.Signal, 1)
signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
<-quit

hub.Shutdown() // close WebSocket connections first

ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()
if err := srv.Shutdown(ctx); err != nil {
    slog.Error("server shutdown error", "err", err)
}
```

**Why close WebSocket before HTTP shutdown:** `srv.Shutdown` stops accepting new requests and waits for in-flight HTTP handlers to finish. WebSocket handlers return immediately after spawning goroutines — those goroutines run outside Gin's request lifecycle. Shutting the hub down first ensures client goroutines exit before the process terminates.

---

## JSON Messages

Use typed message envelopes with `ReadJSON`/`WriteJSON` to make the wire protocol explicit.

```go
// internal/ws/message.go
package ws

import (
    "log/slog"
    "time"

    "github.com/gorilla/websocket"
)

// MessageType identifies the kind of payload.
type MessageType string

const (
    MsgChat   MessageType = "chat"
    MsgSystem MessageType = "system"
    MsgError  MessageType = "error"
)

// Envelope is the wire format for all WebSocket messages.
type Envelope struct {
    Type    MessageType `json:"type"`
    Payload any `json:"payload"`
    SentAt  time.Time   `json:"sent_at"`
}

// ChatPayload is the payload for MsgChat messages.
type ChatPayload struct {
    UserID  string `json:"user_id"`
    Content string `json:"content"`
}

// readJSON reads one message from the connection into dst.
func readJSON(conn *websocket.Conn, dst any) error {
    conn.SetReadDeadline(time.Now().Add(pongWait))
    return conn.ReadJSON(dst)
}

// writeJSON writes src to the connection as JSON.
func writeJSON(conn *websocket.Conn, src any) error {
    conn.SetWriteDeadline(time.Now().Add(writeWait))
    return conn.WriteJSON(src)
}

// Example: handler that processes typed messages
func handleTypedMessage(conn *websocket.Conn) {
    for {
        var env Envelope
        if err := readJSON(conn, &env); err != nil {
            if websocket.IsUnexpectedCloseError(err,
                websocket.CloseGoingAway,
                websocket.CloseNormalClosure,
            ) {
                slog.Warn("ws read error", "err", err)
            }
            return
        }

        switch env.Type {
        case MsgChat:
            // Re-encode payload to typed struct for validation
            // (ReadJSON into any gives map[string]any)
            slog.Info("chat message received", "payload", env.Payload)
        default:
            resp := Envelope{
                Type:    MsgError,
                Payload: "unknown message type",
                SentAt:  time.Now(),
            }
            if err := writeJSON(conn, resp); err != nil {
                slog.Warn("ws write error", "err", err)
                return
            }
        }
    }
}
```

**Why `any` in `Envelope.Payload`:** The envelope type is fixed, but the payload varies per message type. Unmarshal into `Envelope` first, then use `json.Unmarshal` on the re-encoded payload for strict typing per type:

```go
import "encoding/json"

raw, err := json.Marshal(env.Payload)
if err != nil {
    return
}
var chat ChatPayload
if err := json.Unmarshal(raw, &chat); err != nil {
    slog.Warn("invalid chat payload", "err", err)
    return
}
```

---

## Testing

WebSocket tests require a real TCP server — `httptest.NewServer` + `websocket.DefaultDialer.Dial`. You cannot use `httptest.NewRecorder` for WebSocket because the upgrade requires a hijackable connection.

```go
// internal/ws/echo_test.go
package ws_test

import (
    "net/http"
    "net/http/httptest"
    "strings"
    "testing"
    "time"

    "github.com/gin-gonic/gin"
    "github.com/gorilla/websocket"

    ws "github.com/yourorg/yourapp/internal/ws"
)

func newTestServer(t *testing.T) *httptest.Server {
    t.Helper()
    gin.SetMode(gin.TestMode)
    r := gin.New()
    r.GET("/ws/echo", ws.EchoHandler)
    return httptest.NewServer(r)
}

func wsURL(srv *httptest.Server, path string) string {
    return "ws" + strings.TrimPrefix(srv.URL, "http") + path
}

func TestEchoHandler_RoundTrip(t *testing.T) {
    srv := newTestServer(t)
    defer srv.Close()

    dialer := websocket.Dialer{HandshakeTimeout: 3 * time.Second}
    conn, resp, err := dialer.Dial(wsURL(srv, "/ws/echo"), nil)
    if err != nil {
        t.Fatalf("dial failed: %v (status %d)", err, resp.StatusCode)
    }
    defer conn.Close()

    want := "hello"
    if err := conn.WriteMessage(websocket.TextMessage, []byte(want)); err != nil {
        t.Fatalf("write failed: %v", err)
    }

    conn.SetReadDeadline(time.Now().Add(3 * time.Second))
    _, got, err := conn.ReadMessage()
    if err != nil {
        t.Fatalf("read failed: %v", err)
    }

    if string(got) != want {
        t.Errorf("got %q, want %q", got, want)
    }
}

func TestEchoHandler_ClientClose(t *testing.T) {
    srv := newTestServer(t)
    defer srv.Close()

    conn, _, err := websocket.DefaultDialer.Dial(wsURL(srv, "/ws/echo"), nil)
    if err != nil {
        t.Fatalf("dial failed: %v", err)
    }

    // Send a normal close frame; server should exit its read loop cleanly.
    err = conn.WriteMessage(
        websocket.CloseMessage,
        websocket.FormatCloseMessage(websocket.CloseNormalClosure, "done"),
    )
    if err != nil {
        t.Fatalf("close write failed: %v", err)
    }

    conn.SetReadDeadline(time.Now().Add(2 * time.Second))
    _, _, err = conn.ReadMessage()
    if err == nil {
        t.Error("expected error after close, got nil")
    }
    // A CloseError or io.EOF is expected — both indicate server closed the connection.
    if !websocket.IsCloseError(err, websocket.CloseNormalClosure) {
        if err.Error() != "EOF" && !strings.Contains(err.Error(), "use of closed") {
            t.Logf("close read returned: %v (acceptable)", err)
        }
    }
}

func TestChatHandler_Broadcast(t *testing.T) {
    gin.SetMode(gin.TestMode)
    hub := ws.NewHub()
    go hub.Run()
    defer hub.Shutdown()

    r := gin.New()
    r.GET("/ws/chat", ws.ChatHandler(hub))
    srv := httptest.NewServer(r)
    defer srv.Close()

    dial := func() *websocket.Conn {
        conn, _, err := websocket.DefaultDialer.Dial(wsURL(srv, "/ws/chat"), nil)
        if err != nil {
            t.Fatalf("dial failed: %v", err)
        }
        return conn
    }

    c1 := dial()
    c2 := dial()
    defer c1.Close()
    defer c2.Close()

    // Small sleep to let register events process
    time.Sleep(50 * time.Millisecond)

    want := "broadcast-test"
    if err := c1.WriteMessage(websocket.TextMessage, []byte(want)); err != nil {
        t.Fatalf("c1 write failed: %v", err)
    }

    // Both c1 and c2 should receive the broadcast
    for _, conn := range []*websocket.Conn{c1, c2} {
        conn.SetReadDeadline(time.Now().Add(2 * time.Second))
        _, msg, err := conn.ReadMessage()
        if err != nil {
            t.Fatalf("read failed: %v", err)
        }
        if string(msg) != want {
            t.Errorf("got %q, want %q", msg, want)
        }
    }
}
```

**Key testing patterns:**
- Use `httptest.NewServer` (not `httptest.NewRecorder`) — WebSocket requires a hijackable TCP connection.
- Convert `http://` to `ws://` with `strings.TrimPrefix`.
- Set `SetReadDeadline` in tests to prevent hangs on failure.
- Test both happy path and clean disconnect to verify server-side cleanup.
- `hub.Shutdown()` in defer ensures test goroutines exit cleanly.
</file>

<file path="skills/golang-gin-auth/references/rbac.md">
# rbac.md — Role-Based Access Control

Complete reference for RBAC and permission-based authorization in Gin APIs. Covers role middleware, permission checks, role hierarchy, multi-tenant auth, and resource-level authorization.

All patterns are architectural recommendations — they use the `Claims` struct from the gin-auth SKILL.md and Gin's `c.Set`/`c.Get` for context propagation.

## Table of Contents

1. [Role-Based Middleware](#role-based-middleware)
2. [Permission-Based Middleware](#permission-based-middleware)
3. [Role Hierarchy](#role-hierarchy)
4. [Multi-Tenant Authorization](#multi-tenant-authorization)
5. [Resource-Level Authorization](#resource-level-authorization)
6. [Admin Impersonation](#admin-impersonation)
7. [Complete RBAC Example](#complete-rbac-example)

---

## Role-Based Middleware

`RequireRole` and `RequireAnyRole` sit after `Auth` middleware in the chain. `Auth` validates the JWT and injects claims; RBAC middleware reads the role from those claims.

```go
// pkg/middleware/rbac.go
package middleware

import (
    "net/http"

    "github.com/gin-gonic/gin"
    "myapp/internal/auth"
)

// RequireRole aborts with 403 if the authenticated user's role does not match.
// Must be used after Auth middleware.
func RequireRole(role string) gin.HandlerFunc {
    return func(c *gin.Context) {
        claims := claimsFromContext(c)
        if claims == nil {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "unauthorized"})
            return
        }
        if claims.Role != role {
            c.AbortWithStatusJSON(http.StatusForbidden, gin.H{"error": "insufficient role"})
            return
        }
        c.Next()
    }
}

// RequireAnyRole aborts with 403 if the user's role is not in the allowed set.
func RequireAnyRole(roles ...string) gin.HandlerFunc {
    allowed := make(map[string]struct{}, len(roles))
    for _, r := range roles {
        allowed[r] = struct{}{}
    }
    return func(c *gin.Context) {
        claims := claimsFromContext(c)
        if claims == nil {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "unauthorized"})
            return
        }
        if _, ok := allowed[claims.Role]; !ok {
            c.AbortWithStatusJSON(http.StatusForbidden, gin.H{"error": "insufficient role"})
            return
        }
        c.Next()
    }
}

// claimsFromContext is a helper to safely extract *auth.Claims from gin.Context.
func claimsFromContext(c *gin.Context) *auth.Claims {
    val, exists := c.Get(ClaimsKey)
    if !exists {
        return nil
    }
    claims, _ := val.(*auth.Claims)
    return claims
}
```

**Usage in route registration:**

```go
protected := api.Group("")
protected.Use(middleware.Auth(tokenCfg, logger))
{
    // Any authenticated user
    protected.GET("/users/me", userHandler.GetMe)

    // Moderators and admins
    modGroup := protected.Group("")
    modGroup.Use(middleware.RequireAnyRole("admin", "moderator"))
    {
        modGroup.PUT("/posts/:id/hide", postHandler.Hide)
    }

    // Admins only
    adminGroup := protected.Group("/admin")
    adminGroup.Use(middleware.RequireRole("admin"))
    {
        adminGroup.GET("/users", userHandler.List)
        adminGroup.DELETE("/users/:id", userHandler.Delete)
    }
}
```

---

## Permission-Based Middleware

For fine-grained control beyond roles, embed permissions in the JWT claims or check them against a database.

**Option A — permissions in JWT claims** (fast, no DB lookup per request):

```go
// internal/auth/claims.go (extended)
package auth

import "github.com/golang-jwt/jwt/v5"

type Claims struct {
    jwt.RegisteredClaims
    UserID      string   `json:"uid"`
    Email       string   `json:"email"`
    Role        string   `json:"role"`
    Permissions []string `json:"perms"` // e.g. ["posts:write", "users:read"]
}
```

```go
// pkg/middleware/rbac.go (additional function)

// RequirePermission aborts with 403 if the user does not have the required permission.
// Permission format convention: "resource:action" (e.g. "users:delete").
func RequirePermission(permission string) gin.HandlerFunc {
    return func(c *gin.Context) {
        claims := claimsFromContext(c)
        if claims == nil {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "unauthorized"})
            return
        }
        for _, p := range claims.Permissions {
            if p == permission {
                c.Next()
                return
            }
        }
        c.AbortWithStatusJSON(http.StatusForbidden, gin.H{"error": "permission denied"})
    }
}
```

**Option B — DB lookup per request** (always up-to-date, higher latency):

```go
// pkg/middleware/rbac.go
func RequirePermissionDB(permSvc PermissionService, permission string) gin.HandlerFunc {
    return func(c *gin.Context) {
        claims := claimsFromContext(c)
        if claims == nil {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "unauthorized"})
            return
        }

        ok, err := permSvc.HasPermission(c.Request.Context(), claims.UserID, permission)
        if err != nil {
            c.AbortWithStatusJSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
            return
        }
        if !ok {
            c.AbortWithStatusJSON(http.StatusForbidden, gin.H{"error": "permission denied"})
            return
        }
        c.Next()
    }
}

// PermissionService is an interface — define at the consumer, implement in the service layer.
type PermissionService interface {
    HasPermission(ctx context.Context, userID, permission string) (bool, error)
}
```

**Trade-off:** JWT permissions are fast but stale (user retains permissions until token expires). DB lookup is always current but adds latency. Cache the DB result with a short TTL (e.g. 30s in Redis) as a middle ground.

---

## Role Hierarchy

Encode hierarchy as a map — higher-ranked roles inherit lower-ranked permissions.

```go
// internal/auth/roles.go
package auth

// roleRank maps role name to numeric rank. Higher = more privileged.
var roleRank = map[string]int{
    "guest":     0,
    "user":      1,
    "moderator": 2,
    "admin":     3,
    "superadmin": 4,
}

// HasRoleAtLeast returns true if userRole has rank >= requiredRole.
// Returns false for any unrecognized role — unknown roles are denied, not defaulted to guest.
func HasRoleAtLeast(userRole, requiredRole string) bool {
    ur, ok1 := roleRank[userRole]
    rr, ok2 := roleRank[requiredRole]
    return ok1 && ok2 && ur >= rr
}
```

```go
// pkg/middleware/rbac.go

// RequireMinRole allows any role with rank >= minRole (e.g. "moderator" also allows "admin").
func RequireMinRole(minRole string) gin.HandlerFunc {
    return func(c *gin.Context) {
        claims := claimsFromContext(c)
        if claims == nil {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "unauthorized"})
            return
        }
        if !auth.HasRoleAtLeast(claims.Role, minRole) {
            c.AbortWithStatusJSON(http.StatusForbidden, gin.H{
                "error": "insufficient privileges",
            })
            return
        }
        c.Next()
    }
}
```

---

## Multi-Tenant Authorization

In multi-tenant systems, users belong to a tenant. Requests must be validated both for authentication AND tenant membership.

**Add TenantID to claims:**

```go
// internal/auth/claims.go
type Claims struct {
    jwt.RegisteredClaims
    UserID   string `json:"uid"`
    Email    string `json:"email"`
    Role     string `json:"role"`
    TenantID string `json:"tid"` // tenant identifier
}
```

**Tenant middleware — injects TenantID from claims into context:**

```go
// pkg/middleware/tenant.go
package middleware

import (
    "net/http"
    "github.com/gin-gonic/gin"
)

const TenantIDKey = "tenant_id"

// RequireTenant ensures requests carry a valid tenant context.
// Must be used after Auth middleware.
func RequireTenant() gin.HandlerFunc {
    return func(c *gin.Context) {
        claims := claimsFromContext(c)
        if claims == nil || claims.TenantID == "" {
            c.AbortWithStatusJSON(http.StatusForbidden, gin.H{"error": "tenant context required"})
            return
        }
        c.Set(TenantIDKey, claims.TenantID)
        c.Next()
    }
}
```

**Enforce tenant isolation in repositories:**

```go
// internal/repository/post_repository_gorm.go

func (r *gormPostRepository) GetByID(ctx context.Context, tenantID, postID string) (*domain.Post, error) {
    var m PostModel
    // Always scope queries to the tenant — prevents cross-tenant data leaks
    err := r.db.WithContext(ctx).
        Where("tenant_id = ? AND id = ?", tenantID, postID).
        First(&m).Error
    if err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, domain.ErrNotFound
        }
        return nil, domain.ErrInternal
    }
    return m.ToDomain(), nil
}
```

**Handler reads tenant from context:**

```go
func (h *PostHandler) GetByID(c *gin.Context) {
    tenantID := c.GetString(middleware.TenantIDKey)

    type uriParams struct {
        ID string `uri:"id" binding:"required"`
    }
    var params uriParams
    if err := c.ShouldBindURI(&params); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "invalid request"})
        return
    }

    post, err := h.svc.GetByID(c.Request.Context(), tenantID, params.ID)
    if err != nil {
        handleServiceError(c, err, h.logger)
        return
    }

    c.JSON(http.StatusOK, post)
}
```

---

## Resource-Level Authorization

> **Security note:** In production, never expose raw `err.Error()` to clients. Return generic messages and log the error server-side. See **golang-gin-clean-arch** error handling patterns.

Verify the authenticated user owns or has access to the specific resource — not just the right role.

```go
// internal/handler/user_handler.go

// Update allows users to edit their own profile, or admins to edit any profile.
func (h *UserHandler) Update(c *gin.Context) {
    type uriParams struct {
        ID string `uri:"id" binding:"required"`
    }
    var params uriParams
    if err := c.ShouldBindURI(&params); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "invalid request"})
        return
    }

    claims := claimsFromContext(c)
    if claims == nil {
        c.JSON(http.StatusUnauthorized, gin.H{"error": "unauthorized"})
        return
    }

    // Resource-level check: only the owner or an admin may update
    if claims.UserID != params.ID && claims.Role != "admin" {
        c.JSON(http.StatusForbidden, gin.H{"error": "cannot modify another user's profile"})
        return
    }

    var req domain.UpdateUserRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "invalid request body"})
        return
    }

    user, err := h.svc.Update(c.Request.Context(), params.ID, req)
    if err != nil {
        handleServiceError(c, err, h.logger)
        return
    }

    c.JSON(http.StatusOK, user)
}
```

**Why in the handler, not middleware?** Resource ownership depends on business logic (the specific resource ID). Middleware runs before the route parameters are bound to a domain object — the handler is the right layer for this check.

---

## Admin Impersonation

Allows admins to act as another user for support or debugging. Inject an `ImpersonatedBy` field so audit logs capture both identities.

```go
// internal/auth/claims.go (extended)
type Claims struct {
    jwt.RegisteredClaims
    UserID         string `json:"uid"`
    Email          string `json:"email"`
    Role           string `json:"role"`
    ImpersonatedBy string `json:"imp,omitempty"` // set when admin impersonates
}
```

```go
// internal/handler/admin_handler.go

type impersonateRequest struct {
    TargetUserID string `json:"target_user_id" binding:"required"`
}

// Impersonate generates a short-lived access token for a target user.
// Only admins may call this endpoint.
func (h *AdminHandler) Impersonate(c *gin.Context) {
    adminClaims := claimsFromContext(c)
    if adminClaims == nil || adminClaims.Role != "admin" {
        c.JSON(http.StatusForbidden, gin.H{"error": "admin access required"})
        return
    }

    var req impersonateRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "invalid request body"})
        return
    }

    target, err := h.userRepo.GetByID(c.Request.Context(), req.TargetUserID)
    if err != nil {
        handleServiceError(c, err, h.logger)
        return
    }

    // Prevent admin-to-admin privilege escalation
    if target.Role == "admin" || target.Role == "superadmin" {
        c.JSON(http.StatusForbidden, gin.H{"error": "cannot impersonate admin users"})
        return
    }

    // Build impersonation token: short TTL, records impersonating admin
    claims := auth.Claims{
        RegisteredClaims: jwt.RegisteredClaims{
            Subject:   target.ID,
            IssuedAt:  jwt.NewNumericDate(time.Now()),
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(1 * time.Hour)), // short TTL
        },
        UserID:         target.ID,
        Email:          target.Email,
        Role:           target.Role,
        ImpersonatedBy: adminClaims.UserID,
    }
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    signed, err := token.SignedString(h.tokenCfg.AccessSecret)
    if err != nil {
        h.logger.Error("impersonate: failed to sign token", "error", err)
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
        return
    }

    h.logger.Info("admin impersonation", "admin_id", adminClaims.UserID, "target_id", target.ID)
    c.JSON(http.StatusOK, gin.H{"access_token": signed})
}
```

**Critical:** Log all impersonation events. Restrict the impersonation endpoint to `superadmin` role if you have a role hierarchy. Never allow impersonating another admin.

---

## Complete RBAC Example

Full wiring from routes to middleware to handler, using all the patterns above:

```go
// cmd/api/main.go — route registration with RBAC
func registerRoutes(
    r *gin.Engine,
    authHandler *handler.AuthHandler,
    userHandler *handler.UserHandler,
    postHandler *handler.PostHandler,
    adminHandler *handler.AdminHandler,
    tokenCfg auth.TokenConfig,
    logger *slog.Logger,
) {
    api := r.Group("/api/v1")

    // Public endpoints — no auth required
    api.POST("/auth/login", authHandler.Login)
    api.POST("/auth/refresh", authHandler.Refresh)
    api.POST("/users", userHandler.Create)

    // Authenticated endpoints
    authed := api.Group("")
    authed.Use(middleware.Auth(tokenCfg, logger))
    {
        authed.POST("/auth/logout", authHandler.Logout)

        // Self-service — any authenticated user
        authed.GET("/users/me", userHandler.GetMe)
        authed.PUT("/users/:id", userHandler.Update) // resource-level check inside handler

        // Posts — user and above (role hierarchy)
        posts := authed.Group("/posts")
        posts.Use(middleware.RequireMinRole("user"))
        {
            posts.GET("", postHandler.List)
            posts.GET("/:id", postHandler.GetByID)
            posts.POST("", postHandler.Create)
        }

        // Moderation — moderator and above
        moderation := authed.Group("/moderation")
        moderation.Use(middleware.RequireMinRole("moderator"))
        {
            moderation.PUT("/posts/:id/hide", postHandler.Hide)
            moderation.DELETE("/posts/:id", postHandler.Delete)
        }

        // Admin panel — admin role only
        admin := authed.Group("/admin")
        admin.Use(middleware.RequireRole("admin"))
        {
            admin.GET("/users", userHandler.List)
            admin.DELETE("/users/:id", userHandler.Delete)
            admin.POST("/impersonate", adminHandler.Impersonate)
        }
    }
}
```

**Middleware execution order for a protected admin route:**

```
Request → Auth (validate JWT, inject claims)
        → RequireRole("admin") (check claims.Role)
        → Handler (resource-level check if needed)
        → Response
```

**Key principles:**
- Auth middleware is always first in the protected group
- Role/permission middleware comes immediately after Auth
- Resource-level checks (owner == caller) belong in the handler, not middleware
- Use `RequireMinRole` for hierarchy, `RequireAnyRole` for explicit sets, `RequireRole` for exact match
- Always log authorization failures with enough context to audit (user ID, resource, action)
</file>

<file path="skills/golang-gin-deploy/references/dockerfile.md">
# Dockerfile Reference

This file covers Docker packaging for Go Gin APIs: multi-stage build rationale, base image comparison (distroless vs Alpine vs scratch), build arguments and secrets, layer caching, non-root user, HEALTHCHECK instruction, image size optimization, and a complete production Dockerfile. Use when you need to understand the tradeoffs or customize the build.

> **Architectural recommendation:** Docker patterns are not part of the Gin framework. These are mainstream Go community patterns.

## Table of Contents

1. [Multi-Stage Build Explained](#multi-stage-build-explained)
2. [Base Image Comparison](#base-image-comparison)
3. [Layer Caching Optimization](#layer-caching-optimization)
4. [Build Arguments and Secrets](#build-arguments-and-secrets)
5. [Non-Root User](#non-root-user)
6. [HEALTHCHECK Instruction](#healthcheck-instruction)
7. [Image Size Optimization](#image-size-optimization)
8. [.dockerignore](#dockerignore)
9. [Complete Production Dockerfile](#complete-production-dockerfile)

---

## Multi-Stage Build Explained

A multi-stage build uses multiple `FROM` instructions in one Dockerfile. Each stage is independent — only explicitly copied artifacts carry forward.

**Why it matters for Go:** The Go toolchain (~800 MB) is only needed at compile time. The final image only needs the compiled binary (~10 MB).

```
Stage 1 (builder)          Stage 2 (runtime)
─────────────────          ─────────────────
golang:1.24 (~800 MB)      distroless (~2 MB)
+ source code              + /server binary (~8 MB)
+ go modules               ────────────────────────
+ compiled binary          Final image: ~10 MB
```

The key directive:

```dockerfile
COPY --from=builder /app/server /server
```

Only the compiled binary crosses stage boundaries — source code, module cache, and the Go toolchain stay in the builder stage and are discarded.

---

## Base Image Comparison

| Image | Size | Shell | Package manager | Use case |
|-------|------|-------|----------------|----------|
| `gcr.io/distroless/static-debian12` | ~2 MB | No | No | Production (recommended) |
| `gcr.io/distroless/base-debian12` | ~20 MB | No | No | When CGO is required |
| `alpine:3.21` | ~7 MB | sh | apk | When shell access needed |
| `scratch` | 0 MB | No | No | Smallest possible; no TLS certs |

**Distroless (recommended):**

```dockerfile
FROM gcr.io/distroless/static-debian12:nonroot
```

- No shell = no shell injection attacks
- Includes CA certificates and timezone data (unlike `scratch`)
- `nonroot` tag runs as UID 65532 by default — no `USER` directive needed
- `CGO_ENABLED=0` required in builder stage

**When to use Alpine instead:**

```dockerfile
FROM alpine:3.21

# Alpine needs ca-certificates for HTTPS outbound calls if not using distroless
RUN apk add --no-cache ca-certificates tzdata
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

COPY --from=builder /app/server /app/server
ENTRYPOINT ["/app/server"]
```

Use Alpine when you need: shell for debugging, `apk` to install runtime deps (e.g., `git`, `openssh-client`), or when the team is unfamiliar with distroless.

**When to use scratch:**

```dockerfile
FROM scratch
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /app/server /server
ENTRYPOINT ["/server"]
```

Use scratch only when you need the absolute minimum image size and can manage TLS certificates manually. No timezone data — `time.LoadLocation` will fail without explicit tz data copy.

---

## Layer Caching Optimization

Docker caches each layer. A cache miss invalidates all subsequent layers. Ordering matters.

**Anti-pattern — cache broken by every code change:**

```dockerfile
# BAD: source copy before go mod download
COPY . .
RUN go mod download  # re-runs on every source change
RUN go build ...
```

**Correct pattern — dependencies cached separately:**

```dockerfile
# GOOD: module files copied first
COPY go.mod go.sum ./
RUN go mod download  # only re-runs when go.mod or go.sum changes

COPY . .             # source changes don't invalidate module cache
RUN go build ...
```

**Further optimization with BuildKit cache mount:**

```dockerfile
# syntax=docker/dockerfile:1

RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 go build -o /app/server ./cmd/api
```

The `--mount=type=cache` keeps the module and build caches across builds on the same host, making rebuilds ~5-10x faster when only source files change.

---

## Build Arguments and Secrets

**Build arguments (non-sensitive, embedded in image):**

```dockerfile
ARG VERSION=dev
ARG BUILD_TIME

RUN CGO_ENABLED=0 go build \
    -ldflags="-s -w -X main.Version=${VERSION} -X main.BuildTime=${BUILD_TIME}" \
    -o /app/server \
    ./cmd/api
```

Pass at build time:

```bash
docker build \
  --build-arg VERSION=$(git describe --tags --always) \
  --build-arg BUILD_TIME=$(date -u +%Y-%m-%dT%H:%M:%SZ) \
  -t myapp:latest .
```

Expose in the health endpoint:

```go
// Set via -ldflags at build time
var (
    Version   = "dev"
    BuildTime = "unknown"
)

func (h *HealthHandler) Check(c *gin.Context) {
    c.JSON(http.StatusOK, gin.H{
        "status":     "ok",
        "version":    Version,
        "build_time": BuildTime,
    })
}
```

**Secrets (never embedded — use BuildKit secret mount):**

```dockerfile
# syntax=docker/dockerfile:1

# Mount a secret during build without writing it to any layer
RUN --mount=type=secret,id=github_token \
    GITHUB_TOKEN=$(cat /run/secrets/github_token) \
    GONOSUMCHECK=github.com/myorg/* \
    GOFLAGS=-mod=mod \
    go mod download
```

Pass the secret at build time:

```bash
docker build --secret id=github_token,src=~/.github_token -t myapp .
```

**Warning:** Never use `ARG` for secrets — build args are visible in `docker history` and image metadata. Use BuildKit secret mounts for any sensitive value needed at build time.

---

## Non-Root User

Running as root inside a container is a security risk — a container escape would give root on the host. Always run as non-root.

**With distroless:nonroot — nothing extra needed:**

```dockerfile
FROM gcr.io/distroless/static-debian12:nonroot
# Already configured to run as UID 65532 (nonroot)
COPY --from=builder /app/server /server
ENTRYPOINT ["/server"]
```

**With Alpine — create a dedicated user:**

```dockerfile
FROM alpine:3.21

RUN apk add --no-cache ca-certificates tzdata && \
    addgroup -S appgroup && \
    adduser -S -G appgroup -u 10001 appuser

COPY --from=builder /app/server /app/server
RUN chown appuser:appgroup /app/server

USER appuser
ENTRYPOINT ["/app/server"]
```

**File permissions:** If the app needs to write files (e.g., temp uploads), ensure the target directory is owned by the app user or use a volume with appropriate permissions.

---

## HEALTHCHECK Instruction

The `HEALTHCHECK` Dockerfile instruction tells Docker how to test whether the container is healthy. Docker Engine (and docker-compose) uses this — Kubernetes uses its own probe configuration instead.

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD ["/server", "-health-check"]
```

**Problem:** distroless has no shell and no `curl`/`wget`. Use a dedicated health-check binary or a flag in the main binary.

**Recommended pattern — embed health check in the binary:**

```go
// cmd/api/main.go
import (
    "flag"
    "fmt"
    "net/http"
    "os"
)

func main() {
    healthCheck := flag.Bool("health-check", false, "run health check and exit")
    flag.Parse()

    if *healthCheck {
        url := fmt.Sprintf("http://localhost:%s/health", os.Getenv("PORT"))
        resp, err := http.Get(url)
        if err != nil {
            os.Exit(1)
        }
        defer resp.Body.Close()
        if resp.StatusCode != http.StatusOK {
            os.Exit(1)
        }
        os.Exit(0)
    }

    // ... normal server startup
}
```

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
    CMD ["/server", "-health-check"]
```

**Kubernetes note:** When deploying to K8s, the `HEALTHCHECK` instruction is ignored. K8s uses `livenessProbe` and `readinessProbe` in the pod spec. See `references/kubernetes.md`.

---

## Image Size Optimization

Techniques to minimize the final image size:

**1. Use `-ldflags="-s -w"`**

```bash
go build -ldflags="-s -w" ...
```

- `-s` strips the symbol table (~30% size reduction)
- `-w` strips DWARF debug info (additional ~10% reduction)

**2. Use `-trimpath`**

```bash
go build -trimpath ...
```

Removes local file system paths from the binary. Improves reproducibility and hides build machine paths.

**3. Use `upx` compression (optional, tradeoff)**

```dockerfile
FROM builder AS compressor
RUN apt-get install -y upx-ucl && upx --best /app/server
```

UPX can reduce binary size by 50-70% but adds startup latency (~100-300ms) due to in-memory decompression. Use only for extremely size-constrained environments.

**4. Avoid copying unnecessary files**

Use `.dockerignore` to exclude test files, documentation, and development tooling from the build context.

**Size comparison for a typical Gin API binary:**

| Configuration | Binary size | Final image |
|---------------|-------------|-------------|
| Default build | ~12 MB | ~14 MB (distroless) |
| `-ldflags="-s -w"` | ~8 MB | ~10 MB |
| `-ldflags="-s -w"` + UPX | ~3 MB | ~5 MB |

---

## .dockerignore

```dockerignore
# Version control
.git
.gitignore
.github

# Development tools
.air.toml
air.toml
.golangci.yml
Makefile

# Local environment
.env
.env.local
.env.*
!.env.example

# Docker files (not needed in build context)
docker-compose*.yml
Dockerfile*

# Documentation
*.md
docs/
plans/

# Test artifacts
*_test.go
coverage.html
coverage.out
*.test

# Editor
.vscode/
.idea/
*.swp
```

**Critical:** Always exclude `.env` files from the build context. Even if you don't `COPY` them explicitly, they exist in the build context and could be accidentally included by a broad `COPY . .` in future changes.

---

## Complete Production Dockerfile

```dockerfile
# syntax=docker/dockerfile:1
# Build: docker build --build-arg VERSION=$(git describe --tags --always) -t myapp:latest .

# ── Stage 1: Dependency cache ─────────────────────────────────────────────────
FROM golang:1.24-bookworm AS deps

WORKDIR /build

# Copy only module files first — cached until go.mod/go.sum change
COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download && go mod verify

# ── Stage 2: Builder ──────────────────────────────────────────────────────────
FROM deps AS builder

ARG VERSION=dev
ARG BUILD_TIME
ARG TARGETARCH

# Copy source code
COPY . .

# Build statically linked binary
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 GOOS=linux GOARCH=${TARGETARCH} go build \
    -ldflags="-s -w -X main.Version=${VERSION} -X main.BuildTime=${BUILD_TIME}" \
    -trimpath \
    -o /app/server \
    ./cmd/api

# Verify the binary was built correctly
RUN /app/server -health-check 2>/dev/null || true

# ── Stage 3: Runtime ──────────────────────────────────────────────────────────
FROM gcr.io/distroless/static-debian12:nonroot

# Metadata labels
LABEL org.opencontainers.image.source="https://github.com/myorg/myapp"
LABEL org.opencontainers.image.description="Gin REST API"
LABEL org.opencontainers.image.licenses="MIT"

COPY --from=builder /app/server /server

# Expose port (documentation only — actual binding via -p or K8s service)
EXPOSE 8080

# Health check for Docker Engine and docker-compose
HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
    CMD ["/server", "-health-check"]

# distroless:nonroot runs as UID 65532 — no USER directive needed
ENTRYPOINT ["/server"]
```

Build and run:

```bash
# Build
docker build \
  --build-arg VERSION=$(git describe --tags --always) \
  --build-arg BUILD_TIME=$(date -u +%Y-%m-%dT%H:%M:%SZ) \
  -t myapp:latest .

# Run locally
docker run --rm \
  -p 8080:8080 \
  -e DATABASE_URL="postgres://user:pass@host:5432/db?sslmode=require" \
  -e JWT_SECRET="your-secret" \
  -e GIN_MODE=release \
  myapp:latest

# Inspect image size
docker images myapp:latest
```
</file>

<file path="skills/golang-gin-deploy/references/kubernetes.md">
# Kubernetes Reference

This file covers Kubernetes deployment for Go Gin APIs: Deployment manifest, Service, ConfigMap, Secret, liveness/readiness probes, Horizontal Pod Autoscaler, Ingress, PVC for PostgreSQL, complete manifests, brief Helm chart structure, and a GitHub Actions CI/CD workflow. Use when deploying a Gin API to a Kubernetes cluster.

> **Architectural recommendation:** Kubernetes patterns are not part of the Gin framework. These are mainstream cloud-native Go community patterns.

## Table of Contents

1. [Deployment Manifest](#deployment-manifest)
2. [Service Manifest](#service-manifest)
3. [ConfigMap and Secret](#configmap-and-secret)
4. [Liveness and Readiness Probes](#liveness-and-readiness-probes)
5. [Horizontal Pod Autoscaler](#horizontal-pod-autoscaler)
6. [Ingress](#ingress)
7. [PVC for PostgreSQL](#pvc-for-postgresql)
8. [PodDisruptionBudget](#poddisruptionbudget)
9. [NetworkPolicy](#networkpolicy)
10. [Complete Manifests](#complete-manifests)
11. [Helm Chart Structure](#helm-chart-structure)
12. [GitHub Actions CI/CD Workflow](#github-actions-cicd-workflow)

---

## Deployment Manifest

The Deployment declares the desired state: which image to run, how many replicas, resource limits, and probe configuration.

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: myapp
  labels:
    app: myapp
    version: "1.0.0"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0    # never reduce below desired replicas during update
      maxSurge: 1          # allow one extra pod during rollout
  template:
    metadata:
      labels:
        app: myapp
    spec:
      # Graceful termination: allow 30s for in-flight requests to complete
      terminationGracePeriodSeconds: 30

      # Prevent all pods landing on same node
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: myapp
                topologyKey: kubernetes.io/hostname

      containers:
        - name: myapp
          image: ghcr.io/myorg/myapp:latest
          imagePullPolicy: Always
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP

          # Load non-sensitive config from ConfigMap
          envFrom:
            - configMapRef:
                name: myapp-config

          # Load sensitive values from Secret
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: myapp-secret
                  key: database-url
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: myapp-secret
                  key: jwt-secret
            - name: REDIS_URL
              valueFrom:
                secretKeyRef:
                  name: myapp-secret
                  key: redis-url

          # Resource requests/limits — tune for your workload
          resources:
            requests:
              cpu: 100m       # 0.1 CPU core
              memory: 64Mi
            limits:
              cpu: 500m       # 0.5 CPU core
              memory: 256Mi

          # Probes — see dedicated section below
          livenessProbe:
            httpGet:
              path: /health
              port: http
            initialDelaySeconds: 15
            periodSeconds: 20
            timeoutSeconds: 5
            failureThreshold: 3

          readinessProbe:
            httpGet:
              path: /health
              port: http
            initialDelaySeconds: 5
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3

          # Security context — run as non-root
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            runAsNonRoot: true
            runAsUser: 65532      # matches distroless:nonroot UID
            capabilities:
              drop: ["ALL"]

      # Pod-level security context
      securityContext:
        seccompProfile:
          type: RuntimeDefault
```

---

## Service Manifest

The Service exposes the Deployment as a stable network endpoint inside the cluster.

```yaml
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: myapp
  labels:
    app: myapp
spec:
  type: ClusterIP        # internal only — Ingress handles external traffic
  selector:
    app: myapp           # routes to pods with this label
  ports:
    - name: http
      port: 80           # service port (cluster-internal)
      targetPort: http   # named port from Deployment container
      protocol: TCP
```

**Service types:**

| Type | Use case |
|------|----------|
| `ClusterIP` | Default — internal access only (use with Ingress) |
| `NodePort` | Expose on each node's IP — dev/testing only |
| `LoadBalancer` | Cloud provider creates an external load balancer |

---

## ConfigMap and Secret

**ConfigMap — non-sensitive configuration:**

```yaml
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
  namespace: myapp
data:
  PORT: "8080"
  GIN_MODE: "release"
  MIGRATIONS_PATH: "db/migrations"
  READ_TIMEOUT: "10s"
  WRITE_TIMEOUT: "10s"
  SHUTDOWN_TIMEOUT: "30s"
```

**Secret — sensitive values:**

```yaml
# k8s/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secret
  namespace: myapp
type: Opaque
# Values must be base64-encoded
# echo -n 'postgres://user:pass@host:5432/db?sslmode=require' | base64
data:
  database-url: cG9zdGdyZXM6Ly91c2VyOnBhc3NAaG9zdDo1NDMyL2RiP3NzbG1vZGU9cmVxdWlyZQ==
  jwt-secret: eW91ci1zZWNyZXQtaGVyZQ==
  redis-url: cmVkaXM6Ly9yZWRpczo2Mzc5
```

**Critical:** Never commit real Secret values to git. Use a secrets manager (Vault, AWS Secrets Manager, GCP Secret Manager) and inject at deploy time via CI/CD, or use sealed-secrets / External Secrets Operator.

**Generate base64 values:**

```bash
echo -n 'your-value' | base64
# Decode to verify:
echo 'eW91ci12YWx1ZQ==' | base64 --decode
```

**Create secrets from CLI without manifest (avoids base64 in files):**

```bash
kubectl create secret generic myapp-secret \
  --from-literal=database-url="postgres://user:pass@host:5432/db?sslmode=require" \
  --from-literal=jwt-secret="your-jwt-secret" \
  --from-literal=redis-url="redis://redis:6379" \
  --namespace=myapp
```

---

## Liveness and Readiness Probes

Both probes hit the `/health` endpoint from the **golang-gin-api** skill's `HealthHandler`.

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: http
  initialDelaySeconds: 15   # wait 15s before first check (app startup time)
  periodSeconds: 20          # check every 20s
  timeoutSeconds: 5          # fail if no response in 5s
  failureThreshold: 3        # restart after 3 consecutive failures
  successThreshold: 1

readinessProbe:
  httpGet:
    path: /health
    port: http
  initialDelaySeconds: 5    # readiness can start sooner than liveness
  periodSeconds: 10
  timeoutSeconds: 3
  failureThreshold: 3        # remove from load balancer after 3 failures
  successThreshold: 1
```

**Probe behavior:**

| Probe | On failure | On success |
|-------|-----------|------------|
| `livenessProbe` | Restart the container | Nothing |
| `readinessProbe` | Remove from Service endpoints (stop receiving traffic) | Add to Service endpoints |

**When to split into separate endpoints:**

```go
// /health/live — process health only (no external deps)
r.GET("/health/live", func(c *gin.Context) {
    c.JSON(http.StatusOK, gin.H{"status": "ok"})
})

// /health/ready — can the app serve traffic? (checks DB)
r.GET("/health/ready", healthHandler.Check)
```

```yaml
livenessProbe:
  httpGet:
    path: /health/live
    port: http
  initialDelaySeconds: 5

readinessProbe:
  httpGet:
    path: /health/ready
    port: http
  initialDelaySeconds: 10
```

Use separate endpoints when: app startup takes longer than DB connection (avoids liveness killing a healthy-but-slow-starting pod).

**Startup probe (for slow-starting apps):**

```yaml
startupProbe:
  httpGet:
    path: /health/live
    port: http
  failureThreshold: 30  # 30 * 10s = 5 minutes max startup time
  periodSeconds: 10
```

Startup probe runs first. Only after it succeeds do liveness and readiness probes activate. Prevents liveness from killing a legitimately slow-starting pod.

---

## Horizontal Pod Autoscaler

Scale replicas based on CPU or memory utilization.

```yaml
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp
  namespace: myapp
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70   # scale up when avg CPU > 70%
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # wait 5 min before scaling down
      policies:
        - type: Pods
          value: 1
          periodSeconds: 60            # remove at most 1 pod per minute
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60            # add at most 2 pods per minute
```

**Critical:** `resources.requests` must be set in the Deployment for HPA to calculate utilization percentages. HPA uses `actual / request` to compute utilization.

---

## Ingress

Ingress routes external HTTP/HTTPS traffic to the Service. Requires an Ingress controller (nginx-ingress, Traefik, etc.) installed in the cluster.

```yaml
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  namespace: myapp
  annotations:
    # nginx-ingress specific — adjust for your controller
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "30"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "30"
    # TLS cert managed by cert-manager
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.example.com
      secretName: myapp-tls   # cert-manager creates this Secret
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp
                port:
                  name: http
```

---

## PVC for PostgreSQL

In production, use a managed database (RDS, Cloud SQL, Neon). For self-hosted PostgreSQL in Kubernetes, use a PersistentVolumeClaim.

```yaml
# k8s/postgres.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: myapp
spec:
  accessModes:
    - ReadWriteOnce      # single node access — suitable for stateful DB
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard   # use your cluster's StorageClass

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: myapp
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:17-alpine
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_DB
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: db-name
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: db-user
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: db-password
          volumeMounts:
            - name: postgres-storage
              mountPath: /var/lib/postgresql/data
          resources:
            requests:
              memory: 256Mi
              cpu: 100m
            limits:
              memory: 512Mi
              cpu: 500m
  volumeClaimTemplates:
    - metadata:
        name: postgres-storage
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi

---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: myapp
spec:
  type: ClusterIP
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
```

---

## PodDisruptionBudget

Ensures at least one pod remains available during voluntary disruptions (node drains, cluster upgrades). Without a PDB, a rolling upgrade or node drain can take all replicas offline simultaneously.

```yaml
# k8s/pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
  namespace: myapp
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: myapp
```

**When to use `minAvailable` vs `maxUnavailable`:**

| Setting | Behavior |
|---------|----------|
| `minAvailable: 1` | At least 1 pod stays up — safe for 2+ replicas |
| `maxUnavailable: 1` | At most 1 pod goes down at a time |

For 2 replicas, `minAvailable: 1` and `maxUnavailable: 1` are equivalent. With 3+ replicas, `minAvailable: 2` is more conservative.

**Critical:** PDB only protects against *voluntary* disruptions (kubectl drain, cluster upgrades). It does not prevent crashes from OOMKill or node failures.

---

## NetworkPolicy

Restricts which pods can send traffic to the application. Without a NetworkPolicy, any pod in the cluster can reach the app on port 8080.

```yaml
# k8s/netpol.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: myapp-netpol
  namespace: myapp
spec:
  podSelector:
    matchLabels:
      app: myapp
  policyTypes: [Ingress]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: ingress-nginx
      ports:
        - port: 8080
```

This policy allows inbound traffic only from the `ingress-nginx` namespace (the Ingress controller). All other ingress is denied by default once any NetworkPolicy selects the pod.

**Label the ingress-nginx namespace if not already done:**

```bash
kubectl label namespace ingress-nginx name=ingress-nginx
```

**Extend to allow inter-service communication:**

```yaml
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: ingress-nginx
      ports:
        - port: 8080
    - from:
        - podSelector:
            matchLabels:
              app: internal-service   # allow calls from a specific internal service
      ports:
        - port: 8080
```

**Note:** NetworkPolicy requires a CNI plugin that supports it (Calico, Cilium, Weave). Default GKE, EKS, and AKS clusters support NetworkPolicy when enabled at cluster creation.

---

## Complete Manifests

Apply all manifests at once:

```bash
# Create namespace
kubectl create namespace myapp

# Apply all manifests
kubectl apply -f k8s/ --namespace=myapp

# Verify rollout
kubectl rollout status deployment/myapp --namespace=myapp

# Check pods
kubectl get pods --namespace=myapp

# View logs
kubectl logs -l app=myapp --namespace=myapp --tail=100 -f

# Describe a pod (events, probes, status)
kubectl describe pod -l app=myapp --namespace=myapp
```

**Recommended k8s directory layout:**

```
k8s/
├── namespace.yaml
├── configmap.yaml
├── secret.yaml          # gitignored — created via CI/CD
├── deployment.yaml
├── service.yaml
├── ingress.yaml
├── hpa.yaml
├── pdb.yaml             # PodDisruptionBudget
├── netpol.yaml          # NetworkPolicy
├── migrate-job.yaml     # run before deploying new version
└── postgres/            # only if self-hosting postgres
    ├── statefulset.yaml
    ├── service.yaml
    └── secret.yaml
```

**Migration Job (run before deploying new app version):**

```yaml
# k8s/migrate-job.yaml
# NOTE: For Helm deployments, replace the static name with:
#   name: db-migrate-{{ .Values.image.tag }}
# For raw kubectl apply, use a timestamp suffix or generateName instead:
#   generateName: db-migrate-
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate-v1-0-0   # replace with your version tag (e.g. db-migrate-v1-2-3)
  namespace: myapp
spec:
  backoffLimit: 3
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
                  name: myapp-secret
                  key: database-url
          volumeMounts:
            - name: migrations
              mountPath: /migrations
      volumes:
        - name: migrations
          configMap:
            name: db-migrations
```

See the **golang-gin-database** skill (`references/migrations.md`) for the full migration strategy and zero-downtime patterns.

---

## Helm Chart Structure

Helm templates manifests with values, enabling environment-specific configuration without duplicating YAML.

```
charts/myapp/
├── Chart.yaml           # chart metadata
├── values.yaml          # default values
├── values.prod.yaml     # production overrides
├── templates/
│   ├── _helpers.tpl     # named templates (labels, selectors)
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── hpa.yaml
│   └── migrate-job.yaml
└── .helmignore
```

`Chart.yaml`:

```yaml
apiVersion: v2
name: myapp
description: Gin REST API
type: application
version: 0.1.0
appVersion: "1.0.0"
```

`values.yaml` (excerpt):

```yaml
image:
  repository: ghcr.io/myorg/myapp
  tag: latest
  pullPolicy: Always

replicaCount: 2

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  className: nginx
  host: api.example.com
  tls: true

resources:
  requests:
    cpu: 100m
    memory: 64Mi
  limits:
    cpu: 500m
    memory: 256Mi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilization: 70

config:
  ginMode: release
  port: "8080"
  migrationsPath: db/migrations
```

Install / upgrade:

```bash
# Install
helm install myapp ./charts/myapp \
  --namespace=myapp \
  --create-namespace \
  --values=charts/myapp/values.prod.yaml \
  --set image.tag=v1.2.3

# Upgrade (rolling deploy)
helm upgrade myapp ./charts/myapp \
  --namespace=myapp \
  --values=charts/myapp/values.prod.yaml \
  --set image.tag=v1.2.4

# Rollback to previous release
helm rollback myapp 1 --namespace=myapp
```

---

## GitHub Actions CI/CD Workflow

Build image, push to registry, run migrations, and deploy to Kubernetes.

```yaml
# .github/workflows/deploy.yml
name: Build and Deploy

on:
  push:
    branches: [main]
  workflow_dispatch:

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version: "1.24"
          cache: true

      - name: Run unit tests
        run: go test -v -race -cover ./...

  build:
    needs: test
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      image-digest: ${{ steps.build.outputs.digest }}

    steps:
      - uses: actions/checkout@v4

      - name: Log in to container registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=sha-
            type=ref,event=branch
            type=semver,pattern={{version}}

      - name: Build and push image
        id: build
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          build-args: |
            VERSION=${{ github.sha }}
            BUILD_TIME=${{ github.event.head_commit.timestamp }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Scan image for vulnerabilities
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }}
          severity: CRITICAL,HIGH
          exit-code: 1

  migrate:
    needs: build
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Install migrate CLI
        run: |
          go install -tags 'postgres' \
            github.com/golang-migrate/migrate/v4/cmd/migrate@latest

      - name: Run migrations
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
        run: |
          migrate -path db/migrations -database "$DATABASE_URL" up

  deploy:
    needs: [build, migrate]
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Set up kubectl
        uses: azure/setup-kubectl@v4

      - name: Configure kubeconfig
        run: |
          mkdir -p ~/.kube
          echo "${{ secrets.KUBECONFIG }}" | base64 --decode > ~/.kube/config
          chmod 600 ~/.kube/config

      - name: Deploy to Kubernetes
        env:
          IMAGE_TAG: sha-${{ github.sha }}
        run: |
          kubectl set image deployment/myapp \
            myapp=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${IMAGE_TAG} \
            --namespace=myapp

          kubectl rollout status deployment/myapp \
            --namespace=myapp \
            --timeout=5m

      - name: Verify deployment
        run: |
          kubectl get pods -l app=myapp --namespace=myapp
          kubectl get deployment myapp --namespace=myapp
```

**Required GitHub secrets:** `DATABASE_URL` (PostgreSQL connection string for migrations), `KUBECONFIG` (base64-encoded kubeconfig for cluster access).

**Deployment flow:** `push → test → build (image push) → migrate → deploy (kubectl set image) → verify (rollout status)`.
</file>

<file path="skills/golang-gin-swagger/references/annotations.md">
# annotations.md — Complete Swagger Annotation Reference

Full reference for swaggo/swag annotations. Covers all `@Param` types, response patterns, file uploads, enums from constants, model renaming, grouped responses, and multiple auth schemes.

## Table of Contents

1. [Annotation Order](#1-annotation-order)
2. [General API Info](#2-general-api-info)
3. [Param Patterns](#3-param-patterns)
4. [Response Patterns](#4-response-patterns)
5. [CRUD Handler Annotation Examples](#5-crud-handler-annotation-examples)
6. [Auth Endpoint Annotations](#6-auth-endpoint-annotations)
7. [Security Definitions](#7-security-definitions)
8. [Model Tags](#8-model-tags)
9. [Enums from Constants](#9-enums-from-constants)
10. [File Uploads](#10-file-uploads)
11. [Response Headers](#11-response-headers)
12. [Model Renaming](#12-model-renaming)
13. [Deprecating Endpoints](#13-deprecating-endpoints)
14. [Tag Metadata](#14-tag-metadata)
15. [Custom Extensions](#15-custom-extensions)

---

## 1. Annotation Order

Conventional order (not enforced, but expected by `swag fmt`):

```go
// FunctionName godoc        ← standard Go doc comment (REQUIRED)
//
// @Summary      Short title
// @Description  Longer description
// @ID           unique-operation-id
// @Tags         tag1,tag2
// @Accept       json
// @Produce      json
// @Security     BearerAuth
// @Param        ...
// @Success      ...
// @Failure      ...
// @Header       ...
// @Router       /path [method]
```

Always start with a Go doc comment (`// FunctionName godoc`) followed by an empty `//` line before `@Summary`. Without the doc comment, `swag fmt` rejects the annotations.

---

## 2. General API Info

Place once in `cmd/api/main.go`, directly before `main()`:

```go
// @title           My API
// @version         1.0
// @description     Production REST API

// @termsOfService  https://example.com/terms

// @contact.name    API Support
// @contact.email   support@example.com
// @contact.url     https://example.com/support

// @license.name    MIT
// @license.url     https://opensource.org/licenses/MIT

// @host            localhost:8080
// @BasePath        /api/v1
// @schemes         http https

// @externalDocs.description  Full API documentation
// @externalDocs.url          https://docs.example.com

// Tag metadata (shows description in Swagger UI sidebar)
// @tag.name         users
// @tag.description  Operations on user accounts

// Custom extensions (vendor-specific metadata)
// @x-custom-key     {"env": "production"}

// Security definitions (see Section 5)
// @securityDefinitions.apikey  BearerAuth
// @in                          header
// @name                        Authorization
// @description                 Enter: Bearer {token}
func main() { ... }
```

Only the first annotation block is used. Do not scatter general annotations across files.

---

## 3. Param Patterns

Syntax: `@Param <name> <in> <type> <required> "<description>" [attributes]`

### Path Parameters

```go
// @Param  id    path  string  true  "User ID (UUID)"
// @Param  slug  path  string  true  "URL-friendly slug"
```

**Critical:** Use `{id}` in `@Router` (OpenAPI style), not `:id` (Gin style).

### Query Parameters

```go
// @Param  page    query  int     false  "Page number"     default(1) minimum(1)
// @Param  limit   query  int     false  "Items per page"  default(20) minimum(1) maximum(100)
// @Param  role    query  string  false  "Filter by role"  Enums(admin, user, guest)
// @Param  q       query  string  false  "Search query"    minLength(1) maxLength(200)
// @Param  sort    query  string  false  "Sort field"      default(created_at)
// @Param  order   query  string  false  "Sort order"      Enums(asc, desc) default(desc)
// @Param  active  query  bool    false  "Active only"     default(true)
```

### Header Parameters

```go
// @Param  X-Request-ID   header  string  false  "Request tracing ID"
// @Param  Accept-Language  header  string  false  "Preferred language"  default(en)
```

### Body Parameters

```go
// JSON body — reference a named struct
// @Param  request  body  domain.CreateUserRequest  true  "Request body"

// Inline primitive (rare — prefer structs)
// @Param  data  body  string  true  "Raw text payload"
```

Only one `body` param per endpoint. The struct must be importable by swag (use `--parseInternal` for `internal/` packages).

### FormData Parameters

```go
// @Param  name   formData  string  true   "User name"
// @Param  email  formData  string  true   "Email address"
// @Param  age    formData  int     false  "Age"
```

---

## 4. Response Patterns

Syntax: `@Success <code> {<type>} <model> "<description>"`

### Single Object

```go
// @Success  200  {object}  domain.User             "OK"
// @Success  201  {object}  domain.User             "Created"
```

### Array

```go
// @Success  200  {array}  domain.User  "List of users"
```

### Paginated Response

Define a wrapper struct for pagination:

```go
type PaginatedResponse struct {
    Data       []domain.User `json:"data"`
    Page       int           `json:"page"        example:"1"`
    Limit      int           `json:"limit"       example:"20"`
    TotalCount int           `json:"total_count"  example:"150"`
    TotalPages int           `json:"total_pages"  example:"8"`
}

// @Success  200  {object}  PaginatedResponse  "Paginated list"
```

### No Content

```go
// @Success  204  "No content"
```

### Primitive Response

```go
// @Success  200  {string}   string  "Plain text response"
// @Success  200  {integer}  int     "Count"
// @Success  200  {boolean}  bool    "Status"
```

### Failure Responses

Document all possible error codes:

```go
// @Failure  400  {object}  domain.ErrorResponse  "Bad request"
// @Failure  401  {object}  domain.ErrorResponse  "Unauthorized"
// @Failure  403  {object}  domain.ErrorResponse  "Forbidden"
// @Failure  404  {object}  domain.ErrorResponse  "Not found"
// @Failure  409  {object}  domain.ErrorResponse  "Conflict"
// @Failure  422  {object}  domain.ErrorResponse  "Validation failed"
// @Failure  429  {object}  domain.ErrorResponse  "Rate limit exceeded"
// @Failure  500  {object}  domain.ErrorResponse  "Internal server error"
```

### Grouped Failure Codes

When multiple codes share the same response type:

```go
// @Failure  400,401,404,500  {object}  domain.ErrorResponse
```

---

## 5. CRUD Handler Annotation Examples

Complete annotations for all standard CRUD operations. The SKILL.md covers Create, GetByID, and List. This section adds Update, Patch, Delete, and auth endpoints.

### Update (PUT — full replacement)

```go
// Update godoc
//
// @Summary      Update user
// @Description  Replace all user fields
// @Tags         users
// @Accept       json
// @Produce      json
// @Security     BearerAuth
// @Param        id     path  string                   true  "User ID" format(uuid)
// @Param        input  body  domain.UpdateUserRequest  true  "Fields to update"
// @Success      200    {object}  domain.UserResponse
// @Failure      400    {object}  domain.ErrorResponse
// @Failure      401    {object}  domain.ErrorResponse
// @Failure      404    {object}  domain.ErrorResponse
// @Failure      500    {object}  domain.ErrorResponse
// @Router       /users/{id} [put]
func (h *UserHandler) Update(c *gin.Context) {}
```

### Patch (PATCH — partial update)

```go
// Patch godoc
//
// @Summary      Partially update user
// @Description  Update only the provided fields; omitted fields are unchanged
// @Tags         users
// @Accept       json
// @Produce      json
// @Security     BearerAuth
// @Param        id     path  string                  true  "User ID" format(uuid)
// @Param        input  body  domain.PatchUserRequest  true  "Fields to patch"
// @Success      200    {object}  domain.UserResponse
// @Failure      400    {object}  domain.ErrorResponse
// @Failure      401    {object}  domain.ErrorResponse
// @Failure      404    {object}  domain.ErrorResponse
// @Failure      500    {object}  domain.ErrorResponse
// @Router       /users/{id} [patch]
func (h *UserHandler) Patch(c *gin.Context) {}
```

### Delete

```go
// Delete godoc
//
// @Summary      Delete user
// @Description  Permanently remove a user account
// @Tags         users
// @Security     BearerAuth
// @Param        id  path  string  true  "User ID" format(uuid)
// @Success      204
// @Failure      401    {object}  domain.ErrorResponse
// @Failure      403    {object}  domain.ErrorResponse
// @Failure      404    {object}  domain.ErrorResponse
// @Failure      500    {object}  domain.ErrorResponse
// @Router       /users/{id} [delete]
func (h *UserHandler) Delete(c *gin.Context) {}
```

---

## 6. Auth Endpoint Annotations

Annotate login, register, and token refresh endpoints. These do not require `@Security` because they produce tokens (not consume them).

### Register

```go
// Register godoc
//
// @Summary      Register a new account
// @Description  Create a user account and return access + refresh tokens
// @Tags         auth
// @Accept       json
// @Produce      json
// @Param        request  body      domain.RegisterRequest  true  "Registration payload"
// @Success      201      {object}  domain.TokenResponse
// @Failure      400      {object}  domain.ErrorResponse
// @Failure      409      {object}  domain.ErrorResponse   "Email already registered"
// @Failure      500      {object}  domain.ErrorResponse
// @Router       /auth/register [post]
func (h *AuthHandler) Register(c *gin.Context) {}
```

### Login

```go
// Login godoc
//
// @Summary      Log in
// @Description  Authenticate with email and password; returns access + refresh tokens
// @Tags         auth
// @Accept       json
// @Produce      json
// @Param        request  body      domain.LoginRequest  true  "Login credentials"
// @Success      200      {object}  domain.TokenResponse
// @Failure      400      {object}  domain.ErrorResponse
// @Failure      401      {object}  domain.ErrorResponse  "Invalid credentials"
// @Failure      500      {object}  domain.ErrorResponse
// @Router       /auth/login [post]
func (h *AuthHandler) Login(c *gin.Context) {}
```

### Refresh Token

```go
// RefreshToken godoc
//
// @Summary      Refresh access token
// @Description  Exchange a valid refresh token for a new access token
// @Tags         auth
// @Accept       json
// @Produce      json
// @Param        request  body      domain.RefreshRequest  true  "Refresh token payload"
// @Success      200      {object}  domain.TokenResponse
// @Failure      400      {object}  domain.ErrorResponse
// @Failure      401      {object}  domain.ErrorResponse  "Refresh token expired or invalid"
// @Failure      500      {object}  domain.ErrorResponse
// @Router       /auth/refresh [post]
func (h *AuthHandler) RefreshToken(c *gin.Context) {}
```

**Supporting DTOs for auth annotations:**

```go
// domain/auth.go

type RegisterRequest struct {
    Name     string `json:"name"     example:"Jane Doe"        binding:"required,min=2,max=100"`
    Email    string `json:"email"    example:"jane@example.com" binding:"required,email"`
    Password string `json:"password" example:"s3cur3P@ss!"      binding:"required,min=8"`
}

type LoginRequest struct {
    Email    string `json:"email"    example:"jane@example.com" binding:"required,email"`
    Password string `json:"password" example:"s3cur3P@ss!"      binding:"required"`
}

type RefreshRequest struct {
    RefreshToken string `json:"refresh_token" example:"eyJhbGci..." binding:"required"`
}

type TokenResponse struct {
    AccessToken  string `json:"access_token"  example:"eyJhbGci..."`
    RefreshToken string `json:"refresh_token" example:"eyJhbGci..."`
    ExpiresIn    int    `json:"expires_in"    example:"3600"`
}
```

---

## 7. Security Definitions

### API Key (Bearer JWT)

```go
// In main.go:
// @securityDefinitions.apikey  BearerAuth
// @in                          header
// @name                        Authorization
// @description                 Enter: Bearer {token}

// On each protected handler:
// @Security  BearerAuth
```

### API Key (Custom Header)

```go
// @securityDefinitions.apikey  ApiKeyAuth
// @in                          header
// @name                        X-API-Key
// @description                 API key for service access
```

### Basic Auth

```go
// @securityDefinitions.basic  BasicAuth
```

### OAuth2

```go
// @securityDefinitions.oauth2.implicit       OAuth2Implicit
// @authorizationUrl                          https://auth.example.com/authorize
// @scope.read                                Read access
// @scope.write                               Write access

// @securityDefinitions.oauth2.accessCode     OAuth2AccessCode
// @tokenUrl                                  https://auth.example.com/token
// @authorizationUrl                          https://auth.example.com/authorize
```

### Multiple Schemes on One Endpoint

```go
// Both required (AND logic):
// @Security  BearerAuth
// @Security  ApiKeyAuth

// Either accepted (OR logic) — use double pipe:
// @Security  BearerAuth || ApiKeyAuth
```

---

## 8. Model Tags

Complete list of struct tags recognized by swag:

```go
type Example struct {
    // Value constraints
    Name   string  `json:"name"   example:"Jane"     minLength:"2" maxLength:"100"`
    Age    int     `json:"age"    example:"30"        minimum:"0"   maximum:"150"`
    Score  float64 `json:"score"  example:"9.5"       minimum:"0"   maximum:"10"`

    // Format hints
    ID        string    `json:"id"         example:"550e8400-..."  format:"uuid"`
    Email     string    `json:"email"      example:"j@example.com" format:"email"`
    Website   string    `json:"website"    example:"https://..."   format:"uri"`
    CreatedAt time.Time `json:"created_at" example:"2024-01-15T10:30:00Z" format:"date-time"`
    Birthday  string    `json:"birthday"   example:"1990-05-15"    format:"date"`

    // Enums
    Role   string `json:"role"   example:"admin"  enums:"admin,user,guest"`
    Status string `json:"status" example:"active" enums:"active,inactive,banned"`

    // Defaults
    Page  int `json:"page"  default:"1"`
    Limit int `json:"limit" default:"20"`

    // Type overrides (for types swag can't infer)
    UpdatedAt  CustomTime    `json:"updated_at"  swaggertype:"string"  format:"date-time"`
    ExternalID sql.NullInt64 `json:"external_id" swaggertype:"integer"`
    Tags       []big.Float   `json:"tags"        swaggertype:"array,number"`
    Metadata   interface{}   `json:"metadata"    swaggertype:"object"`

    // Exclusions
    PasswordHash string `json:"-"               swaggerignore:"true"`
    InternalFlag bool   `json:"internal_flag"   swaggerignore:"true"`

    // Read-only / Write-only
    ID2 string `json:"id2" readOnly:"true"`   // shown in response only
    Pwd string `json:"pwd" writeOnly:"true"`  // shown in request only (hidden in response)

    // Required
    Required string `json:"required" binding:"required"` // swag infers required from binding tag
}
```

---

## 9. Enums from Constants

Swag auto-detects Go `const` blocks and generates enum values:

```go
type OrderStatus string

const (
    OrderPending    OrderStatus = "pending"
    OrderProcessing OrderStatus = "processing"
    OrderShipped    OrderStatus = "shipped"
    OrderDelivered  OrderStatus = "delivered"
    OrderCancelled  OrderStatus = "cancelled"
)

type Order struct {
    ID     string      `json:"id"`
    Status OrderStatus `json:"status"` // swagger generates enum automatically
}
```

This works when `OrderStatus` is a named type with `const` values of that type in the same package.

---

## 10. File Uploads

### Single File

```go
// UploadAvatar godoc
//
// @Summary      Upload user avatar
// @Tags         users
// @Accept       multipart/form-data
// @Produce      json
// @Security     BearerAuth
// @Param        id    path      string  true  "User ID"
// @Param        file  formData  file    true  "Avatar image"
// @Success      200   {object}  domain.UploadResponse
// @Failure      400   {object}  domain.ErrorResponse
// @Router       /users/{id}/avatar [post]
func (h *UserHandler) UploadAvatar(c *gin.Context) { ... }
```

### Multiple Files

```go
// @Param  files  formData  file  true  "Upload files"  collection(multi)
```

### File + Form Fields

```go
// @Accept  multipart/form-data
// @Param   name   formData  string  true  "Document name"
// @Param   file   formData  file    true  "Document file"
```

---

## 11. Response Headers

Document headers returned with responses:

```go
// @Success  200  {object}  domain.User
// @Header   200  {string}  X-Request-ID   "Unique request identifier"
// @Header   200  {integer} X-RateLimit-Remaining  "Remaining requests"
// @Header   200  {string}  X-RateLimit-Reset      "Reset timestamp"
```

---

## 12. Model Renaming

Override the model name in Swagger output (useful for disambiguating types with the same name across packages):

```go
// UserResponse is the public representation of a user.
//
// @name UserResponse
type User struct {
    ID    string `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
}
```

In generated docs, this appears as `UserResponse` instead of `domain.User`.

---

## 13. Deprecating Endpoints

Mark an endpoint as deprecated in docs:

```go
// GetUserV1 godoc
//
// @Summary      Get user (deprecated)
// @Description  Use GET /api/v2/users/{id} instead
// @Deprecated
// @Tags         users
// @Router       /users/{id} [get]
func (h *UserHandler) GetUserV1(c *gin.Context) { ... }
```

The endpoint appears with a strikethrough in Swagger UI.

---

## 14. Tag Metadata

Add descriptions to tags in the Swagger UI sidebar. Place in the general API info block:

```go
// @tag.name         users
// @tag.description  Operations on user accounts

// @tag.name         auth
// @tag.description  Authentication and token management

// @tag.name         admin
// @tag.description  Administrative operations (requires admin role)
```

Tags without metadata still appear in Swagger UI, but without descriptions.

---

## 15. Custom Extensions

Use `@x-` prefix for vendor-specific or custom metadata. Values are JSON:

### General API Extensions

```go
// @x-logo  {"url": "https://example.com/logo.png", "altText": "API Logo"}
```

### Operation Extensions

```go
// ListUsers godoc
//
// @Summary      List users
// @x-codeSamples  [{"lang": "curl", "source": "curl -H 'Authorization: Bearer {token}' https://api.example.com/users"}]
// @Router       /users [get]
func (h *UserHandler) List(c *gin.Context) { ... }
```

### Field Extensions

```go
type User struct {
    Role string `json:"role" x-order:"1"`
}
```

Custom extensions appear in the raw `swagger.json` output and are consumed by tools that understand them (e.g., Redoc uses `x-logo`, `x-codeSamples`).
</file>

<file path="AGENTS.md">
# AGENTS.md

Multi-agent guidance for the `golang-gin-best-practices` repository.

## Repository Overview

6 Agent Skills for Go/Gin REST API development, published on skills.sh. Skills live under `skills/`, build docs under `docs/`.

## Agent Roles

### Skill Author
- Writes or updates `SKILL.md` and `references/*.md` files
- Must read `docs/GIN-API-REFERENCE.md` before writing any Gin code
- Must read `docs/SPECIFICATION.md` for content requirements
- Keeps `SKILL.md` under 500 lines; moves overflow to reference files

### Reviewer
- Verifies all Gin API calls against `docs/GIN-API-REFERENCE.md`
- Checks `ShouldBind*` usage (not `Bind*`), `gin.New()` (not `gin.Default()`), `c.Copy()` for goroutines
- Validates frontmatter fields in `SKILL.md`
- Runs quality checklist from `docs/SPECIFICATION.md`

### Packager
- Generates zip files under `skills/` (one per skill)
- Updates `metadata.json` version fields
- Ensures `README.md` paths match actual structure

## File Ownership

| Path | Owner |
|------|-------|
| `skills/*/SKILL.md` | Skill Author |
| `skills/*/references/*.md` | Skill Author |
| `skills/*/metadata.json` | Packager |
| `skills/*/README.md` | Packager |
| `skills/*.zip` | Packager |
| `docs/*` | Skill Author |
| `README.md` | Packager |

## Conventions

- All code examples must compile, handle errors, use `context.Context`, and use `log/slog`
- Use the shared `User` domain model across all skills (defined in `docs/SPECIFICATION.md`)
- Cross-skill references use skill name (e.g., "see the **golang-gin-api** skill"), not file paths
- Conventional commits: `feat:`, `fix:`, `docs:`, `refactor:`
</file>

<file path="CLAUDE.md">
# CLAUDE.md

Agent guidance for the `golang-gin-best-practices` repository.

## What This Repo Is

A collection of 6 Agent Skills for building production-grade REST APIs with Go and the Gin framework. Published on [skills.sh](https://skills.sh). Each skill lives under `skills/` and follows the [Agent Skills open standard](https://agentskills.io).

## Repository Structure

```
golang-gin-best-practices/
├── skills/
│   ├── golang-gin-api/          # Core REST API — routing, handlers, binding, errors
│   ├── golang-gin-auth/         # JWT auth, RBAC middleware, token lifecycle
│   ├── golang-gin-database/     # PostgreSQL with GORM/sqlx, repository pattern
│   ├── golang-gin-deploy/       # Docker, docker-compose, Kubernetes, CI/CD
│   ├── golang-gin-testing/      # Unit, integration, and e2e tests
│   └── golang-gin-swagger/      # Swagger/OpenAPI docs with swaggo/swag
├── CLAUDE.md             # This file (agent guidance)
├── AGENTS.md             # Multi-agent guidance
├── README.md             # Project overview
└── LICENSE               # MIT
```

## Skill Format

Each skill directory contains:
- `SKILL.md` — Primary skill file with YAML frontmatter (`name`, `description`, `license`, `metadata`). Under 500 lines. Loaded by agents automatically.
- `references/*.md` — Deep-dive reference files loaded on demand.
- `metadata.json` — Version, author, abstract, external references.
- `README.md` — Brief description and structure overview.

## Conventions

- **Gin API calls**: Must match official Gin documentation exactly. Never invent methods.
- **Binding**: Use `ShouldBind*` (not `Bind*`) in all examples.
- **Server setup**: Use `gin.New()` + explicit `r.Use(...)` (not `gin.Default()`).
- **Goroutine safety**: Call `c.Copy()` before passing `*gin.Context` to goroutines.
- **Logging**: Use `log/slog` (not `fmt.Println` or `log.Println`).
- **Context**: Pass `c.Request.Context()` to all downstream blocking calls.
- **Handlers**: Thin — bind input, call service, format response. No DB calls in handlers.

## Modifying Skills

1. Read official Gin documentation before writing any Gin code
2. Keep `SKILL.md` under 500 lines — move detail to `references/`
3. Use `ShouldBind*`, `gin.New()`, `c.Copy()`, `log/slog`, and `context.Context` consistently
</file>

<file path="skills/golang-gin-database/metadata.json">
{
  "version": "1.0.4",
  "organization": "henriqueatila",
  "date": "March 2026",
  "abstract": "PostgreSQL integration for Go Gin APIs using GORM and sqlx — repository pattern, connection retry with backoff, cursor/keyset pagination, context-based transactions, TLS/sslmode, migrations, and dependency injection.",
  "references": [
    "https://gin-gonic.com/docs/",
    "https://gorm.io/docs/",
    "https://pkg.go.dev/github.com/jmoiron/sqlx",
    "https://context7.com/websites/gin-gonic_en"
  ]
}
</file>

<file path="skills/golang-gin-deploy/references/observability.md">
# Observability Reference

This file covers OpenTelemetry (OTel) instrumentation for Go Gin APIs: distributed tracing, RED metrics, and log-trace correlation. Use when adding observability to a Gin service, configuring an OTLP exporter, or setting up a local Jaeger stack. Covers the full pipeline from SDK initialization to graceful shutdown.

> **Architectural recommendation:** OTel Go SDK is stable at v1.x. Pin your `go.opentelemetry.io/otel` and `go.opentelemetry.io/contrib` versions explicitly — contrib packages release independently and may break without a pin.

## Table of Contents

1. [OTel SDK Setup](#otel-sdk-setup)
2. [Gin Middleware](#gin-middleware)
3. [Manual Spans in Service Layer](#manual-spans-in-service-layer)
4. [Error Recording](#error-recording)
5. [Metrics Middleware](#metrics-middleware)
6. [Log-Trace Correlation](#log-trace-correlation)
7. [Sampling](#sampling)
8. [Docker Compose — Local Observability Stack](#docker-compose--local-observability-stack)
9. [Graceful Shutdown](#graceful-shutdown)

---

## OTel SDK Setup

The OTel SDK uses a provider → exporter → processor pipeline. Initialize both `TracerProvider` and `MeterProvider` once at startup and register them as globals.

**Why OTLP gRPC:** The OTLP protocol is the vendor-neutral standard. It works with Jaeger, Grafana Tempo, Honeycomb, Datadog, and any OTel Collector. gRPC is preferred over HTTP for lower overhead in high-throughput services.

```go
// internal/telemetry/telemetry.go
package telemetry

import (
	"context"
	"fmt"
	"time"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
	"go.opentelemetry.io/otel/exporters/otlp/otlpmetric/otlpmetricgrpc"
	"go.opentelemetry.io/otel/propagation"
	sdkmetric "go.opentelemetry.io/otel/sdk/metric"
	"go.opentelemetry.io/otel/sdk/resource"
	sdktrace "go.opentelemetry.io/otel/sdk/trace"
	semconv "go.opentelemetry.io/otel/semconv/v1.26.0"
	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
)

// Providers holds the initialized providers for graceful shutdown.
type Providers struct {
	TracerProvider *sdktrace.TracerProvider
	MeterProvider  *sdkmetric.MeterProvider
}

// Init initializes OTel tracing and metrics, registers globals, and returns
// providers for shutdown. otlpEndpoint is host:port (e.g. "localhost:4317").
func Init(ctx context.Context, serviceName, serviceVersion, otlpEndpoint string) (*Providers, error) {
	res, err := resource.New(ctx,
		resource.WithAttributes(
			semconv.ServiceName(serviceName),
			semconv.ServiceVersion(serviceVersion),
		),
		resource.WithFromEnv(),   // picks up OTEL_RESOURCE_ATTRIBUTES
		resource.WithProcess(),   // pid, executable name
		resource.WithOS(),
	)
	if err != nil {
		return nil, fmt.Errorf("telemetry: create resource: %w", err)
	}

	conn, err := grpc.NewClient(otlpEndpoint,
		grpc.WithTransportCredentials(insecure.NewCredentials()), // use TLS in production
	)
	if err != nil {
		return nil, fmt.Errorf("telemetry: dial otlp endpoint %s: %w", otlpEndpoint, err)
	}

	// ── Tracing ──────────────────────────────────────────────────────────────
	traceExporter, err := otlptracegrpc.New(ctx, otlptracegrpc.WithGRPCConn(conn))
	if err != nil {
		return nil, fmt.Errorf("telemetry: create trace exporter: %w", err)
	}

	tp := sdktrace.NewTracerProvider(
		sdktrace.WithBatcher(traceExporter),
		sdktrace.WithResource(res),
		sdktrace.WithSampler(productionSampler()),
	)
	otel.SetTracerProvider(tp)
	otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
		propagation.TraceContext{}, // W3C traceparent header
		propagation.Baggage{},
	))

	// ── Metrics ──────────────────────────────────────────────────────────────
	metricExporter, err := otlpmetricgrpc.New(ctx, otlpmetricgrpc.WithGRPCConn(conn))
	if err != nil {
		return nil, fmt.Errorf("telemetry: create metric exporter: %w", err)
	}

	mp := sdkmetric.NewMeterProvider(
		sdkmetric.WithReader(sdkmetric.NewPeriodicReader(metricExporter,
			sdkmetric.WithInterval(15*time.Second),
		)),
		sdkmetric.WithResource(res),
	)
	otel.SetMeterProvider(mp)

	return &Providers{TracerProvider: tp, MeterProvider: mp}, nil
}

func productionSampler() sdktrace.Sampler {
	// See Sampling section for full explanation.
	return sdktrace.ParentBased(sdktrace.TraceIDRatioBased(0.1))
}
```

**go.mod dependencies:**

```
go.opentelemetry.io/otel v1.33.0
go.opentelemetry.io/otel/sdk v1.33.0
go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc v1.33.0
go.opentelemetry.io/otel/exporters/otlp/otlpmetric/otlpmetricgrpc v1.33.0
go.opentelemetry.io/contrib/instrumentation/github.com/gin-gonic/gin/otelgin v0.58.0
google.golang.org/grpc v1.69.0
```

---

## Gin Middleware

`otelgin.Middleware` automatically creates a span for every HTTP request, sets standard HTTP attributes (`http.method`, `http.route`, `http.status_code`), and propagates W3C `traceparent` headers from incoming requests.

**Why register before other middleware:** The span wraps the full request lifecycle. Register `otelgin` first so that any subsequent middleware (auth, logging) runs inside the span.

```go
// cmd/api/main.go
package main

import (
	"log/slog"
	"net/http"
	"os"

	"github.com/gin-gonic/gin"
	"go.opentelemetry.io/contrib/instrumentation/github.com/gin-gonic/gin/otelgin"
)

func setupRouter(serviceName string, logger *slog.Logger) *gin.Engine {
	r := gin.New()

	// OTel tracing — must be first to wrap entire request lifecycle
	r.Use(otelgin.Middleware(serviceName))

	// Recovery and logging after otelgin so they run within the trace span
	r.Use(middleware.Recovery(logger))
	r.Use(requestLogger(logger))

	return r
}

func requestLogger(logger *slog.Logger) gin.HandlerFunc {
	return func(c *gin.Context) {
		c.Next()
		logger.InfoContext(c.Request.Context(), "request",
			"method", c.Request.Method,
			"path",   c.Request.URL.Path,
			"status", c.Writer.Status(),
		)
	}
}
```

**Route group example:**

```go
v1 := r.Group("/api/v1")
{
	v1.GET("/users/:id", userHandler.GetByID)
	v1.POST("/users", userHandler.Create)
}
```

`otelgin` uses the registered route pattern (`/users/:id`) as the span name, not the resolved path — preventing high-cardinality spans.

---

## Manual Spans in Service Layer

Automatic middleware creates one span per HTTP request. Create child spans for significant operations in the service and repository layers to get call-level visibility.

**Why:** Without manual spans, you see "POST /orders took 400ms" but not whether it was the DB query, an external HTTP call, or business logic that was slow.

```go
// internal/service/order_service.go
package service

import (
	"context"
	"fmt"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/trace"
)

// tracer is package-level; name matches the instrumentation scope.
var tracer = otel.Tracer("myapp/service/order")

type OrderService struct {
	repo OrderRepository
}

func (s *OrderService) CreateOrder(ctx context.Context, userID string, items []Item) (*Order, error) {
	ctx, span := tracer.Start(ctx, "OrderService.CreateOrder",
		trace.WithAttributes(
			attribute.String("order.user_id", userID),
			attribute.Int("order.item_count", len(items)),
		),
	)
	defer span.End()

	order, err := s.repo.Insert(ctx, userID, items) // ctx carries the span
	if err != nil {
		return nil, fmt.Errorf("create order: %w", err)
	}

	span.SetAttributes(attribute.String("order.id", order.ID))
	return order, nil
}
```

**Key rules:**
- Always `defer span.End()` immediately after `tracer.Start` — never forget to close a span
- Pass `ctx` (not `c.Request.Context()` — that's only in handlers) down the call chain
- Name spans as `"Package.Method"` for easy filtering in Jaeger

---

## Error Recording

A span's status is `Unset` by default — not OK, not Error. Set it explicitly so trace UIs can filter on failed spans.

```go
// internal/service/order_service.go
import (
	"context"
	"fmt"

	"go.opentelemetry.io/otel/codes"
	"go.opentelemetry.io/otel/trace"
)

func (s *OrderService) CreateOrder(ctx context.Context, userID string, items []Item) (*Order, error) {
	ctx, span := tracer.Start(ctx, "OrderService.CreateOrder")
	defer span.End()

	order, err := s.repo.Insert(ctx, userID, items)
	if err != nil {
		// RecordError attaches the error as a span event with stack trace.
		span.RecordError(err)
		// SetStatus marks the span as failed — visible in Jaeger's error filter.
		span.SetStatus(codes.Error, err.Error())
		return nil, fmt.Errorf("create order: %w", err)
	}

	span.SetStatus(codes.Ok, "")
	return order, nil
}
```

**RecordError vs SetStatus:**
- `span.RecordError(err)` — attaches error details as a structured span event (message, stack)
- `span.SetStatus(codes.Error, msg)` — marks the overall span status; used by trace UIs for error highlighting and alerting

Always call both. `RecordError` alone does not mark the span as failed in the UI.

---

## Metrics Middleware

OTel metrics expose RED metrics (Rate, Errors, Duration) for every route. Use the global `MeterProvider` (registered in `Init`) to create instruments once at startup.

**Why custom middleware over otelgin metrics:** `otelgin` focuses on tracing. Custom metric middleware gives control over metric names, label cardinality, and histogram bucket boundaries.

```go
// internal/middleware/metrics_middleware.go
package middleware

import (
	"net/http"
	"strconv"
	"time"

	"github.com/gin-gonic/gin"
	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/metric"
)

type HTTPMetrics struct {
	requestDuration metric.Float64Histogram
	requestsTotal   metric.Int64Counter
}

// NewHTTPMetrics creates OTel metric instruments. Call once at startup.
func NewHTTPMetrics() (*HTTPMetrics, error) {
	meter := otel.Meter("myapp/http")

	duration, err := meter.Float64Histogram(
		"http.server.request.duration",
		metric.WithDescription("HTTP request duration in seconds"),
		metric.WithUnit("s"),
		metric.WithExplicitBucketBoundaries(
			0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10,
		),
	)
	if err != nil {
		return nil, err
	}

	total, err := meter.Int64Counter(
		"http.server.requests.total",
		metric.WithDescription("Total HTTP requests"),
	)
	if err != nil {
		return nil, err
	}

	return &HTTPMetrics{requestDuration: duration, requestsTotal: total}, nil
}

// Handler returns a Gin middleware that records request metrics.
func (m *HTTPMetrics) Handler() gin.HandlerFunc {
	return func(c *gin.Context) {
		start := time.Now()

		c.Next()

		attrs := []attribute.KeyValue{
			attribute.String("http.method", c.Request.Method),
			attribute.String("http.route", c.FullPath()), // pattern, not resolved path
			attribute.String("http.status_code", strconv.Itoa(c.Writer.Status())),
		}

		m.requestDuration.Record(c.Request.Context(),
			time.Since(start).Seconds(),
			metric.WithAttributes(attrs...),
		)
		m.requestsTotal.Add(c.Request.Context(), 1,
			metric.WithAttributes(attrs...),
		)
	}
}
```

Register in `main.go`:

```go
httpMetrics, err := middleware.NewHTTPMetrics()
if err != nil {
	logger.Error("failed to create http metrics", "error", err)
	os.Exit(1)
}

r := setupRouter(cfg.ServiceName, logger)
r.Use(httpMetrics.Handler())
```

**Cardinality warning:** Never use `c.Request.URL.Path` (resolved path) as a label — it creates unbounded cardinality (one label value per unique ID). Always use `c.FullPath()` (route pattern like `/users/:id`).

---

## Log-Trace Correlation

Connecting logs to traces lets you jump from a log line to the full trace in Jaeger. Inject `trace_id` and `span_id` into every log record produced within a request.

**Why slog:** `log/slog` supports structured key-value logging and a composable `Handler` interface, making trace injection clean without third-party libraries.

```go
// internal/telemetry/trace_log_handler.go
package telemetry

import (
	"context"
	"log/slog"

	"go.opentelemetry.io/otel/trace"
)

// TraceLogHandler wraps any slog.Handler and injects trace_id/span_id
// from the context into every log record.
type TraceLogHandler struct {
	inner slog.Handler
}

func NewTraceLogHandler(inner slog.Handler) *TraceLogHandler {
	return &TraceLogHandler{inner: inner}
}

func (h *TraceLogHandler) Enabled(ctx context.Context, level slog.Level) bool {
	return h.inner.Enabled(ctx, level)
}

func (h *TraceLogHandler) Handle(ctx context.Context, r slog.Record) error {
	if span := trace.SpanFromContext(ctx); span.SpanContext().IsValid() {
		sc := span.SpanContext()
		r.AddAttrs(
			slog.String("trace_id", sc.TraceID().String()),
			slog.String("span_id", sc.SpanID().String()),
		)
	}
	return h.inner.Handle(ctx, r)
}

func (h *TraceLogHandler) WithAttrs(attrs []slog.Attr) slog.Handler {
	return &TraceLogHandler{inner: h.inner.WithAttrs(attrs)}
}

func (h *TraceLogHandler) WithGroup(name string) slog.Handler {
	return &TraceLogHandler{inner: h.inner.WithGroup(name)}
}
```

Initialize in `main.go`:

```go
baseHandler := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelInfo})
logger := slog.New(telemetry.NewTraceLogHandler(baseHandler))
slog.SetDefault(logger)
```

Log within a handler using `c.Request.Context()`:

```go
func (h *UserHandler) GetByID(c *gin.Context) {
	user, err := h.svc.GetUser(c.Request.Context(), c.Param("id"))
	if err != nil {
		// trace_id and span_id are injected automatically
		h.logger.ErrorContext(c.Request.Context(), "get user failed",
			"user_id", c.Param("id"),
			"error", err,
		)
		c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
		return
	}
	c.JSON(http.StatusOK, user)
}
```

Example log output (JSON):

```json
{
  "time": "2024-11-01T10:00:00Z",
  "level": "ERROR",
  "msg": "get user failed",
  "user_id": "123",
  "error": "sql: no rows in result set",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7"
}
```

---

## Sampling

Sampling controls what fraction of traces are exported. Sampling 100% of traces in production is expensive — a high-traffic API can generate gigabytes of trace data per hour.

**Strategy:** Use `ParentBased(TraceIDRatioBased)` so that:
- If an upstream service already started a trace (W3C `traceparent` header present), respect its sampling decision — don't drop a trace mid-flight
- For root spans (no parent), sample at a configured ratio

```go
// internal/telemetry/sampler.go
package telemetry

import (
	"os"
	"strconv"

	sdktrace "go.opentelemetry.io/otel/sdk/trace"
)

// SamplerFromEnv reads OTEL_SAMPLING_RATE (0.0–1.0).
// Defaults to 1.0 (100%) when unset — good for development.
// Set to 0.1 in production for 10% sampling.
func SamplerFromEnv() sdktrace.Sampler {
	rate := 1.0
	if v := os.Getenv("OTEL_SAMPLING_RATE"); v != "" {
		if parsed, err := strconv.ParseFloat(v, 64); err == nil {
			rate = parsed
		}
	}
	return sdktrace.ParentBased(sdktrace.TraceIDRatioBased(rate))
}
```

Use in `Init`:

```go
tp := sdktrace.NewTracerProvider(
	sdktrace.WithBatcher(traceExporter),
	sdktrace.WithResource(res),
	sdktrace.WithSampler(SamplerFromEnv()),
)
```

| Environment | `OTEL_SAMPLING_RATE` | Rationale |
|-------------|----------------------|-----------|
| Development | `1.0` (default) | See every trace while debugging |
| Staging | `0.5` | 50% — catch issues without full cost |
| Production | `0.1` | 10% — statistically representative |

**Head-based vs tail-based:** `TraceIDRatioBased` is head-based (decision made at trace start). For tail-based sampling (keep all error traces, sample successful ones), use an OTel Collector with the `tail_sampling` processor.

---

## Docker Compose — Local Observability Stack

Add Jaeger (all-in-one) and an OTel Collector to the existing docker-compose. The Collector receives OTLP from the app and forwards to Jaeger.

```yaml
# docker-compose.observability.yml
# Run alongside the main docker-compose.yml:
#   docker compose -f docker-compose.yml -f docker-compose.observability.yml up

services:
  otel-collector:
    image: otel/opentelemetry-collector-contrib:0.117.0
    command: ["--config=/etc/otel/config.yaml"]
    volumes:
      - ./otel-collector-config.yaml:/etc/otel/config.yaml:ro
    ports:
      - "4317:4317"   # OTLP gRPC — app sends traces/metrics here
      - "4318:4318"   # OTLP HTTP
      - "8888:8888"   # Collector self-metrics (Prometheus scrape)
    depends_on:
      - jaeger

  jaeger:
    image: jaegertracing/all-in-one:1.65
    environment:
      - COLLECTOR_OTLP_ENABLED=true
    ports:
      - "16686:16686"  # Jaeger UI
      - "4317"         # OTLP gRPC — internal only (used by collector)
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:16686"]
      interval: 10s
      timeout: 5s
      retries: 5
```

OTel Collector config:

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 5s
    send_batch_size: 512

exporters:
  otlp/jaeger:
    endpoint: jaeger:4317
    tls:
      insecure: true

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp/jaeger]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp/jaeger]
```

App environment variables:

```yaml
# in docker-compose.yml app service
environment:
  - OTLP_ENDPOINT=otel-collector:4317
  - OTEL_SAMPLING_RATE=1.0   # 100% in local dev
```

Access Jaeger UI at `http://localhost:16686` after `docker compose up`.

---

## Graceful Shutdown

OTel providers buffer telemetry in memory and flush to the exporter in batches. Skipping shutdown on process exit drops the last batch of spans and metrics — losing visibility into the final seconds before shutdown.

**Why this matters:** Kubernetes sends `SIGTERM`, waits (default 30s), then `SIGKILL`. Proper shutdown ensures all buffered traces from in-flight requests are exported before the process exits.

```go
// cmd/api/main.go
package main

import (
	"context"
	"errors"
	"log/slog"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/gin-gonic/gin"
	"go.opentelemetry.io/contrib/instrumentation/github.com/gin-gonic/gin/otelgin"
	"myapp/internal/config"
	"myapp/internal/telemetry"
)

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	cfg, err := config.Load()
	if err != nil {
		logger.Error("config load failed", "error", err)
		os.Exit(1)
	}

	// Initialize OTel — must happen before any tracer/meter is used
	ctx := context.Background()
	providers, err := telemetry.Init(ctx, cfg.ServiceName, cfg.Version, cfg.OTLPEndpoint)
	if err != nil {
		logger.Error("telemetry init failed", "error", err)
		os.Exit(1)
	}

	r := gin.New()
	r.Use(otelgin.Middleware(cfg.ServiceName))
	// ... register routes

	srv := &http.Server{
		Addr:              ":" + cfg.Port,
		Handler:           r,
		ReadHeaderTimeout: 5 * time.Second,  // Slowloris protection
		IdleTimeout:       120 * time.Second,
	}

	// Start server in background
	go func() {
		if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
			logger.Error("server error", "error", err)
			os.Exit(1)
		}
	}()

	logger.Info("server started", "port", cfg.Port)

	// Wait for OS signal
	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	<-quit

	logger.Info("shutting down")

	// Give in-flight requests time to complete
	shutdownCtx, cancel := context.WithTimeout(context.Background(), cfg.ShutdownTimeout)
	defer cancel()

	if err := srv.Shutdown(shutdownCtx); err != nil {
		logger.Error("server shutdown error", "error", err)
	}

	// Flush and close OTel providers — order matters: tracer before meter
	if err := providers.TracerProvider.Shutdown(shutdownCtx); err != nil {
		logger.Error("tracer provider shutdown error", "error", err)
	}
	if err := providers.MeterProvider.Shutdown(shutdownCtx); err != nil {
		logger.Error("meter provider shutdown error", "error", err)
	}

	logger.Info("shutdown complete")
}
```

**Shutdown order:** HTTP server first (stops accepting new requests) → OTel providers (flushes buffered telemetry from completed requests). Reversing the order risks flushing incomplete spans.

**Timeout budget:** `ShutdownTimeout` (typically 25–28s when K8s terminationGracePeriodSeconds is 30s) must cover both HTTP drain and OTel flush. The OTel batcher flushes within the timeout or drops remaining data.
</file>

<file path="skills/golang-gin-deploy/metadata.json">
{
  "version": "1.0.3",
  "organization": "henriqueatila",
  "date": "March 2026",
  "abstract": "Containerization and deployment for Go Gin APIs — multi-stage Docker, Kubernetes (PDB, NetworkPolicy, HPA), Trivy image scanning, OpenTelemetry observability, graceful shutdown, and GitHub Actions CI/CD.",
  "references": [
    "https://gin-gonic.com/docs/",
    "https://docs.docker.com/",
    "https://kubernetes.io/docs/",
    "https://context7.com/websites/gin-gonic_en"
  ]
}
</file>

<file path="skills/golang-gin-swagger/metadata.json">
{
  "version": "1.0.3",
  "organization": "henriqueatila",
  "date": "March 2026",
  "abstract": "Swagger/OpenAPI documentation for Go Gin APIs using swaggo/swag — full CRUD + auth endpoint annotations, model tags, Swagger UI setup, JWT security definitions, and CI/CD integration.",
  "references": [
    "https://github.com/swaggo/swag",
    "https://github.com/swaggo/gin-swagger",
    "https://pkg.go.dev/github.com/swaggo/gin-swagger",
    "https://swagger.io/specification/v2/"
  ]
}
</file>

<file path="skills/golang-gin-testing/references/integration-tests.md">
# Integration Tests — Real Database with testcontainers

This file covers: test database setup with testcontainers-go, `TestMain` for DB lifecycle, cleanup between tests, repository integration tests, API integration tests with a running router, build tags, and fixture loading.

**When to use:** Write integration tests when you need to verify actual SQL behavior — migrations, GORM/sqlx queries, constraint enforcement, transaction rollbacks. Unit tests with mocked repositories cannot catch these.

## Table of Contents

1. [Dependencies](#dependencies)
2. [Build Tags](#build-tags)
3. [TestMain — DB Lifecycle](#testmain--db-lifecycle)
4. [Test Database Helper](#test-database-helper)
5. [Cleanup Between Tests](#cleanup-between-tests)
6. [Repository Integration Tests](#repository-integration-tests)
7. [API Integration Tests](#api-integration-tests)
8. [Testing Migrations](#testing-migrations)
9. [Fixture Loading](#fixture-loading)

---

## Dependencies

```bash
go get github.com/testcontainers/testcontainers-go
go get github.com/testcontainers/testcontainers-go/modules/postgres
go get gorm.io/driver/postgres
go get github.com/golang-migrate/migrate/v4
```

Requires Docker running on the host (or in CI). Each test run starts a real PostgreSQL container.

---

## Build Tags

Isolate integration tests from the unit test run using build tags. This keeps `go test ./...` fast.

```go
//go:build integration
// +build integration
```

Place this at the **top of every integration test file**, before the `package` declaration.

```bash
# Run unit tests only (fast, default CI step)
go test -v -race -cover -tags='!integration' ./...

# Run integration tests (requires Docker)
go test -v -race -tags=integration ./internal/repository/...

# Run all tests
go test -v -race -tags=integration ./...
```

---

## TestMain — DB Lifecycle

`TestMain` runs once per package, starts the container, runs all tests, then cleans up. This amortizes container startup cost across all tests in the package.

```go
//go:build integration

// internal/repository/integration_test.go
package repository_test

import (
    "context"
    "fmt"
    "log/slog"
    "os"
    "testing"
    "time"

    "github.com/testcontainers/testcontainers-go"
    tcpostgres "github.com/testcontainers/testcontainers-go/modules/postgres"
    "github.com/testcontainers/testcontainers-go/wait"
    gormpostgres "gorm.io/driver/postgres"
    "gorm.io/gorm"

    "myapp/internal/repository"
)

var (
    testDB        *gorm.DB
    testContainer testcontainers.Container
)

func TestMain(m *testing.M) {
    ctx := context.Background()

    // Start PostgreSQL container
    pgContainer, err := tcpostgres.RunContainer(ctx,
        testcontainers.WithImage("postgres:16-alpine"),
        tcpostgres.WithDatabase("testdb"),
        tcpostgres.WithUsername("testuser"),
        tcpostgres.WithPassword("testpass"),
        testcontainers.WithWaitStrategy(
            wait.ForLog("database system is ready to accept connections").
                WithOccurrence(2).
                WithStartupTimeout(30*time.Second),
        ),
    )
    if err != nil {
        slog.Error("failed to start postgres container", "error", err)
        os.Exit(1)
    }
    testContainer = pgContainer

    // Build DSN from container
    host, err := pgContainer.Host(ctx)
    if err != nil {
        slog.Error("failed to get container host", "error", err)
        cleanupContainer(ctx)
        os.Exit(1)
    }
    port, err := pgContainer.MappedPort(ctx, "5432")
    if err != nil {
        slog.Error("failed to get mapped port", "error", err)
        cleanupContainer(ctx)
        os.Exit(1)
    }
    dsn := fmt.Sprintf("host=%s port=%s user=testuser password=testpass dbname=testdb sslmode=disable",
        host, port.Port())

    // Connect with GORM
    db, err := gorm.Open(gormpostgres.Open(dsn), &gorm.Config{})
    if err != nil {
        slog.Error("failed to connect to test db", "error", err)
        cleanupContainer(ctx)
        os.Exit(1)
    }
    testDB = db

    // Run migrations
    if err := runTestMigrations(db); err != nil {
        slog.Error("migrations failed", "error", err)
        cleanupContainer(ctx)
        os.Exit(1)
    }

    // Run all tests in this package
    code := m.Run()

    // Cleanup
    cleanupContainer(ctx)
    os.Exit(code)
}

func cleanupContainer(ctx context.Context) {
    if testContainer != nil {
        if err := testContainer.Terminate(ctx); err != nil {
            slog.Error("failed to terminate container", "error", err)
        }
    }
}

func runTestMigrations(db *gorm.DB) error {
    // Auto-migrate models for integration tests
    return db.AutoMigrate(&repository.UserModel{})
}
```

---

## Test Database Helper

Encapsulate container setup in a reusable helper so multiple packages can share the same pattern.

```go
//go:build integration

// internal/testutil/testdb/postgres_container.go
package testdb

import (
    "context"
    "fmt"
    "testing"
    "time"

    "github.com/testcontainers/testcontainers-go"
    "github.com/testcontainers/testcontainers-go/modules/postgres"
    "github.com/testcontainers/testcontainers-go/wait"
    pgdriver "gorm.io/driver/postgres"
    "gorm.io/gorm"
)

// NewPostgres starts a PostgreSQL container and returns a connected *gorm.DB.
// The container is terminated when the test completes via t.Cleanup.
func NewPostgres(t *testing.T, models ...any) *gorm.DB {
    t.Helper()
    ctx := context.Background()

    pgContainer, err := postgres.RunContainer(ctx,
        testcontainers.WithImage("postgres:16-alpine"),
        postgres.WithDatabase("testdb"),
        postgres.WithUsername("testuser"),
        postgres.WithPassword("testpass"),
        testcontainers.WithWaitStrategy(
            wait.ForLog("database system is ready to accept connections").
                WithOccurrence(2).
                WithStartupTimeout(30*time.Second),
        ),
    )
    if err != nil {
        t.Fatalf("testdb.NewPostgres: start container: %v", err)
    }
    t.Cleanup(func() {
        if err := pgContainer.Terminate(ctx); err != nil {
            t.Logf("testdb.NewPostgres: terminate container: %v", err)
        }
    })

    host, err := pgContainer.Host(ctx)
    if err != nil {
        t.Fatalf("testdb.NewPostgres: get host: %v", err)
    }
    port, err := pgContainer.MappedPort(ctx, "5432")
    if err != nil {
        t.Fatalf("testdb.NewPostgres: get mapped port: %v", err)
    }
    dsn := fmt.Sprintf("host=%s port=%s user=testuser password=testpass dbname=testdb sslmode=disable",
        host, port.Port())

    db, err := gorm.Open(pgdriver.Open(dsn), &gorm.Config{})
    if err != nil {
        t.Fatalf("testdb.NewPostgres: gorm.Open: %v", err)
    }

    if len(models) > 0 {
        if err := db.AutoMigrate(models...); err != nil {
            t.Fatalf("testdb.NewPostgres: AutoMigrate: %v", err)
        }
    }

    return db
}

// Truncate clears all rows from the given tables.
// Call between tests to avoid state leaking across test cases.
//
// SAFETY: table names must be compile-time constants from your codebase.
// Never pass user-supplied input here — PostgreSQL does not support
// parameterized DDL statements, so string concatenation is unavoidable.
// The caller is responsible for ensuring only known, hardcoded table names
// are passed (e.g., Truncate(t, db, "users", "orders")).
func Truncate(t *testing.T, db *gorm.DB, tables ...string) {
    t.Helper()
    for _, table := range tables {
        if err := db.Exec("TRUNCATE TABLE " + table + " RESTART IDENTITY CASCADE").Error; err != nil {
            t.Fatalf("testdb.Truncate(%q): %v", table, err)
        }
    }
}
```

---

## Cleanup Between Tests

Each test must start with a clean database state. Two patterns:

### Pattern 1: Truncate in t.Cleanup (recommended)

```go
//go:build integration

func TestRepository_Create(t *testing.T) {
    // Truncate users table after this test completes (runs even on failure)
    t.Cleanup(func() {
        testDB.Exec("TRUNCATE TABLE users RESTART IDENTITY CASCADE")
    })

    repo := repository.NewUserRepository(testDB)
    user := &domain.User{
        ID:    "u1",
        Name:  "Alice",
        Email: "alice@example.com",
        Role:  "user",
    }

    err := repo.Create(context.Background(), user)
    if err != nil {
        t.Fatalf("Create: %v", err)
    }
}
```

### Pattern 2: Transaction rollback per test

Wrap each test in a transaction that is always rolled back. Fast, but doesn't test transaction commit behavior.

```go
//go:build integration

func withTx(t *testing.T, db *gorm.DB, fn func(tx *gorm.DB)) {
    t.Helper()
    tx := db.Begin()
    if tx.Error != nil {
        t.Fatalf("begin tx: %v", tx.Error)
    }
    t.Cleanup(func() { tx.Rollback() })
    fn(tx)
}

func TestRepository_GetByID_InTx(t *testing.T) {
    withTx(t, testDB, func(tx *gorm.DB) {
        repo := repository.NewUserRepository(tx)
        // All writes are rolled back after the test
        _ = repo.Create(context.Background(), &domain.User{
            ID: "tmp", Name: "Temp", Email: "tmp@example.com", Role: "user",
        })
        got, err := repo.GetByID(context.Background(), "tmp")
        if err != nil {
            t.Fatalf("GetByID: %v", err)
        }
        if got.Name != "Temp" {
            t.Errorf("want 'Temp', got %q", got.Name)
        }
    })
}
```

---

## Repository Integration Tests

Test actual SQL behavior: constraints, not-found semantics, list filtering, and pagination.

```go
//go:build integration

// internal/repository/user_repository_integration_test.go
package repository_test

import (
    "context"
    "errors"
    "testing"

    "myapp/internal/domain"
    "myapp/internal/repository"
)

func TestUserRepository_Create_AndGetByID(t *testing.T) {
    t.Cleanup(func() {
        testDB.Exec("TRUNCATE TABLE users RESTART IDENTITY CASCADE")
    })

    repo := repository.NewUserRepository(testDB)
    ctx := context.Background()

    user := &domain.User{
        ID:    "int-test-id",
        Name:  "Alice",
        Email: "alice@example.com",
        Role:  "user",
    }

    if err := repo.Create(ctx, user); err != nil {
        t.Fatalf("Create: %v", err)
    }

    got, err := repo.GetByID(ctx, user.ID)
    if err != nil {
        t.Fatalf("GetByID: %v", err)
    }
    if got.Email != user.Email {
        t.Errorf("want email %q, got %q", user.Email, got.Email)
    }
}

func TestUserRepository_GetByID_NotFound(t *testing.T) {
    repo := repository.NewUserRepository(testDB)

    _, err := repo.GetByID(context.Background(), "does-not-exist")
    if !errors.Is(err, domain.ErrNotFound) {
        t.Errorf("want ErrNotFound, got %v", err)
    }
}

func TestUserRepository_Create_DuplicateEmail_ReturnsConflict(t *testing.T) {
    t.Cleanup(func() {
        testDB.Exec("TRUNCATE TABLE users RESTART IDENTITY CASCADE")
    })

    repo := repository.NewUserRepository(testDB)
    ctx := context.Background()

    first := &domain.User{ID: "u1", Name: "Alice", Email: "alice@example.com", Role: "user"}
    if err := repo.Create(ctx, first); err != nil {
        t.Fatalf("Create first: %v", err)
    }

    duplicate := &domain.User{ID: "u2", Name: "Alice2", Email: "alice@example.com", Role: "user"}
    err := repo.Create(ctx, duplicate)
    if !errors.Is(err, domain.ErrConflict) {
        t.Errorf("want ErrConflict on duplicate email, got %v", err)
    }
}

func TestUserRepository_List_FiltersByRole(t *testing.T) {
    t.Cleanup(func() {
        testDB.Exec("TRUNCATE TABLE users RESTART IDENTITY CASCADE")
    })

    repo := repository.NewUserRepository(testDB)
    ctx := context.Background()

    _ = repo.Create(ctx, &domain.User{ID: "u1", Name: "Admin", Email: "admin@example.com", Role: "admin"})
    _ = repo.Create(ctx, &domain.User{ID: "u2", Name: "User1", Email: "user1@example.com", Role: "user"})
    _ = repo.Create(ctx, &domain.User{ID: "u3", Name: "User2", Email: "user2@example.com", Role: "user"})

    users, total, err := repo.List(ctx, domain.ListOptions{Page: 1, Limit: 10, Role: "user"})
    if err != nil {
        t.Fatalf("List: %v", err)
    }
    if total != 2 {
        t.Errorf("want 2 users with role 'user', got %d", total)
    }
    if len(users) != 2 {
        t.Errorf("want 2 returned users, got %d", len(users))
    }
}

func TestUserRepository_Delete_SoftDelete(t *testing.T) {
    t.Cleanup(func() {
        testDB.Exec("TRUNCATE TABLE users RESTART IDENTITY CASCADE")
    })

    repo := repository.NewUserRepository(testDB)
    ctx := context.Background()

    user := &domain.User{ID: "del-me", Name: "Alice", Email: "alice@example.com", Role: "user"}
    if err := repo.Create(ctx, user); err != nil {
        t.Fatalf("Create: %v", err)
    }

    if err := repo.Delete(ctx, user.ID); err != nil {
        t.Fatalf("Delete: %v", err)
    }

    _, err := repo.GetByID(ctx, user.ID)
    if !errors.Is(err, domain.ErrNotFound) {
        t.Errorf("want ErrNotFound after delete, got %v", err)
    }
}
```

---

## API Integration Tests

Test the full stack — real router with real database — by calling `router.ServeHTTP`. No network socket needed.

```go
//go:build integration

// internal/handler/user_handler_integration_test.go
package handler_test

import (
    "encoding/json"
    "log/slog"
    "net/http"
    "net/http/httptest"
    "strings"
    "testing"

    "github.com/gin-gonic/gin"
    "myapp/internal/domain"
    "myapp/internal/handler"
    "myapp/internal/repository"
    "myapp/internal/service"
    "myapp/internal/testutil/testdb"
)

func init() { gin.SetMode(gin.TestMode) }

func buildIntegrationRouter(t *testing.T) (*gin.Engine, func()) {
    t.Helper()
    db := testdb.NewPostgres(t, &repository.UserModel{})
    repo := repository.NewUserRepository(db)
    svc := service.NewUserService(repo, slog.Default())
    h := handler.NewUserHandler(svc, slog.Default())

    r := gin.New()
    r.POST("/users", h.Create)
    r.GET("/users/:id", h.GetByID)

    cleanup := func() {
        db.Exec("TRUNCATE TABLE users RESTART IDENTITY CASCADE")
    }
    return r, cleanup
}

func TestUserAPI_CreateAndRetrieve(t *testing.T) {
    router, cleanup := buildIntegrationRouter(t)
    t.Cleanup(cleanup)

    // Step 1: Create user
    body := `{"name":"Alice","email":"alice@example.com","password":"secret1234"}`
    req := httptest.NewRequest(http.MethodPost, "/users", strings.NewReader(body))
    req.Header.Set("Content-Type", "application/json")
    w := httptest.NewRecorder()
    router.ServeHTTP(w, req)

    if w.Code != http.StatusCreated {
        t.Fatalf("Create: want 201, got %d; body: %s", w.Code, w.Body)
    }

    var created domain.User
    if err := json.Unmarshal(w.Body.Bytes(), &created); err != nil {
        t.Fatalf("parse create response: %v", err)
    }
    if created.ID == "" {
        t.Fatal("expected non-empty ID in response")
    }

    // Step 2: Retrieve the same user
    req2 := httptest.NewRequest(http.MethodGet, "/users/"+created.ID, nil)
    w2 := httptest.NewRecorder()
    router.ServeHTTP(w2, req2)

    if w2.Code != http.StatusOK {
        t.Fatalf("GetByID: want 200, got %d; body: %s", w2.Code, w2.Body)
    }

    var fetched domain.User
    if err := json.Unmarshal(w2.Body.Bytes(), &fetched); err != nil {
        t.Fatalf("parse get response: %v", err)
    }
    if fetched.Email != "alice@example.com" {
        t.Errorf("want email 'alice@example.com', got %q", fetched.Email)
    }
}
```

---

## Testing Migrations

Verify migrations apply cleanly and the schema matches the expected structure.

```go
//go:build integration

// internal/repository/migrations_integration_test.go
package repository_test

import (
    "context"
    "fmt"
    "testing"
    "time"

    "github.com/golang-migrate/migrate/v4"
    migratepg "github.com/golang-migrate/migrate/v4/database/postgres"
    _ "github.com/golang-migrate/migrate/v4/source/file"
    "github.com/testcontainers/testcontainers-go"
    tcpostgres "github.com/testcontainers/testcontainers-go/modules/postgres"
    "github.com/testcontainers/testcontainers-go/wait"
    "gorm.io/driver/postgres"
    "gorm.io/gorm"
)

func TestMigrations_UpAndDown(t *testing.T) {
    ctx := context.Background()

    pgc, err := tcpostgres.RunContainer(ctx,
        testcontainers.WithImage("postgres:16-alpine"),
        tcpostgres.WithDatabase("migrate_test"),
        tcpostgres.WithUsername("u"),
        tcpostgres.WithPassword("p"),
        testcontainers.WithWaitStrategy(
            wait.ForLog("database system is ready to accept connections").
                WithOccurrence(2).
                WithStartupTimeout(20*time.Second),
        ),
    )
    if err != nil {
        t.Fatalf("start container: %v", err)
    }
    t.Cleanup(func() { pgc.Terminate(ctx) })

    host, err := pgc.Host(ctx)
    if err != nil {
        t.Fatalf("get container host: %v", err)
    }
    port, err := pgc.MappedPort(ctx, "5432")
    if err != nil {
        t.Fatalf("get mapped port: %v", err)
    }
    dsn := fmt.Sprintf("host=%s port=%s user=u password=p dbname=migrate_test sslmode=disable",
        host, port.Port())

    db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{})
    if err != nil {
        t.Fatalf("gorm.Open: %v", err)
    }
    sqlDB, _ := db.DB()

    driver, err := migratepg.WithInstance(sqlDB, &migratepg.Config{})
    if err != nil {
        t.Fatalf("migrate driver: %v", err)
    }

    m, err := migrate.NewWithDatabaseInstance("file://../../migrations", "postgres", driver)
    if err != nil {
        t.Fatalf("migrate.New: %v", err)
    }

    // Apply all migrations
    if err := m.Up(); err != nil && err != migrate.ErrNoChange {
        t.Fatalf("migrate Up: %v", err)
    }

    // Verify expected tables exist
    var tableName string
    row := sqlDB.QueryRow("SELECT table_name FROM information_schema.tables WHERE table_schema='public' AND table_name='users'")
    if err := row.Scan(&tableName); err != nil {
        t.Fatalf("users table not found after migration: %v", err)
    }

    // Roll back all migrations
    if err := m.Down(); err != nil && err != migrate.ErrNoChange {
        t.Fatalf("migrate Down: %v", err)
    }
}
```

---

## Fixture Loading

Load SQL fixture files to populate a known database state before tests.

```go
//go:build integration

// internal/testutil/testdb/fixtures.go
package testdb

import (
    "os"
    "testing"

    "gorm.io/gorm"
)

// LoadFixture executes SQL from a fixture file against the given db.
// Fixture files live in internal/testutil/fixtures/*.sql
func LoadFixture(t *testing.T, db *gorm.DB, path string) {
    t.Helper()

    sql, err := os.ReadFile(path)
    if err != nil {
        t.Fatalf("LoadFixture: read %q: %v", path, err)
    }

    if err := db.Exec(string(sql)).Error; err != nil {
        t.Fatalf("LoadFixture: exec %q: %v", path, err)
    }
}
```

Example fixture file `internal/testutil/fixtures/users.sql`:

```sql
INSERT INTO users (id, name, email, role, created_at, updated_at)
VALUES
    ('seed-user-1', 'Alice',  'alice@example.com', 'user',  NOW(), NOW()),
    ('seed-user-2', 'Bob',    'bob@example.com',   'user',  NOW(), NOW()),
    ('seed-admin',  'Admin',  'admin@example.com', 'admin', NOW(), NOW());
```

Usage in a test:

```go
//go:build integration

func TestUserRepository_List_WithFixtures(t *testing.T) {
    t.Cleanup(func() {
        testDB.Exec("TRUNCATE TABLE users RESTART IDENTITY CASCADE")
    })

    testdb.LoadFixture(t, testDB, "../../testutil/fixtures/users.sql")

    repo := repository.NewUserRepository(testDB)
    users, total, err := repo.List(context.Background(), domain.ListOptions{Page: 1, Limit: 10})
    if err != nil {
        t.Fatalf("List: %v", err)
    }
    if total != 3 {
        t.Errorf("want 3 users from fixture, got %d", total)
    }
    _ = users
}
```
</file>

<file path="skills/golang-gin-testing/metadata.json">
{
  "version": "1.0.3",
  "organization": "henriqueatila",
  "date": "March 2026",
  "abstract": "Testing patterns for Go Gin REST APIs — unit tests, table-driven tests, benchmarks, fuzz tests, golden file snapshots, testcontainers integration, e2e flows, coverage thresholds, and race detection.",
  "references": [
    "https://gin-gonic.com/docs/",
    "https://pkg.go.dev/net/http/httptest",
    "https://golang.testcontainers.org/",
    "https://context7.com/websites/gin-gonic_en"
  ]
}
</file>

<file path="skills/golang-gin-api/references/error-handling.md">
# Error Handling Reference

This file covers the full domain error system for Gin APIs: AppError struct, sentinel errors, service error mapping, validation error formatting, panic recovery, and consistent JSON response format. Use this when implementing error responses beyond the basics shown in gin-api/SKILL.md.

## Table of Contents

1. [AppError Struct](#apperror-struct)
2. [Sentinel Errors](#sentinel-errors)
3. [handleServiceError Function](#handleserviceerror-function)
4. [Error Wrapping & Unwrapping](#error-wrapping--unwrapping)
5. [Validation Error Formatting](#validation-error-formatting)
6. [Panic Recovery Middleware](#panic-recovery-middleware)
7. [Consistent JSON Error Format](#consistent-json-error-format)

---

## AppError Struct

`AppError` carries an HTTP status code alongside the message, so the transport layer never decides status codes — the domain does.

> **Note:** This `AppError` is a simplified version for this skill's examples. For the canonical domain error pattern with `Detail` field and 5xx guard, see the **golang-gin-clean-arch** skill.

```go
// internal/domain/errors.go
package domain

import "errors"

// AppError is a domain error that maps directly to an HTTP response.
// Services return *AppError; handlers use handleServiceError to render it.
type AppError struct {
    Code    int    // HTTP status code
    Message string // User-facing message (safe to expose)
    Err     error  // Wrapped cause (for logging, not exposed to client)
}

func (e *AppError) Error() string { return e.Message }
func (e *AppError) Unwrap() error  { return e.Err }
func (e AppError) Is(target error) bool {
    switch t := target.(type) {
    case *AppError:
        return e.Code == t.Code
    case AppError:
        return e.Code == t.Code
    }
    return false
}

// New wraps an underlying cause with a domain error.
func (e *AppError) New(cause error) *AppError {
    return &AppError{Code: e.Code, Message: e.Message, Err: cause}
}
```

**Why:** Keeping HTTP status codes in the domain layer means handler code stays thin. The service returns `domain.ErrNotFound.New(err)` and the handler maps it without knowing the HTTP status.

---

## Sentinel Errors

Define one sentinel per failure mode. Services return these (wrapped with cause if needed).

```go
// internal/domain/errors.go (continued)
package domain

var (
    // ErrNotFound is returned when a requested resource does not exist.
    ErrNotFound = &AppError{Code: 404, Message: "resource not found"}

    // ErrUnauthorized is returned when authentication is missing or invalid.
    ErrUnauthorized = &AppError{Code: 401, Message: "unauthorized"}

    // ErrForbidden is returned when the user lacks permission for the action.
    ErrForbidden = &AppError{Code: 403, Message: "forbidden"}

    // ErrConflict is returned when a unique constraint would be violated.
    ErrConflict = &AppError{Code: 409, Message: "resource already exists"}

    // ErrValidation is returned when business rule validation fails (not binding).
    ErrValidation = &AppError{Code: 422, Message: "validation failed"}

    // ErrInternal is returned for unexpected errors. Do NOT expose the cause.
    ErrInternal = &AppError{Code: 500, Message: "internal server error"}
)
```

Usage in a service:

```go
// internal/service/user_service.go
func (s *userService) GetByID(ctx context.Context, id string) (*domain.User, error) {
    user, err := s.repo.GetByID(ctx, id)
    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, domain.ErrNotFound.New(err)
        }
        return nil, domain.ErrInternal.New(err)
    }
    return user, nil
}

func (s *userService) Create(ctx context.Context, req domain.CreateUserRequest) (*domain.User, error) {
    existing, err := s.repo.GetByEmail(ctx, req.Email)
    if err != nil && !errors.Is(err, sql.ErrNoRows) {
        return nil, domain.ErrInternal.New(err)
    }
    if existing != nil {
        return nil, domain.ErrConflict.New(fmt.Errorf("email %s already registered", req.Email))
    }
    // ...
}
```

---

## handleServiceError Function

Place this in the handler package — it is the single translation point between domain errors and HTTP responses.

```go
// internal/handler/errors.go
package handler

import (
    "errors"
    "log/slog"
    "net/http"

    "github.com/gin-gonic/gin"
    "myapp/internal/domain"
)

// handleServiceError maps a domain error to a JSON HTTP response.
// All unrecognised errors become 500 — the cause is logged, NOT sent to client.
func handleServiceError(c *gin.Context, err error, logger *slog.Logger) {
    var appErr *domain.AppError
    if errors.As(err, &appErr) {
        if appErr.Code >= 500 {
            logger.ErrorContext(c.Request.Context(), "service error",
                "error", appErr.Unwrap(),
                "path", c.FullPath(),
            )
        }
        c.JSON(appErr.Code, gin.H{"error": appErr.Message})
        return
    }

    // Unexpected error — log it, return generic message
    logger.ErrorContext(c.Request.Context(), "unexpected error",
        "error", err,
        "path", c.FullPath(),
    )
    c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
}
```

Handler usage:

> **Security note:** In production, never expose raw `err.Error()` to clients. Return generic messages and log the error server-side. See **golang-gin-clean-arch** error handling patterns.

```go
func (h *UserHandler) GetByID(c *gin.Context) {
    type uriParams struct {
        ID string `uri:"id" binding:"required"`
    }
    var p uriParams
    if err := c.ShouldBindURI(&p); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    user, err := h.svc.GetByID(c.Request.Context(), p.ID)
    if err != nil {
        handleServiceError(c, err, h.logger)
        return
    }

    c.JSON(http.StatusOK, user)
}
```

---

## Error Wrapping & Unwrapping

Use `fmt.Errorf` with `%w` to wrap errors while preserving the chain. Use `errors.As` (not type assertion) to unwrap.

```go
// Wrapping — preserves the chain
return nil, fmt.Errorf("userService.Create: %w", domain.ErrConflict.New(err))

// Unwrapping — errors.As walks the chain
var appErr *domain.AppError
if errors.As(err, &appErr) {
    // appErr is the first *AppError in the chain
}

// errors.Is for sentinel comparison (requires Unwrap() on AppError)
if errors.Is(err, domain.ErrNotFound) {
    // true if ErrNotFound appears anywhere in the chain
}
```

**Note:** `errors.Is` compares by pointer identity on the sentinel by default. Wrapping with `.New(cause)` creates a new `*AppError`, which would break `errors.Is` comparison — but `AppError` implements `Is()` (matching by `Code`), so `errors.Is(err, domain.ErrNotFound)` works correctly through the chain. Use `errors.As` when you need access to the full `AppError` value.

---

## Validation Error Formatting

Gin binding errors (`ShouldBindJSON` failure) return raw `validator/v10` error messages. Format them into field-level errors for API consumers.

```go
// internal/handler/validation.go
package handler

import (
    "errors"
    "fmt"
    "net/http"

    "github.com/gin-gonic/gin"
    "github.com/go-playground/validator/v10"
)

// FieldError represents a single field validation failure.
type FieldError struct {
    Field   string `json:"field"`
    Message string `json:"message"`
}

// validationErrors formats validator.ValidationErrors into field-level messages.
func validationErrors(err error) []FieldError {
    var ve validator.ValidationErrors
    if !errors.As(err, &ve) {
        return []FieldError{{Field: "request", Message: err.Error()}}
    }

    out := make([]FieldError, 0, len(ve))
    for _, fe := range ve {
        out = append(out, FieldError{
            Field:   fe.Field(),
            Message: fieldMessage(fe),
        })
    }
    return out
}

func fieldMessage(fe validator.FieldError) string {
    switch fe.Tag() {
    case "required":
        return "field is required"
    case "email":
        return "must be a valid email address"
    case "min":
        return fmt.Sprintf("must be at least %s characters", fe.Param())
    case "max":
        return fmt.Sprintf("must be at most %s characters", fe.Param())
    case "oneof":
        return fmt.Sprintf("must be one of: %s", fe.Param())
    default:
        return fmt.Sprintf("failed validation: %s", fe.Tag())
    }
}

// Usage in handler:
func (h *UserHandler) Create(c *gin.Context) {
    var req domain.CreateUserRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{
            "error":  "validation failed",
            "fields": validationErrors(err),
        })
        return
    }
    // ...
}
```

Response body:
```json
{
  "error": "validation failed",
  "fields": [
    {"field": "Email", "message": "must be a valid email address"},
    {"field": "Password", "message": "must be at least 8 characters"}
  ]
}
```

---

## Panic Recovery Middleware

Use custom recovery to return JSON instead of Gin's default plain-text 500.

```go
// pkg/middleware/recovery.go
package middleware

import (
    "log/slog"
    "net/http"

    "github.com/gin-gonic/gin"
)

// Recovery returns a Gin middleware that recovers from panics and logs them.
// Register with r.Use(middleware.Recovery(logger)) instead of gin.Recovery().
func Recovery(logger *slog.Logger) gin.HandlerFunc {
    return func(c *gin.Context) {
        defer func() {
            if rec := recover(); rec != nil {
                logger.ErrorContext(c.Request.Context(), "panic recovered",
                    "panic", rec,
                    "path", c.FullPath(),
                    "method", c.Request.Method,
                )
                c.AbortWithStatusJSON(http.StatusInternalServerError, gin.H{
                    "error": "internal server error",
                })
            }
        }()
        c.Next()
    }
}
```

Register in main.go:

```go
r := gin.New()
r.Use(middleware.Logger(logger))
r.Use(middleware.Recovery(logger))
```

---

## Consistent JSON Error Format

All error responses must follow the same shape so clients can parse them predictably.

```go
// ErrorResponse is the canonical error body shape.
// Use gin.H for simple errors, this type when you need field-level errors.
type ErrorResponse struct {
    Error  string       `json:"error"`
    Fields []FieldError `json:"fields,omitempty"` // only for validation errors
}
```

Examples:

```json
// 400 binding error
{"error": "invalid JSON body"}

// 400 validation error (with fields)
{"error": "validation failed", "fields": [{"field": "Email", "message": "must be a valid email address"}]}

// 401 unauthorized
{"error": "unauthorized"}

// 404 not found
{"error": "resource not found"}

// 409 conflict
{"error": "resource already exists"}

// 500 internal
{"error": "internal server error"}
```

**Critical:** Never include internal error messages, stack traces, or database errors in 5xx responses. Log internally, return generic message to client.
</file>

<file path="skills/golang-gin-api/references/middleware.md">
# Middleware Reference

This file covers Gin middleware patterns: signature and chain execution order, CORS configuration, request logging with log/slog, rate limiting, request ID injection, recovery with custom JSON response, timeout middleware, and a custom middleware template.

## Table of Contents

1. [Middleware Signature & Chain Execution](#middleware-signature--chain-execution)
2. [CORS Configuration](#cors-configuration)
3. [Security Headers Middleware](#security-headers-middleware)
4. [Request Logging with log/slog](#request-logging-with-logslog)
5. [Rate Limiting](#rate-limiting)
6. [Request ID Middleware](#request-id-middleware)
7. [Recovery Middleware](#recovery-middleware)
8. [Timeout Middleware](#timeout-middleware)
9. [Custom Middleware Template](#custom-middleware-template)

---

## Middleware Signature & Chain Execution

All handlers and middleware share the same signature: `func(*gin.Context)`.

```go
// Execution order with two middleware and one handler:
// Middleware1-before → Middleware2-before → Handler → Middleware2-after → Middleware1-after

func Middleware1(c *gin.Context) {
    // runs BEFORE handler
    c.Next()
    // runs AFTER handler (and all subsequent middleware)
    status := c.Writer.Status()
    _ = status
}

func Middleware2(c *gin.Context) {
    c.Next()
}

func Handler(c *gin.Context) {
    c.JSON(200, gin.H{"ok": true})
}
```

Registration order matters. Register middleware before routes:

```go
r := gin.New()                    // no middleware
r.Use(middleware.RequestID())     // 1st: inject request ID (used by logger)
r.Use(middleware.Logger(logger))  // 2nd: logs with request ID available
r.Use(middleware.Recovery(logger)) // 3rd: catches panics from any handler

// Routes registered after middleware
r.GET("/health", healthHandler)
```

**Why `gin.New()` over `gin.Default()`:** `gin.Default()` adds Logger and Recovery with their default configuration. Use `gin.New()` + explicit middleware so you control format, destination, and behaviour of each middleware — critical for structured logging and custom error responses in production.

---

## CORS Configuration

Use `github.com/gin-contrib/cors` for CORS. Configure per environment.

```go
// pkg/middleware/cors.go
package middleware

import (
    "os"
    "strings"
    "time"

    "github.com/gin-contrib/cors"
    "github.com/gin-gonic/gin"
)

// CORS returns a configured CORS middleware.
// In development, allows all origins. In production, restricts to configured origins.
func CORS() gin.HandlerFunc {
    env := os.Getenv("APP_ENV")

    if env == "development" {
        return cors.New(cors.Config{
            AllowOrigins:     []string{"http://localhost:3000", "http://localhost:5173"},
            AllowMethods:     []string{"GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"},
            AllowHeaders:     []string{"Origin", "Content-Type", "Authorization", "X-Request-ID"},
            AllowCredentials: true,
            MaxAge:           12 * time.Hour,
        })
    }

    // Production: explicit origin allowlist
    origins := os.Getenv("ALLOWED_ORIGINS") // e.g. "https://app.example.com,https://admin.example.com"
    allowedOrigins := []string{}
    if origins != "" {
        for _, o := range strings.Split(origins, ",") {
            allowedOrigins = append(allowedOrigins, strings.TrimSpace(o))
        }
    }

    return cors.New(cors.Config{
        AllowOrigins:     allowedOrigins,
        AllowMethods:     []string{"GET", "POST", "PUT", "PATCH", "DELETE"},
        AllowHeaders:     []string{"Origin", "Content-Type", "Authorization", "X-Request-ID"},
        ExposeHeaders:    []string{"X-Request-ID"},
        AllowCredentials: true,
        MaxAge:           1 * time.Hour,
    })
}
```

Register before routes:

```go
r := gin.New()
r.Use(middleware.CORS())
r.Use(middleware.RequestID())
r.Use(middleware.Logger(logger))
r.Use(middleware.Recovery(logger))
```

**Warning:** `AllowAllOrigins: true` with `AllowCredentials: true` is rejected by browsers. Use one or the other, never both in production.

---

## Security Headers Middleware

Set HTTP security headers on every response. Register early in the chain so they are always applied, even when a later middleware aborts the request.

```go
// pkg/middleware/security_headers.go
package middleware

import "github.com/gin-gonic/gin"

// SecurityHeaders sets recommended HTTP security headers on every response.
// Register before route-specific middleware so headers are always present.
func SecurityHeaders() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Header("X-Content-Type-Options", "nosniff")
        c.Header("X-Frame-Options", "DENY")
        c.Header("X-XSS-Protection", "0") // disabled — CSP supersedes it; non-zero triggers bugs in old browsers
        c.Header("Referrer-Policy", "strict-origin-when-cross-origin")
        c.Header("Content-Security-Policy", "default-src 'self'")
        c.Header("Strict-Transport-Security", "max-age=63072000; includeSubDomains; preload")
        c.Header("Permissions-Policy", "geolocation=(), microphone=(), camera=()")
        c.Next()
    }
}
```

Register in main.go after CORS and before route groups:

```go
r := gin.New()
r.Use(middleware.CORS())
r.Use(middleware.SecurityHeaders())
r.Use(middleware.RequestID())
r.Use(middleware.Logger(logger))
r.Use(middleware.Recovery(logger))
```

**Header notes:**
- `X-XSS-Protection: 0` — disables the legacy XSS auditor; `Content-Security-Policy` is the correct control.
- `Strict-Transport-Security` — only effective over HTTPS; set `preload` after submitting your domain to the HSTS preload list.
- `Content-Security-Policy` — `default-src 'self'` is a safe starting point; tighten per route group if the API serves HTML or allows CDN assets.

---

## Request Logging with log/slog

Log each request with structured fields: method, path, status, duration, request ID.

```go
// pkg/middleware/logger.go
package middleware

import (
    "log/slog"
    "time"

    "github.com/gin-gonic/gin"
)

// Logger returns a Gin middleware that logs each request using log/slog.
// Requires RequestID middleware to run first for request ID injection.
func Logger(logger *slog.Logger) gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()
        path := c.Request.URL.Path
        query := c.Request.URL.RawQuery

        c.Next() // process request

        duration := time.Since(start)
        status := c.Writer.Status()

        attrs := []slog.Attr{
            slog.String("method", c.Request.Method),
            slog.String("path", path),
            slog.Int("status", status),
            slog.Duration("duration", duration),
            slog.String("ip", c.ClientIP()),
            slog.String("request_id", c.GetString("request_id")),
        }
        if query != "" {
            attrs = append(attrs, slog.String("query", query))
        }
        if len(c.Errors) > 0 {
            attrs = append(attrs, slog.String("errors", c.Errors.String()))
        }

        level := slog.LevelInfo
        if status >= 500 {
            level = slog.LevelError
        } else if status >= 400 {
            level = slog.LevelWarn
        }

        logger.LogAttrs(c.Request.Context(), level, "http request", attrs...)
    }
}
```

Sample output (JSON handler):

```json
{"time":"2026-03-01T10:00:00Z","level":"INFO","msg":"http request","method":"POST","path":"/api/v1/users","status":201,"duration":"2.3ms","ip":"192.168.1.1","request_id":"a1b2c3d4"}
```

---

## Rate Limiting

Basic in-memory rate limiting uses `golang.org/x/time/rate` (token bucket) per client IP:

```go
// Global: 10 req/s, burst of 20
r.Use(middleware.RateLimiter(10, 20))

// Stricter limit on auth endpoints
auth := r.Group("/api/v1/auth")
auth.Use(middleware.RateLimiter(2, 5))
```

For the full implementation and production patterns (Redis distributed limiting, sliding window, per-user/API-key quotas, tiered limits, response headers), see **[rate-limiting.md](rate-limiting.md)**.

---

## Request ID Middleware

Inject a unique request ID per request. Propagate it in the response header and store in context for logging.

```go
// pkg/middleware/request_id.go
package middleware

import (
    "github.com/gin-gonic/gin"
    "github.com/google/uuid"
)

const RequestIDKey = "request_id"
const RequestIDHeader = "X-Request-ID"

// RequestID injects a unique request ID into the context and response headers.
// Reuses the incoming X-Request-ID header if present (for distributed tracing).
func RequestID() gin.HandlerFunc {
    return func(c *gin.Context) {
        id := c.GetHeader(RequestIDHeader)
        if id == "" {
            id = uuid.NewString() // UUIDv4 via google/uuid
        }

        c.Set(RequestIDKey, id)
        c.Header(RequestIDHeader, id)
        c.Next()
    }
}
```

Retrieve in any handler or downstream service:

```go
requestID := c.GetString(middleware.RequestIDKey)

// Pass to slog context for structured logging
logger := slog.With("request_id", requestID)
```

---

## Recovery Middleware

Return JSON on panic instead of Gin's default plain-text 500.

```go
// pkg/middleware/recovery.go
package middleware

import (
    "log/slog"
    "net/http"
    "runtime/debug"

    "github.com/gin-gonic/gin"
)

// Recovery returns middleware that recovers from panics, logs the stack trace,
// and returns a JSON 500 response. Use instead of gin.Recovery().
func Recovery(logger *slog.Logger) gin.HandlerFunc {
    return func(c *gin.Context) {
        defer func() {
            if rec := recover(); rec != nil {
                logger.ErrorContext(c.Request.Context(), "panic recovered",
                    "panic", rec,
                    "stack", string(debug.Stack()),
                    "method", c.Request.Method,
                    "path", c.FullPath(),
                    "request_id", c.GetString(RequestIDKey),
                )
                c.AbortWithStatusJSON(http.StatusInternalServerError, gin.H{
                    "error": "internal server error",
                })
            }
        }()
        c.Next()
    }
}
```

---

## Timeout Middleware

Cancel the request context after a deadline. Handlers must respect `c.Request.Context()` for this to work.

```go
// pkg/middleware/timeout.go
package middleware

import (
    "context"
    "net/http"
    "time"

    "github.com/gin-gonic/gin"
)

// Timeout returns middleware that cancels the request context after d.
// Handlers must pass c.Request.Context() to all blocking calls for cancellation to work.
//
// The context deadline propagates to every downstream blocking call that respects it
// (database queries, HTTP clients, gRPC calls). This does NOT forcibly interrupt
// handlers that ignore context — it is cooperative cancellation only.
func Timeout(d time.Duration) gin.HandlerFunc {
    return func(c *gin.Context) {
        ctx, cancel := context.WithTimeout(c.Request.Context(), d)
        defer cancel()

        // Replace request context with the deadline-bearing one.
        // c.Next() is called on the main goroutine — no data race on c.Writer.
        c.Request = c.Request.WithContext(ctx)
        c.Next()

        // If the context expired before the handler responded, return 504.
        // This handles the case where a handler finished after the deadline
        // but before writing a response (e.g., returned early on ctx.Err()).
        if ctx.Err() == context.DeadlineExceeded && !c.Writer.Written() {
            c.AbortWithStatusJSON(http.StatusGatewayTimeout, gin.H{
                "error": "request timeout",
            })
        }
    }
}
```

Apply per route group, not globally (health check should not time out):

```go
api := r.Group("/api/v1")
api.Use(middleware.Timeout(30 * time.Second))
```

**How this works:** The deadline is embedded in `c.Request.Context()`. Any handler that passes `c.Request.Context()` to a database query, HTTP client, or gRPC call will have that call cancelled when the deadline fires. `c.Next()` runs synchronously on the middleware goroutine — no concurrent writes to `c.Writer`.

**Important:** This is cooperative cancellation. Handlers that do not check `ctx.Done()` or pass context to blocking calls will not be interrupted. For CPU-bound handlers, add an explicit `ctx.Err()` check in tight loops.

**Do NOT call `c.Next()` in a goroutine** for timeout middleware. Gin's `ResponseWriter` is not thread-safe. Calling `c.Next()` in a goroutine without synchronization creates a data race when both the goroutine and the timeout branch attempt to write to `c.Writer` concurrently.

---

## Custom Middleware Template

Use this as a starting point for any new middleware.

```go
// pkg/middleware/my_middleware.go
package middleware

import (
    "log/slog"
    "net/http"

    "github.com/gin-gonic/gin"
)

// MyMiddleware does X. It requires Y to run before it.
// Apply at: r.Use() for global, group.Use() for group-scoped, or inline for per-route.
func MyMiddleware(logger *slog.Logger) gin.HandlerFunc {
    // One-time setup (runs at registration, not per request)
    // e.g., compile regexps, open connections

    return func(c *gin.Context) {
        // --- PRE-HANDLER ---
        // Read request data: c.GetHeader(), c.Query(), c.Param()
        // Short-circuit with c.AbortWithStatusJSON() if validation fails

        val := c.GetHeader("X-My-Header")
        if val == "" {
            c.AbortWithStatusJSON(http.StatusBadRequest, gin.H{
                "error": "X-My-Header is required",
            })
            return
        }

        // Store data for downstream handlers
        c.Set("my_value", val)

        // --- CALL NEXT ---
        c.Next()

        // --- POST-HANDLER ---
        // Read response data: c.Writer.Status(), c.Writer.Size()
        // Add response headers, log, metrics
        status := c.Writer.Status()
        logger.InfoContext(c.Request.Context(), "my middleware post",
            "status", status,
            "value", val,
        )
    }
}
```
</file>

<file path="skills/golang-gin-api/references/rate-limiting.md">
# Rate Limiting Reference

This file covers production rate limiting strategies for Gin APIs: in-memory token bucket (single-node), sliding window counter, Redis-backed distributed limiting (token bucket and sliding window), per-user/API-key quotas, tiered limits, standard response headers, and graceful Redis degradation. See `middleware.md` for a simpler in-memory recap.

## Table of Contents

1. [Algorithm Overview](#algorithm-overview)
2. [In-Memory Token Bucket](#in-memory-token-bucket)
3. [Sliding Window Counter](#sliding-window-counter)
4. [Redis Token Bucket (Lua)](#redis-token-bucket-lua)
5. [Redis Sliding Window](#redis-sliding-window)
6. [Per-User / API-Key Limiting](#per-user--api-key-limiting)
7. [Tiered Limits](#tiered-limits)
8. [Response Headers](#response-headers)
9. [Graceful Degradation](#graceful-degradation)

---

## Algorithm Overview

| Algorithm | Accuracy | Memory | Distributed | Best For |
|---|---|---|---|---|
| Fixed window | Low (boundary burst) | O(1) | Yes (INCR) | Simple global caps |
| Token bucket | High | O(clients) | Yes (Lua) | Smooth traffic shaping |
| Sliding window counter | Medium-high | O(1) | Yes (INCR×2) | Accurate without per-request storage |
| Sliding window log | Exact | O(requests) | Yes (ZADD) | High-value APIs, low traffic |

**Token bucket** (via `golang.org/x/time/rate`) is best for single-node deployments — it allows short bursts while enforcing a long-term rate. **Sliding window counter** blends two fixed-window buckets for near-exact counts without storing individual timestamps. **Redis Lua** scripts ensure atomic check-and-decrement across instances.

Use **in-memory** when: single binary, simplicity matters, losing counts on restart is acceptable.
Use **Redis** when: multiple instances, counts must survive restarts, per-user billing accuracy required.

---

## In-Memory Token Bucket

`golang.org/x/time/rate` wraps the token bucket algorithm. This is an improved version of the snippet in `middleware.md` — it accepts a configurable cleanup interval and a done channel for clean shutdown.

```go
// pkg/middleware/rate_limiter_memory.go
package middleware

import (
	"log/slog"
	"net/http"
	"sync"
	"time"

	"github.com/gin-gonic/gin"
	"golang.org/x/time/rate"
)

type clientLimiter struct {
	limiter  *rate.Limiter
	lastSeen time.Time
}

// MemoryRateLimiterConfig configures the in-memory rate limiter.
type MemoryRateLimiterConfig struct {
	// Rate is the number of requests allowed per second.
	Rate rate.Limit
	// Burst is the maximum number of requests allowed in a single burst.
	Burst int
	// CleanupInterval controls how often stale entries are removed.
	CleanupInterval time.Duration
	// TTL is how long a client entry lives without activity before removal.
	TTL time.Duration
}

// MemoryRateLimiter limits requests per client IP using a token bucket.
// It is NOT suitable for multi-instance deployments — use RedisRateLimiter instead.
func MemoryRateLimiter(cfg MemoryRateLimiterConfig, done <-chan struct{}) gin.HandlerFunc {
	if cfg.CleanupInterval == 0 {
		cfg.CleanupInterval = time.Minute
	}
	if cfg.TTL == 0 {
		cfg.TTL = 3 * time.Minute
	}

	var mu sync.Mutex
	limiters := make(map[string]*clientLimiter)

	// Background cleanup — exits when done is closed.
	go func() {
		ticker := time.NewTicker(cfg.CleanupInterval)
		defer ticker.Stop()
		for {
			select {
			case <-ticker.C:
				mu.Lock()
				for ip, cl := range limiters {
					if time.Since(cl.lastSeen) > cfg.TTL {
						delete(limiters, ip)
					}
				}
				mu.Unlock()
			case <-done:
				return
			}
		}
	}()

	getLimiter := func(ip string) *rate.Limiter {
		mu.Lock()
		defer mu.Unlock()
		cl, ok := limiters[ip]
		if !ok {
			cl = &clientLimiter{limiter: rate.NewLimiter(cfg.Rate, cfg.Burst)}
			limiters[ip] = cl
		}
		cl.lastSeen = time.Now()
		return cl.limiter
	}

	return func(c *gin.Context) {
		ip := c.ClientIP()
		lim := getLimiter(ip)
		if !lim.Allow() {
			slog.WarnContext(c.Request.Context(), "rate limit exceeded", "ip", ip)
			c.Header("Retry-After", "1")
			c.AbortWithStatusJSON(http.StatusTooManyRequests, gin.H{
				"error": "rate limit exceeded",
			})
			return
		}
		c.Next()
	}
}
```

**Usage:**

```go
done := make(chan struct{})
defer close(done)

r.Use(middleware.MemoryRateLimiter(middleware.MemoryRateLimiterConfig{
    Rate:  10,
    Burst: 20,
}, done))
```

**Why `done` channel:** The goroutine would leak if the server restarts or the router is rebuilt in tests. Closing `done` stops the ticker cleanly.

---

## Sliding Window Counter

Blends two consecutive fixed-window buckets weighted by elapsed time in the current window. More accurate than a plain fixed window (no boundary burst) without storing per-request timestamps.

```go
// pkg/middleware/rate_limiter_sliding.go
package middleware

import (
	"log/slog"
	"net/http"
	"sync"
	"time"

	"github.com/gin-gonic/gin"
)

type windowBucket struct {
	count    int
	windowAt time.Time // start of the current window
}

// SlidingWindowLimiter limits requests using a sliding window counter.
// windowSize: duration of one window (e.g. time.Minute).
// limit: max requests per window.
func SlidingWindowLimiter(windowSize time.Duration, limit int) gin.HandlerFunc {
	var mu sync.Mutex
	buckets := make(map[string][2]windowBucket) // [prev, curr]

	return func(c *gin.Context) {
		key := c.ClientIP()
		now := time.Now()
		windowStart := now.Truncate(windowSize)

		mu.Lock()
		pair := buckets[key]

		// Advance windows if necessary.
		if pair[1].windowAt != windowStart {
			if pair[1].windowAt.Add(windowSize).Equal(windowStart) {
				// Shift: current → previous.
				pair[0] = pair[1]
			} else {
				// Gap larger than one window — reset both.
				pair[0] = windowBucket{}
			}
			pair[1] = windowBucket{windowAt: windowStart}
		}

		// Weighted count: fraction of previous window that overlaps current.
		elapsed := now.Sub(windowStart)
		overlap := 1.0 - elapsed.Seconds()/windowSize.Seconds()
		count := int(float64(pair[0].count)*overlap) + pair[1].count

		if count >= limit {
			mu.Unlock()
			slog.WarnContext(c.Request.Context(), "sliding window limit exceeded", "ip", key)
			c.Header("Retry-After", windowSize.String())
			c.AbortWithStatusJSON(http.StatusTooManyRequests, gin.H{
				"error": "rate limit exceeded",
			})
			return
		}

		pair[1].count++
		buckets[key] = pair
		mu.Unlock()

		c.Next()
	}
}
```

**Why weighted overlap:** If a client sent 90 requests in the last window and the current window is 30% complete, the effective count is `90 * 0.7 + current`. This prevents the burst that occurs at fixed-window boundaries.

---

## Redis Token Bucket (Lua)

A Lua script executes atomically on the Redis instance — no race between read and write. Uses two keys per client: `tokens` (current count) and `ts` (last refill timestamp).

```go
// pkg/middleware/rate_limiter_redis.go
package middleware

import (
	"context"
	"errors"
	"log/slog"
	"net/http"
	"strconv"
	"time"

	"github.com/gin-gonic/gin"
	"github.com/redis/go-redis/v9"
)

// tokenBucketScript atomically checks and decrements a token bucket stored in Redis.
// KEYS[1]: token count key, KEYS[2]: last-refill timestamp key
// ARGV[1]: capacity, ARGV[2]: refill rate (tokens/sec), ARGV[3]: current unix timestamp (float)
// Returns: remaining tokens after the request (-1 means denied).
var tokenBucketScript = redis.NewScript(`
local tokens_key = KEYS[1]
local ts_key     = KEYS[2]
local capacity   = tonumber(ARGV[1])
local rate       = tonumber(ARGV[2])
local now        = tonumber(ARGV[3])
local ttl        = math.ceil(capacity / rate) + 1

local last_ts = tonumber(redis.call("GET", ts_key))
local tokens

if last_ts == nil then
    tokens = capacity
else
    local elapsed = now - last_ts
    tokens = math.min(capacity, tonumber(redis.call("GET", tokens_key) or 0) + elapsed * rate)
end

if tokens < 1 then
    return -1
end

tokens = tokens - 1
redis.call("SETEX", tokens_key, ttl, tokens)
redis.call("SETEX", ts_key, ttl, now)
return tokens
`)

// RedisTokenBucketConfig configures the Redis-backed token bucket limiter.
type RedisTokenBucketConfig struct {
	Client   *redis.Client
	Capacity int           // maximum tokens (burst size)
	Rate     float64       // tokens refilled per second
	KeyPrefix string       // e.g. "rl:api:"
}

// RedisTokenBucketLimiter limits requests using an atomic token bucket in Redis.
func RedisTokenBucketLimiter(cfg RedisTokenBucketConfig) gin.HandlerFunc {
	return func(c *gin.Context) {
		ctx := c.Request.Context()
		ip := c.ClientIP()
		base := cfg.KeyPrefix + ip
		tokensKey := base + ":tokens"
		tsKey := base + ":ts"
		now := float64(time.Now().UnixNano()) / 1e9

		remaining, err := tokenBucketScript.Run(ctx, cfg.Client,
			[]string{tokensKey, tsKey},
			cfg.Capacity, cfg.Rate, now,
		).Int()

		if err != nil && !errors.Is(err, redis.Nil) {
			// Redis error — log and allow (fail open). See Graceful Degradation section.
			slog.ErrorContext(ctx, "redis rate limiter error", "err", err)
			c.Next()
			return
		}

		c.Header("X-RateLimit-Limit", strconv.Itoa(cfg.Capacity))

		if remaining < 0 {
			c.Header("X-RateLimit-Remaining", "0")
			c.Header("Retry-After", strconv.Itoa(int(1.0/cfg.Rate)+1))
			c.AbortWithStatusJSON(http.StatusTooManyRequests, gin.H{
				"error": "rate limit exceeded",
			})
			return
		}

		c.Header("X-RateLimit-Remaining", strconv.Itoa(remaining))
		c.Next()
	}
}
```

**Why Lua:** `GET` + conditional `SET` in application code has a TOCTOU race under concurrent requests. A Lua script is executed atomically by the Redis server — no other command runs between the read and write.

---

## Redis Sliding Window

Uses a sorted set where each member is a unique request ID and the score is the Unix timestamp (nanoseconds). `ZREMRANGEBYSCORE` prunes old entries; `ZCARD` counts the current window.

```go
// pkg/middleware/rate_limiter_redis_sliding.go
package middleware

import (
	"context"
	"errors"
	"log/slog"
	"net/http"
	"strconv"
	"time"

	"github.com/gin-gonic/gin"
	"github.com/redis/go-redis/v9"
)

// redisSlidingScript atomically records a request and checks the window count.
// KEYS[1]: sorted set key
// ARGV[1]: window size in nanoseconds, ARGV[2]: current timestamp (ns), ARGV[3]: limit, ARGV[4]: unique request ID
// Returns: 0 = allowed, 1 = denied.
var redisSlidingScript = redis.NewScript(`
local key    = KEYS[1]
local window = tonumber(ARGV[1])
local now    = tonumber(ARGV[2])
local limit  = tonumber(ARGV[3])
local req_id = ARGV[4]
local cutoff = now - window

redis.call("ZREMRANGEBYSCORE", key, "-inf", cutoff)
local count = redis.call("ZCARD", key)

if count >= limit then
    return 1
end

redis.call("ZADD", key, now, req_id)
redis.call("PEXPIRE", key, math.ceil(window / 1e6))  -- ns → ms
return 0
`)

// RedisSlidingWindowConfig configures the Redis sliding window limiter.
type RedisSlidingWindowConfig struct {
	Client    *redis.Client
	Window    time.Duration
	Limit     int
	KeyPrefix string
}

// RedisSlidingWindowLimiter limits requests using a Redis sorted-set sliding window.
func RedisSlidingWindowLimiter(cfg RedisSlidingWindowConfig) gin.HandlerFunc {
	windowNs := cfg.Window.Nanoseconds()

	return func(c *gin.Context) {
		ctx := c.Request.Context()
		ip := c.ClientIP()
		key := cfg.KeyPrefix + ip
		now := time.Now().UnixNano()
		// Unique member per request to avoid ZADD deduplication.
		reqID := strconv.FormatInt(now, 36) + ip

		denied, err := redisSlidingScript.Run(ctx, cfg.Client,
			[]string{key},
			windowNs, now, cfg.Limit, reqID,
		).Int()

		if err != nil && !errors.Is(err, redis.Nil) {
			slog.ErrorContext(ctx, "redis sliding window error", "err", err)
			c.Next() // fail open
			return
		}

		c.Header("X-RateLimit-Limit", strconv.Itoa(cfg.Limit))

		if denied == 1 {
			c.Header("X-RateLimit-Remaining", "0")
			c.Header("Retry-After", strconv.FormatFloat(cfg.Window.Seconds(), 'f', 0, 64))
			c.AbortWithStatusJSON(http.StatusTooManyRequests, gin.H{
				"error": "rate limit exceeded",
			})
			return
		}

		c.Next()
	}
}
```

**Memory trade-off:** Each request is a sorted set entry. For 1000 req/min limit × 10 000 clients = ~10M entries. At ~100 bytes/entry this is ~1 GB. Use the sliding window **counter** approach (two INCR keys) if memory is a concern at scale.

---

## Per-User / API-Key Limiting

Replace the IP key with a user ID from JWT claims or an `X-API-Key` header. This survives IP changes (mobile clients) and is harder to spoof than IP.

```go
// pkg/middleware/rate_limiter_key_extractor.go
package middleware

import (
	"strings"

	"github.com/gin-gonic/gin"
	"github.com/golang-jwt/jwt/v5"
)

// ClientKeyFunc extracts a rate-limit key from the request.
// Returning "" falls back to ClientIP.
type ClientKeyFunc func(c *gin.Context) string

// APIKeyExtractor reads the X-API-Key header.
func APIKeyExtractor(c *gin.Context) string {
	if key := c.GetHeader("X-API-Key"); key != "" {
		return "apikey:" + key
	}
	return ""
}

// JWTSubjectExtractor reads the "sub" claim from a Bearer token.
// It does NOT validate the token — pair with AuthRequired middleware first.
func JWTSubjectExtractor(c *gin.Context) string {
	header := c.GetHeader("Authorization")
	if !strings.HasPrefix(header, "Bearer ") {
		return ""
	}
	tokenStr := strings.TrimPrefix(header, "Bearer ")
	// Parse without validation — signature already verified by AuthRequired.
	claims := jwt.MapClaims{}
	parser := jwt.NewParser()
	token, _, err := parser.ParseUnverified(tokenStr, claims)
	if err != nil {
		return ""
	}
	_ = token // ParseUnverified skips validation — token.Valid is always false
	sub, err := claims.GetSubject()
	if err != nil || sub == "" {
		return ""
	}
	return "user:" + sub
}

// WithKeyExtractor wraps any limiter middleware with a custom key extractor.
// keyFn derives the rate-limit key; if it returns "", c.ClientIP() is used.
// limiterFn must accept a key and return gin.HandlerFunc (partial application pattern).
func WithKeyExtractor(keyFn ClientKeyFunc, next func(key string) gin.HandlerFunc) gin.HandlerFunc {
	return func(c *gin.Context) {
		key := keyFn(c)
		if key == "" {
			key = c.ClientIP()
		}
		next(key)(c)
	}
}
```

**Usage with Redis token bucket:**

```go
// main.go
// withKeyExtractor + limiterFn compose into a single gin.HandlerFunc.
// APIKeyExtractor has signature func(*gin.Context) string (ClientKeyFunc),
// not gin.HandlerFunc — it cannot be used with r.Use() directly.
limiterFn := func(key string) gin.HandlerFunc {
    return middleware.RedisTokenBucketLimiter(middleware.RedisTokenBucketConfig{
        Client:    rdb,
        Capacity:  100,
        Rate:      10,
        KeyPrefix: key + ":",
    })
}

api := r.Group("/api/v1")
api.Use(middleware.WithKeyExtractor(middleware.APIKeyExtractor, limiterFn))
```

---

## Tiered Limits

Different limits per user role or subscription plan. Load from config; avoid hardcoding.

```go
// pkg/middleware/rate_limiter_tiered.go
package middleware

import (
	"log/slog"
	"net/http"
	"strconv"
	"time"

	"github.com/gin-gonic/gin"
	"github.com/redis/go-redis/v9"
	"golang.org/x/time/rate"
)

// TierConfig holds rate limit parameters for one tier.
type TierConfig struct {
	Rate  rate.Limit // requests per second (in-memory) or tokens/sec (Redis)
	Burst int        // max burst
}

// DefaultTiers maps role names to limits. Override from env/config.
var DefaultTiers = map[string]TierConfig{
	"anonymous":     {Rate: 2, Burst: 5},
	"authenticated": {Rate: 20, Burst: 40},
	"premium":       {Rate: 100, Burst: 200},
}

// TieredRateLimiter applies per-role limits using the Redis token bucket.
// It reads "role" from the Gin context (set by AuthRequired middleware).
func TieredRateLimiter(rdb *redis.Client, tiers map[string]TierConfig, keyPrefix string) gin.HandlerFunc {
	fallback := tiers["anonymous"]

	return func(c *gin.Context) {
		ctx := c.Request.Context()

		role, _ := c.Get("role") // set by gin-auth AuthRequired
		roleStr, _ := role.(string)
		cfg, ok := tiers[roleStr]
		if !ok {
			cfg = fallback
		}

		sub, _ := c.Get("sub") // user ID from JWT
		subStr, _ := sub.(string)
		if subStr == "" {
			subStr = c.ClientIP()
		}
		base := keyPrefix + roleStr + ":" + subStr

		now := float64(time.Now().UnixNano()) / 1e9
		remaining, err := tokenBucketScript.Run(ctx, rdb,
			[]string{base + ":tokens", base + ":ts"},
			cfg.Burst, float64(cfg.Rate), now,
		).Int()

		if err != nil && err != redis.Nil {
			slog.ErrorContext(ctx, "tiered limiter redis error", "err", err, "role", roleStr)
			c.Next()
			return
		}

		c.Header("X-RateLimit-Limit", strconv.Itoa(cfg.Burst))

		if remaining < 0 {
			c.Header("X-RateLimit-Remaining", "0")
			retryAfter := int(1.0/float64(cfg.Rate)) + 1
			c.Header("Retry-After", strconv.Itoa(retryAfter))
			c.AbortWithStatusJSON(http.StatusTooManyRequests, gin.H{
				"error": "rate limit exceeded",
			})
			return
		}

		c.Header("X-RateLimit-Remaining", strconv.Itoa(remaining))
		c.Next()
	}
}
```

**Loading tiers from environment:**

```go
// config/rate_limits.go
package config

import (
	"os"
	"strconv"
	"strings"

	"golang.org/x/time/rate"
	"myapp/pkg/middleware"
)

func LoadTiers() map[string]middleware.TierConfig {
	tiers := make(map[string]middleware.TierConfig)
	for _, role := range []string{"anonymous", "authenticated", "premium"} {
		rateKey := "RATE_" + strings.ToUpper(role) + "_RPS"
		burstKey := "RATE_" + strings.ToUpper(role) + "_BURST"
		r, err := strconv.ParseFloat(os.Getenv(rateKey), 64)
		if err != nil {
			r = float64(middleware.DefaultTiers[role].Rate)
		}
		b, err := strconv.Atoi(os.Getenv(burstKey))
		if err != nil {
			b = middleware.DefaultTiers[role].Burst
		}
		tiers[role] = middleware.TierConfig{Rate: rate.Limit(r), Burst: b}
	}
	return tiers
}
```

---

## Response Headers

Set these on every response — not only on 429 — so clients can self-throttle before hitting the limit.

| Header | Value | When |
|---|---|---|
| `X-RateLimit-Limit` | Total requests allowed in window | Always |
| `X-RateLimit-Remaining` | Requests remaining this window | Always |
| `X-RateLimit-Reset` | Unix epoch when window resets | Always (where applicable) |
| `Retry-After` | Seconds until client may retry | 429 responses only |

```go
// pkg/middleware/rate_limit_headers.go
package middleware

import (
	"net/http"
	"strconv"
	"time"

	"github.com/gin-gonic/gin"
)

// SetRateLimitHeaders writes standard rate limit headers.
// resetAt: time when the current window expires (zero value omits X-RateLimit-Reset).
func SetRateLimitHeaders(c *gin.Context, limit, remaining int, resetAt time.Time) {
	c.Header("X-RateLimit-Limit", strconv.Itoa(limit))
	c.Header("X-RateLimit-Remaining", strconv.Itoa(remaining))
	if !resetAt.IsZero() {
		c.Header("X-RateLimit-Reset", strconv.FormatInt(resetAt.Unix(), 10))
	}
}

// AbortRateLimited sends a 429 with Retry-After and standard rate limit headers.
func AbortRateLimited(c *gin.Context, limit int, retryAfterSec int) {
	SetRateLimitHeaders(c, limit, 0, time.Now().Add(time.Duration(retryAfterSec)*time.Second))
	c.Header("Retry-After", strconv.Itoa(retryAfterSec))
	c.AbortWithStatusJSON(http.StatusTooManyRequests, gin.H{
		"error":       "rate limit exceeded",
		"retry_after": retryAfterSec,
	})
}
```

**Why always set headers:** Clients and API gateways use `X-RateLimit-Remaining` to implement proactive back-off. If headers only appear on 429, clients have no signal until they are already blocked.

---

## Graceful Degradation

When Redis is unavailable, fall back to an in-memory limiter instead of blocking all requests or panicking.

```go
// pkg/middleware/rate_limiter_fallback.go
package middleware

import (
	"context"
	"log/slog"
	"sync"
	"time"

	"github.com/gin-gonic/gin"
	"github.com/redis/go-redis/v9"
	"golang.org/x/time/rate"
)

// FallbackRateLimiter tries Redis first; on error falls back to in-memory.
// This ensures availability over strict accuracy when Redis is down.
func FallbackRateLimiter(redisCfg RedisTokenBucketConfig, memCfg MemoryRateLimiterConfig, done <-chan struct{}) gin.HandlerFunc {
	redisLimiter := RedisTokenBucketLimiter(redisCfg)

	// In-memory fallback, initialized once.
	memLimiter := MemoryRateLimiter(memCfg, done)

	// Track Redis health to avoid per-request PING overhead.
	var (
		mu          sync.RWMutex
		redisHealthy = true
		lastCheck    time.Time
		checkEvery   = 10 * time.Second
	)

	checkRedis := func(ctx context.Context) bool {
		mu.RLock()
		healthy := redisHealthy
		since := time.Since(lastCheck)
		mu.RUnlock()

		if since < checkEvery {
			return healthy
		}

		mu.Lock()
		defer mu.Unlock()
		err := redisCfg.Client.Ping(ctx).Err()
		redisHealthy = err == nil
		lastCheck = time.Now()
		if err != nil {
			slog.Warn("redis unhealthy, using in-memory rate limiter", "err", err)
		} else if !healthy {
			slog.Info("redis recovered, resuming redis rate limiter")
		}
		return redisHealthy
	}

	return func(c *gin.Context) {
		if checkRedis(c.Request.Context()) {
			redisLimiter(c)
		} else {
			memLimiter(c)
		}
	}
}
```

**Why fail open (allow) on Redis errors:** Blocking all traffic because a cache is down is worse than briefly allowing excess traffic. Log the error, alert on it, but keep the API serving. If strict enforcement is required, fail closed — change `c.Next()` to `c.AbortWithStatus(503)` in the Redis error paths above.

**Registration example:**

```go
// main.go
done := make(chan struct{})
defer close(done)

r.Use(middleware.FallbackRateLimiter(
    middleware.RedisTokenBucketConfig{
        Client:    rdb,
        Capacity:  100,
        Rate:      10,
        KeyPrefix: "rl:api:",
    },
    middleware.MemoryRateLimiterConfig{
        Rate:  10,
        Burst: 20,
    },
    done,
))
```
</file>

<file path="skills/golang-gin-api/metadata.json">
{
  "version": "1.0.5",
  "organization": "henriqueatila",
  "date": "March 2026",
  "abstract": "Core REST API patterns for Go Gin — routing, handlers, request binding/validation, middleware chains, error handling, security headers (OWASP), CORS, structured logging, timeout middleware, and layered project structure.",
  "references": [
    "https://gin-gonic.com/docs/",
    "https://pkg.go.dev/github.com/gin-gonic/gin",
    "https://context7.com/websites/gin-gonic_en"
  ]
}
</file>

<file path="skills/golang-gin-auth/metadata.json">
{
  "version": "1.0.4",
  "organization": "henriqueatila",
  "date": "March 2026",
  "abstract": "JWT authentication and RBAC for Go Gin APIs — login handlers, token lifecycle (jti blacklisting, refresh rotation), CSRF protection, rate limiting, bcrypt password hashing, secure cookies, RS256/HS256, and admin impersonation guards.",
  "references": [
    "https://gin-gonic.com/docs/",
    "https://pkg.go.dev/github.com/golang-jwt/jwt/v5",
    "https://context7.com/websites/gin-gonic_en"
  ]
}
</file>

<file path="skills/golang-gin-database/references/gorm-patterns.md">
# GORM Patterns Reference

This file covers GORM-specific patterns for PostgreSQL integration in Gin APIs: model definition, CRUD operations, soft deletes, scopes, preloading, raw SQL, batch operations, hooks, connection pooling, error handling, and a complete repository implementation. Use when building the repository layer with GORM.

> **Architectural recommendation:** These are mainstream Go/GORM community patterns, not part of the Gin framework API.

## Table of Contents

1. [Model Definition](#model-definition)
2. [Connection Pooling](#connection-pooling)
3. [CRUD Operations](#crud-operations)
4. [Soft Deletes](#soft-deletes)
5. [Scopes](#scopes)
6. [Cursor / Keyset Pagination](#cursor--keyset-pagination)
7. [Preloading Associations](#preloading-associations)
8. [Raw SQL](#raw-sql)
9. [Batch Operations](#batch-operations)
10. [Hooks — Trade-offs](#hooks--trade-offs)
11. [Error Handling](#error-handling)
12. [PostgreSQL-Specific Features](#postgresql-specific-features)
13. [Complete Repository Implementation](#complete-repository-implementation)

---

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

// toDomain converts a GORM model to the domain entity.
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

---

## Connection Pooling

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

    sqlDB.SetMaxOpenConns(cfg.MaxOpenConns)       // concurrent DB connections
    sqlDB.SetMaxIdleConns(cfg.MaxIdleConns)       // idle connections kept alive
    sqlDB.SetConnMaxLifetime(cfg.ConnMaxLifetime)  // recycle connections

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

Recommended pool settings for most APIs:

| Setting           | Value          | Why                                    |
|-------------------|----------------|----------------------------------------|
| MaxOpenConns      | 25             | Limits PostgreSQL connections          |
| MaxIdleConns      | 5              | Keeps a warm pool without waste        |
| ConnMaxLifetime   | 5 minutes      | Prevents stale connections             |

---

## CRUD Operations

```go
// Create — sets CreatedAt/UpdatedAt automatically
func (r *gormUserRepository) Create(ctx context.Context, user *domain.User) error {
    m := fromDomain(user)
    if err := r.txFromCtx(ctx).WithContext(ctx).Create(m).Error; err != nil {
        return domain.ErrInternal.New(err)
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
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, domain.ErrNotFound.New(err)
        }
        return nil, domain.ErrInternal.New(err)
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
        return domain.ErrInternal.New(result.Error)
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

---

## Soft Deletes

Adding `gorm.DeletedAt` to the model activates soft deletes automatically. GORM adds `WHERE deleted_at IS NULL` to all queries.

```go
type UserModel struct {
    // ...
    DeletedAt gorm.DeletedAt `gorm:"index"` // soft delete column
}

// Delete sets deleted_at = NOW() instead of removing the row
r.db.Delete(&UserModel{}, "id = ?", id)

// Query ignores soft-deleted rows by default
r.db.Find(&users) // WHERE deleted_at IS NULL

// Include soft-deleted rows
r.db.Unscoped().Find(&users)

// Hard delete (permanently remove)
r.db.Unscoped().Delete(&UserModel{}, "id = ?", id)
```

**Trade-off:** Soft deletes keep audit history but require `Unscoped()` everywhere you intentionally query deleted records. Consider a dedicated `archived_users` table for large datasets to avoid index bloat.

---

## Scopes

Scopes encapsulate reusable query fragments.

```go
// internal/repository/scopes.go
package repository

import "gorm.io/gorm"

// ByRole filters users by role.
func ByRole(role string) func(*gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        if role == "" {
            return db
        }
        return db.Where("role = ?", role)
    }
}

// Paginate applies offset/limit pagination.
func Paginate(page, limit int) func(*gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        if page < 1 {
            page = 1
        }
        if limit < 1 || limit > 100 {
            limit = 20
        }
        return db.Offset((page - 1) * limit).Limit(limit)
    }
}

// Usage in List:
func (r *gormUserRepository) List(ctx context.Context, opts domain.ListOptions) ([]domain.User, int64, error) {
    var models []UserModel
    var total int64

    query := r.db.WithContext(ctx).Model(&UserModel{}).Scopes(ByRole(opts.Role))

    if err := query.Count(&total).Error; err != nil {
        return nil, 0, domain.ErrInternal.New(err)
    }

    if err := query.Scopes(Paginate(opts.Page, opts.Limit)).Find(&models).Error; err != nil {
        return nil, 0, domain.ErrInternal.New(err)
    }

    users := make([]domain.User, len(models))
    for i, m := range models {
        users[i] = *m.ToDomain()
    }
    return users, total, nil
}
```

---

## Cursor / Keyset Pagination

Offset pagination (`LIMIT x OFFSET y`) performs an O(n) skip. For large or append-heavy tables, use keyset pagination — a single index seek per page.

```go
// domain/user.go — add alongside existing ListOptions
type CursorOptions struct {
    Cursor time.Time // created_at of last item on previous page; zero = first page
    Limit  int
}

// internal/repository/user_repository_gorm.go
func (r *gormUserRepository) ListAfterCursor(ctx context.Context, opts domain.CursorOptions) ([]domain.User, error) {
    limit := opts.Limit
    if limit <= 0 || limit > 100 {
        limit = 20
    }

    var models []UserModel
    q := r.txFromCtx(ctx).WithContext(ctx).Order("created_at ASC").Limit(limit)
    if !opts.Cursor.IsZero() {
        q = q.Where("created_at > ?", opts.Cursor)
    }
    if err := q.Find(&models).Error; err != nil {
        return nil, domain.ErrInternal.New(err)
    }

    users := make([]domain.User, len(models))
    for i, m := range models {
        users[i] = *m.ToDomain()
    }
    return users, nil
}
```

**Index requirement:** `CREATE INDEX idx_users_created_at ON users(created_at ASC);` — the cursor column must be indexed for O(log n) performance.

**Returning the next cursor:** In the handler or service, take `CreatedAt` of the last returned item and encode it (ISO-8601 or Unix timestamp) as `next_cursor` in the response. The client sends it back on the next request.

| | Offset | Keyset |
|---|---|---|
| Complexity | Simple | Slightly more work |
| Performance at depth | O(offset) | O(log n) |
| Row stability | Drifts on insert/delete | Stable |
| Arbitrary page jump | Yes | No |

**Recommendation:** Use offset for small tables and admin UIs; keyset for feeds, audit logs, and any table that grows quickly.

---

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

**Warning:** `Preload` issues one extra query per association, not one per row (GORM batches it). Still, avoid preloading large datasets — use `Joins` for filtering.

---

## Raw SQL

Use raw SQL for complex queries that GORM can't express cleanly.

```go
// Scan into a custom struct — not a GORM model
type UserStats struct {
    Role  string `gorm:"column:role"`
    Count int64  `gorm:"column:count"`
}

func (r *gormUserRepository) StatsByRole(ctx context.Context) ([]UserStats, error) {
    var stats []UserStats
    if err := r.db.WithContext(ctx).Raw(`
        SELECT role, COUNT(*) AS count
        FROM users
        WHERE deleted_at IS NULL
        GROUP BY role
        ORDER BY count DESC
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

---

## Batch Operations

```go
// CreateInBatches — inserts in chunks to avoid parameter limit
func (r *gormUserRepository) BulkCreate(ctx context.Context, users []domain.User) error {
    models := make([]UserModel, len(users))
    for i, u := range users {
        models[i] = *fromDomain(&u)
    }
    // batchSize=100 — tune based on row size and DB limits
    if err := r.db.WithContext(ctx).CreateInBatches(models, 100).Error; err != nil {
        return domain.ErrInternal.New(err)
    }
    return nil
}

// FindInBatches — process large result sets without loading all rows into memory
func (r *gormUserRepository) ExportAll(ctx context.Context, process func([]domain.User) error) error {
    var models []UserModel // declared outside: GORM populates this each batch
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

---

## Hooks — Trade-offs

GORM hooks (`BeforeCreate`, `AfterCreate`, etc.) run automatically but introduce hidden side effects.

```go
// Use hooks for truly cross-cutting concerns (e.g., UUID generation)
func (m *UserModel) BeforeCreate(tx *gorm.DB) error {
    if m.ID == "" {
        m.ID = uuid.Must(uuid.NewV7()).String() // UUIDv7: time-sortable, better B-tree index performance
    }
    return nil
}
```

**Trade-offs:**

| Pros | Cons |
|------|------|
| Zero boilerplate for UUID gen | Hidden logic — hard to trace |
| Consistent across all create paths | Can't be disabled per call |
| Works with transactions | Tested separately from service logic |

**Recommendation:** Use hooks for `ID` generation and `UpdatedAt` only. Move business logic (hashing passwords, sending emails) to the service layer where it's explicit and testable.

---

## Error Handling

Map GORM errors to domain errors. Never leak raw GORM errors to the handler layer.

```go
// internal/repository/errors.go
package repository

import (
    "errors"

    "github.com/jackc/pgx/v5/pgconn"
    "gorm.io/gorm"
    "myapp/internal/domain"
)

// mapGORMError converts a GORM error to a domain error.
func mapGORMError(err error) error {
    if errors.Is(err, gorm.ErrRecordNotFound) {
        return domain.ErrNotFound.New(err)
    }
    // PostgreSQL unique violation (SQLSTATE 23505) — typed assertion, not string matching
    var pgErr *pgconn.PgError
    if errors.As(err, &pgErr) && pgErr.Code == "23505" {
        return domain.ErrConflict.New(err)
    }
    return domain.ErrInternal.New(err)
}

// Usage — consistent error mapping in every repository method:
func (r *gormUserRepository) Create(ctx context.Context, user *domain.User) error {
    m := fromDomain(user)
    if err := r.db.WithContext(ctx).Create(m).Error; err != nil {
        return mapGORMError(err)
    }
    user.ID = m.ID
    return nil
}
```

**Critical:** Log the raw GORM error at the service layer (it includes the SQL state). Return only the domain error to the handler.

---

## PostgreSQL-Specific Features

```go
import (
    "gorm.io/gorm"
    "gorm.io/gorm/clause"
    "github.com/lib/pq"      // pq.StringArray for PostgreSQL array columns
    "gorm.io/datatypes"      // datatypes.JSON for JSONB columns
)

// ON CONFLICT DO NOTHING — idempotent insert
r.db.WithContext(ctx).
    Clauses(clause.OnConflict{DoNothing: true}).
    Create(&m)

// ON CONFLICT DO UPDATE (upsert)
r.db.WithContext(ctx).
    Clauses(clause.OnConflict{
        Columns:   []clause.Column{{Name: "email"}},
        DoUpdates: clause.AssignmentColumns([]string{"name", "updated_at"}),
    }).
    Create(&m)

// RETURNING — get generated values after insert
r.db.WithContext(ctx).
    Clauses(clause.Returning{Columns: []clause.Column{{Name: "id"}, {Name: "created_at"}}}).
    Create(&m)

// Array type (requires github.com/lib/pq)
type UserModel struct {
    Tags pq.StringArray `gorm:"type:text[]"`
}

// JSONB column
type UserModel struct {
    Metadata datatypes.JSON `gorm:"type:jsonb"`
}
```

---

## Complete Repository Implementation

Full `UserRepository` satisfying the `domain.UserRepository` interface. All methods use `txFromCtx` so they transparently participate in a service-layer transaction when one is present in context.

```go
// internal/repository/tx.go
package repository

import (
    "context"

    "gorm.io/gorm"
)

type txKey struct{}

// WithTx stores a *gorm.DB transaction in ctx.
// Call this in the service layer before passing ctx to repositories.
func WithTx(ctx context.Context, tx *gorm.DB) context.Context {
    return context.WithValue(ctx, txKey{}, tx)
}

// txFromCtx returns the transaction stored in ctx, or the repository's default db.
// Every repository write method should call this instead of using r.db directly.
func (r *gormUserRepository) txFromCtx(ctx context.Context) *gorm.DB {
    if tx, ok := ctx.Value(txKey{}).(*gorm.DB); ok {
        return tx
    }
    return r.db
}
```

```go
// internal/repository/user_repository_gorm.go
package repository

import (
    "context"
    "fmt"
    "time"

    "gorm.io/gorm"
    "myapp/internal/domain"
)

type gormUserRepository struct {
    db *gorm.DB
}

// NewUserRepository returns a domain.UserRepository backed by GORM.
func NewUserRepository(db *gorm.DB) domain.UserRepository {
    return &gormUserRepository{db: db}
}

func (r *gormUserRepository) Create(ctx context.Context, user *domain.User) error {
    m := fromDomain(user)
    if err := r.txFromCtx(ctx).WithContext(ctx).Create(m).Error; err != nil {
        return mapGORMError(err)
    }
    user.ID = m.ID
    user.CreatedAt = m.CreatedAt
    user.UpdatedAt = m.UpdatedAt
    return nil
}

func (r *gormUserRepository) GetByID(ctx context.Context, id string) (*domain.User, error) {
    var m UserModel
    if err := r.txFromCtx(ctx).WithContext(ctx).First(&m, "id = ?", id).Error; err != nil {
        return nil, mapGORMError(err)
    }
    return m.ToDomain(), nil
}

func (r *gormUserRepository) GetByEmail(ctx context.Context, email string) (*domain.User, error) {
    var m UserModel
    if err := r.txFromCtx(ctx).WithContext(ctx).First(&m, "email = ?", email).Error; err != nil {
        return nil, mapGORMError(err)
    }
    return m.ToDomain(), nil
}

func (r *gormUserRepository) List(ctx context.Context, opts domain.ListOptions) ([]domain.User, int64, error) {
    var models []UserModel
    var total int64

    q := r.txFromCtx(ctx).WithContext(ctx).Model(&UserModel{}).Scopes(ByRole(opts.Role))

    if err := q.Count(&total).Error; err != nil {
        return nil, 0, domain.ErrInternal.New(err)
    }
    if err := q.Scopes(Paginate(opts.Page, opts.Limit)).Find(&models).Error; err != nil {
        return nil, 0, domain.ErrInternal.New(err)
    }

    users := make([]domain.User, len(models))
    for i, m := range models {
        users[i] = *m.ToDomain()
    }
    return users, total, nil
}

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

**Service-layer transaction usage:**

```go
// internal/service/user_service.go
func (s *userService) RegisterWithProfile(ctx context.Context, req domain.CreateUserRequest) error {
    return s.db.WithContext(ctx).Transaction(func(tx *gorm.DB) error {
        txCtx := repository.WithTx(ctx, tx)
        if err := s.userRepo.Create(txCtx, &user); err != nil {
            return err // automatic rollback
        }
        return s.profileRepo.Create(txCtx, &profile)
    })
}
```
</file>

<file path="skills/golang-gin-testing/references/unit-tests.md">
# Unit Tests — Handlers, Services, and Middleware

This file covers: handler testing with `httptest`, testing authenticated routes with a mock JWT, service testing with manual mocks, middleware isolation, table-driven subtests, `t.Helper`/`t.Cleanup`/`t.Parallel`, and test fixtures.

All examples use the same `User` domain model and `AppError` pattern from **golang-gin-api**.

## Table of Contents

1. [Handler Testing with httptest](#handler-testing-with-httptest)
2. [Testing JSON and Error Responses](#testing-json-and-error-responses)
3. [Testing Authenticated Routes](#testing-authenticated-routes)
4. [Service Testing with Mocks](#service-testing-with-mocks)
5. [Mock Generation Patterns](#mock-generation-patterns)
6. [Table-Driven Tests with Subtests](#table-driven-tests-with-subtests)
7. [Test Fixtures and Factories](#test-fixtures-and-factories)
8. [t.Helper / t.Cleanup / t.Parallel](#thelper--tcleanup--tparallel)
9. [Testing Middleware in Isolation](#testing-middleware-in-isolation)
10. [Benchmark Tests](#benchmark-tests)
11. [Fuzz Tests](#fuzz-tests)
12. [Golden File / Snapshot Testing](#golden-file--snapshot-testing)
13. [Test Organization](#test-organization)
14. [Testify as an Alternative](#testify-as-an-alternative)

---

## Handler Testing with httptest

**Why:** Handlers translate HTTP ↔ domain. Test the HTTP contract (status codes, JSON shape, headers), not business logic. Business logic belongs in service tests.

**Pattern:** `httptest.NewRecorder()` + `router.ServeHTTP(w, req)`

```go
// internal/handler/user_handler_test.go
package handler_test

import (
    "context"
    "log/slog"
    "net/http"
    "net/http/httptest"
    "strings"
    "testing"
    "time"

    "github.com/gin-gonic/gin"
    "myapp/internal/domain"
    "myapp/internal/handler"
    "myapp/internal/service"
)

func init() {
    gin.SetMode(gin.TestMode)
}

// setupRouter builds a test router with a real handler + mock service.
// Keep route registration here — it documents the contract being tested.
func setupUserRouter(svc service.UserService) *gin.Engine {
    r := gin.New() // gin.New(), not gin.Default() — no Logger noise in test output
    h := handler.NewUserHandler(svc, slog.Default())
    r.POST("/users", h.Create)
    r.GET("/users/:id", h.GetByID)
    r.PUT("/users/:id", h.Update)
    r.DELETE("/users/:id", h.Delete)
    return r
}

func TestUserHandler_GetByID(t *testing.T) {
    now := time.Now().UTC().Truncate(time.Second)
    svc := &mockUserService{
        getByIDFn: func(_ context.Context, id string) (*domain.User, error) {
            if id == "user-123" {
                return &domain.User{
                    ID:        "user-123",
                    Name:      "Alice",
                    Email:     "alice@example.com",
                    Role:      "user",
                    CreatedAt: now,
                    UpdatedAt: now,
                }, nil
            }
            return nil, domain.ErrNotFound
        },
    }

    router := setupUserRouter(svc)

    req := httptest.NewRequest(http.MethodGet, "/users/user-123", nil)
    w := httptest.NewRecorder()
    router.ServeHTTP(w, req)

    if w.Code != http.StatusOK {
        t.Fatalf("want 200, got %d; body: %s", w.Code, w.Body)
    }
    if ct := w.Header().Get("Content-Type"); !strings.Contains(ct, "application/json") {
        t.Errorf("want Content-Type application/json, got %q", ct)
    }
}
```

---

## Testing JSON and Error Responses

Verify the exact JSON shape returned — both success and error paths.

```go
// internal/handler/user_handler_json_test.go
package handler_test

import (
    "context"
    "encoding/json"
    "net/http"
    "net/http/httptest"
    "strings"
    "testing"

    "myapp/internal/domain"
)

func TestUserHandler_Create_ReturnsUserJSON(t *testing.T) {
    svc := &mockUserService{
        createFn: func(_ context.Context, req domain.CreateUserRequest) (*domain.User, error) {
            return &domain.User{
                ID:    "new-id",
                Name:  req.Name,
                Email: req.Email,
                Role:  "user",
            }, nil
        },
    }
    router := setupUserRouter(svc)

    body := `{"name":"Bob","email":"bob@example.com","password":"secret123"}`
    req := httptest.NewRequest(http.MethodPost, "/users", strings.NewReader(body))
    req.Header.Set("Content-Type", "application/json")
    w := httptest.NewRecorder()
    router.ServeHTTP(w, req)

    if w.Code != http.StatusCreated {
        t.Fatalf("want 201, got %d; body: %s", w.Code, w.Body)
    }

    var resp domain.User
    if err := json.Unmarshal(w.Body.Bytes(), &resp); err != nil {
        t.Fatalf("failed to parse response JSON: %v\nbody: %s", err, w.Body)
    }
    if resp.ID != "new-id" {
        t.Errorf("want ID 'new-id', got %q", resp.ID)
    }
    if resp.Email != "bob@example.com" {
        t.Errorf("want email 'bob@example.com', got %q", resp.Email)
    }
    // Password must never be returned
    if strings.Contains(w.Body.String(), "secret123") {
        t.Error("response body must not contain plain-text password")
    }
}

func TestUserHandler_GetByID_ErrorResponse(t *testing.T) {
    svc := &mockUserService{
        getByIDFn: func(_ context.Context, id string) (*domain.User, error) {
            return nil, domain.ErrNotFound
        },
    }
    router := setupUserRouter(svc)

    req := httptest.NewRequest(http.MethodGet, "/users/ghost", nil)
    w := httptest.NewRecorder()
    router.ServeHTTP(w, req)

    if w.Code != http.StatusNotFound {
        t.Fatalf("want 404, got %d", w.Code)
    }

    var errResp map[string]string
    if err := json.Unmarshal(w.Body.Bytes(), &errResp); err != nil {
        t.Fatalf("failed to parse error JSON: %v", err)
    }
    if errResp["error"] == "" {
        t.Error("error response must contain 'error' field")
    }
}
```

---

## Testing Authenticated Routes

Inject a real JWT token generated with the test secret, then apply the `Auth` middleware in the test router.

```go
// internal/handler/protected_handler_test.go
package handler_test

import (
    "context"
    "log/slog"
    "net/http"
    "net/http/httptest"
    "testing"
    "time"

    "github.com/gin-gonic/gin"
    "myapp/internal/auth"
    "myapp/internal/domain"
    "myapp/internal/handler"
    "myapp/internal/service"
    "myapp/pkg/middleware"
)

// testTokenConfig returns a deterministic TokenConfig for tests.
func testTokenConfig() auth.TokenConfig {
    return auth.TokenConfig{
        AccessSecret:  []byte("test-access-secret-32-bytes-long!!"),
        RefreshSecret: []byte("test-refresh-secret-32-bytes-long!"),
        AccessTTL:     15 * time.Minute,
        RefreshTTL:    7 * 24 * time.Hour,
    }
}

// generateTestToken creates a valid signed JWT for test assertions.
func generateTestToken(t *testing.T, userID, email, role string) string {
    t.Helper()
    cfg := testTokenConfig()
    token, err := auth.GenerateAccessToken(cfg, userID, email, role)
    if err != nil {
        t.Fatalf("failed to generate test token: %v", err)
    }
    return token
}

func setupProtectedRouter(svc service.UserService) *gin.Engine {
    r := gin.New()
    cfg := testTokenConfig()
    protected := r.Group("")
    protected.Use(middleware.Auth(cfg, slog.Default()))
    {
        h := handler.NewUserHandler(svc, slog.Default())
        protected.GET("/users/:id", h.GetByID)
    }
    return r
}

func TestProtectedRoute_WithValidToken(t *testing.T) {
    svc := &mockUserService{
        getByIDFn: func(_ context.Context, id string) (*domain.User, error) {
            return &domain.User{ID: id, Name: "Alice", Email: "alice@example.com", Role: "user"}, nil
        },
    }
    router := setupProtectedRouter(svc)
    token := generateTestToken(t, "user-123", "alice@example.com", "user")

    req := httptest.NewRequest(http.MethodGet, "/users/user-123", nil)
    req.Header.Set("Authorization", "Bearer "+token)
    w := httptest.NewRecorder()
    router.ServeHTTP(w, req)

    if w.Code != http.StatusOK {
        t.Errorf("want 200, got %d; body: %s", w.Code, w.Body)
    }
}

func TestProtectedRoute_MissingToken(t *testing.T) {
    router := setupProtectedRouter(&mockUserService{})

    req := httptest.NewRequest(http.MethodGet, "/users/user-123", nil)
    // No Authorization header
    w := httptest.NewRecorder()
    router.ServeHTTP(w, req)

    if w.Code != http.StatusUnauthorized {
        t.Errorf("want 401, got %d", w.Code)
    }
}

func TestProtectedRoute_ExpiredToken(t *testing.T) {
    cfg := auth.TokenConfig{
        AccessSecret: []byte("test-access-secret-32-bytes-long!!"),
        AccessTTL:    -1 * time.Minute, // already expired
    }
    token, err := auth.GenerateAccessToken(cfg, "u1", "a@b.com", "user")
    if err != nil {
        t.Fatal(err)
    }

    router := setupProtectedRouter(&mockUserService{})
    req := httptest.NewRequest(http.MethodGet, "/users/u1", nil)
    req.Header.Set("Authorization", "Bearer "+token)
    w := httptest.NewRecorder()
    router.ServeHTTP(w, req)

    if w.Code != http.StatusUnauthorized {
        t.Errorf("want 401 for expired token, got %d", w.Code)
    }
}
```

---

## Service Testing with Mocks

Test business logic — password hashing, conflict detection, error wrapping — independent of HTTP or DB.

```go
// internal/service/user_service_test.go
package service_test

import (
    "context"
    "errors"
    "log/slog"
    "testing"

    "myapp/internal/domain"
    "myapp/internal/service"
)

type mockUserRepository struct {
    createFn     func(ctx context.Context, user *domain.User) error
    getByEmailFn func(ctx context.Context, email string) (*domain.User, error)
    getByIDFn    func(ctx context.Context, id string) (*domain.User, error)
    listFn       func(ctx context.Context, opts domain.ListOptions) ([]domain.User, int64, error)
    updateFn     func(ctx context.Context, user *domain.User) error
    deleteFn     func(ctx context.Context, id string) error
}

func (m *mockUserRepository) Create(ctx context.Context, user *domain.User) error {
    if m.createFn != nil {
        return m.createFn(ctx, user)
    }
    return nil
}
func (m *mockUserRepository) GetByEmail(ctx context.Context, email string) (*domain.User, error) {
    if m.getByEmailFn != nil {
        return m.getByEmailFn(ctx, email)
    }
    return nil, domain.ErrNotFound
}
func (m *mockUserRepository) GetByID(ctx context.Context, id string) (*domain.User, error) {
    if m.getByIDFn != nil {
        return m.getByIDFn(ctx, id)
    }
    return nil, domain.ErrNotFound
}
func (m *mockUserRepository) List(ctx context.Context, opts domain.ListOptions) ([]domain.User, int64, error) {
    if m.listFn != nil {
        return m.listFn(ctx, opts)
    }
    return nil, 0, nil
}
func (m *mockUserRepository) Update(ctx context.Context, user *domain.User) error {
    if m.updateFn != nil {
        return m.updateFn(ctx, user)
    }
    return nil
}
func (m *mockUserRepository) Delete(ctx context.Context, id string) error {
    if m.deleteFn != nil {
        return m.deleteFn(ctx, id)
    }
    return nil
}

func TestUserService_Create_HashesPassword(t *testing.T) {
    var savedUser *domain.User
    repo := &mockUserRepository{
        getByEmailFn: func(_ context.Context, _ string) (*domain.User, error) {
            return nil, domain.ErrNotFound // email not taken
        },
        createFn: func(_ context.Context, user *domain.User) error {
            savedUser = user
            return nil
        },
    }

    svc := service.NewUserService(repo, slog.Default())
    _, err := svc.Create(context.Background(), domain.CreateUserRequest{
        Name:     "Alice",
        Email:    "alice@example.com",
        Password: "plaintext",
    })
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    if savedUser == nil {
        t.Fatal("expected user to be saved")
    }
    // Password must be stored as bcrypt hash, not plaintext
    if savedUser.PasswordHash == "plaintext" {
        t.Error("password must not be stored as plaintext")
    }
    if savedUser.PasswordHash == "" {
        t.Error("expected PasswordHash to be set")
    }
}

func TestUserService_Create_ConflictOnDuplicateEmail(t *testing.T) {
    repo := &mockUserRepository{
        getByEmailFn: func(_ context.Context, email string) (*domain.User, error) {
            return &domain.User{Email: email}, nil // email already taken
        },
    }

    svc := service.NewUserService(repo, slog.Default())
    _, err := svc.Create(context.Background(), domain.CreateUserRequest{
        Name:     "Alice",
        Email:    "taken@example.com",
        Password: "secret123",
    })

    var appErr *domain.AppError
    if !errors.As(err, &appErr) || appErr.Code != 409 {
        t.Errorf("expected ErrConflict (409), got %v", err)
    }
}
```

---

## Mock Generation Patterns

### Manual mocks (recommended for small interfaces)

Define mock structs with function fields (shown above). Simple, explicit, no extra dependencies.

### gomock (recommended for large interfaces)

```bash
go install go.uber.org/mock/mockgen@latest

# Generate mock for UserRepository interface
mockgen -source=internal/domain/user.go -destination=internal/testutil/mocks/mock_user_repository.go -package=mocks
```

```go
// Using generated mock
import (
    "context"
    "log/slog"
    "testing"

    "go.uber.org/mock/gomock"
    "myapp/internal/domain"
    "myapp/internal/service"
    "myapp/internal/testutil/mocks"
)

func TestWithGoMock(t *testing.T) {
    ctrl := gomock.NewController(t)
    repo := mocks.NewMockUserRepository(ctrl)
    repo.EXPECT().
        GetByEmail(gomock.Any(), "alice@example.com").
        Return(nil, domain.ErrNotFound)
    repo.EXPECT().
        Create(gomock.Any(), gomock.Any()).
        Return(nil)

    svc := service.NewUserService(repo, slog.Default())
    _, err := svc.Create(context.Background(), domain.CreateUserRequest{
        Name:     "Alice",
        Email:    "alice@example.com",
        Password: "secret123",
    })
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
}
```

**Trade-off:** Manual mocks are zero-dependency and easier to debug. gomock catches unexpected calls automatically, which helps on large interfaces.

---

## Table-Driven Tests with Subtests

Structure: define a `tests` slice, loop with `t.Run`, and optionally `t.Parallel()` for speed.

```go
func TestUserService_GetByID(t *testing.T) {
    existingUser := &domain.User{ID: "u1", Name: "Alice", Email: "alice@example.com", Role: "user"}

    tests := []struct {
        name      string
        id        string
        repoUser  *domain.User
        repoErr   error
        wantUser  *domain.User
        wantErr   error
    }{
        {
            name:     "found",
            id:       "u1",
            repoUser: existingUser,
            wantUser: existingUser,
        },
        {
            name:    "not found",
            id:      "missing",
            repoErr: domain.ErrNotFound,
            wantErr: domain.ErrNotFound,
        },
        {
            name:    "repository error",
            id:      "any",
            repoErr: errors.New("db connection lost"),
            wantErr: domain.ErrInternal,
        },
    }

    for _, tc := range tests {
        t.Run(tc.name, func(t *testing.T) {
            t.Parallel()

            repo := &mockUserRepository{
                getByIDFn: func(_ context.Context, id string) (*domain.User, error) {
                    return tc.repoUser, tc.repoErr
                },
            }
            svc := service.NewUserService(repo, slog.Default())
            got, err := svc.GetByID(context.Background(), tc.id)

            if tc.wantErr != nil {
                var appErr *domain.AppError
                if !errors.As(err, &appErr) {
                    t.Errorf("want AppError wrapping %v, got %T: %v", tc.wantErr, err, err)
                }
                return
            }
            if err != nil {
                t.Fatalf("unexpected error: %v", err)
            }
            if got.ID != tc.wantUser.ID {
                t.Errorf("want user ID %q, got %q", tc.wantUser.ID, got.ID)
            }
        })
    }
}
```

---

## Test Fixtures and Factories

Factories build valid domain objects in one line. Keeps tests focused on what varies, not boilerplate.

```go
// internal/testutil/fixtures.go
package testutil

import (
    "time"

    "myapp/internal/domain"
)

// UserFixture returns a valid User. Override fields in the caller as needed.
func UserFixture(overrides ...func(*domain.User)) *domain.User {
    u := &domain.User{
        ID:        "fixture-user-id",
        Name:      "Test User",
        Email:     "test@example.com",
        Role:      "user",
        CreatedAt: time.Date(2024, 1, 1, 0, 0, 0, 0, time.UTC),
        UpdatedAt: time.Date(2024, 1, 1, 0, 0, 0, 0, time.UTC),
    }
    for _, fn := range overrides {
        fn(u)
    }
    return u
}

// CreateUserRequestFixture returns a valid CreateUserRequest.
func CreateUserRequestFixture(overrides ...func(*domain.CreateUserRequest)) domain.CreateUserRequest {
    req := domain.CreateUserRequest{
        Name:     "Test User",
        Email:    "test@example.com",
        Password: "secret1234",
        Role:     "user",
    }
    for _, fn := range overrides {
        fn(&req)
    }
    return req
}
```

Usage — only specify what differs from the fixture:

```go
adminUser := testutil.UserFixture(func(u *domain.User) {
    u.Role = "admin"
    u.Email = "admin@example.com"
})

noRoleReq := testutil.CreateUserRequestFixture(func(r *domain.CreateUserRequest) {
    r.Role = "" // omitempty — valid
})
```

---

## t.Helper / t.Cleanup / t.Parallel

```go
// t.Helper — marks the function as a helper, so failures show the caller's line, not the helper's
func assertStatus(t *testing.T, w *httptest.ResponseRecorder, want int) {
    t.Helper() // <-- ALWAYS add to assertion helpers
    if w.Code != want {
        t.Errorf("want status %d, got %d; body: %s", want, w.Code, w.Body)
    }
}

// t.Cleanup — deferred cleanup that runs even if the test panics
func createTempFile(t *testing.T) string {
    t.Helper()
    f, err := os.CreateTemp("", "test-*")
    if err != nil {
        t.Fatalf("createTempFile: %v", err)
    }
    t.Cleanup(func() { os.Remove(f.Name()) }) // always cleaned up
    return f.Name()
}

// t.Parallel — safe for stateless tests; do NOT use with shared mutable state
func TestHandlerConcurrency(t *testing.T) {
    t.Parallel() // this test can run concurrently with other parallel tests
    // ...
}
```

**When NOT to use `t.Parallel()`:** when tests share a database, write to files, or mutate package-level state.

---

## Testing Middleware in Isolation

Test middleware logic (JWT validation, rate limiting, RBAC) independently of any handler.

```go
// pkg/middleware/auth_test.go
package middleware_test

import (
    "log/slog"
    "net/http"
    "net/http/httptest"
    "testing"
    "time"

    "github.com/gin-gonic/gin"
    "myapp/internal/auth"
    "myapp/pkg/middleware"
)

func init() { gin.SetMode(gin.TestMode) }

// sentinel handler confirms the middleware allowed the request through
var sentinelHandler = func(c *gin.Context) {
    c.JSON(http.StatusOK, gin.H{"reached": true})
}

func setupAuthMiddlewareRouter(cfg auth.TokenConfig) *gin.Engine {
    r := gin.New()
    r.Use(middleware.Auth(cfg, slog.Default()))
    r.GET("/protected", sentinelHandler)
    return r
}

func TestAuthMiddleware_RejectsNoHeader(t *testing.T) {
    cfg := auth.TokenConfig{
        AccessSecret: []byte("test-secret-32-bytes-exactly-!!!"),
        AccessTTL:    15 * time.Minute,
    }
    router := setupAuthMiddlewareRouter(cfg)

    req := httptest.NewRequest(http.MethodGet, "/protected", nil)
    w := httptest.NewRecorder()
    router.ServeHTTP(w, req)

    if w.Code != http.StatusUnauthorized {
        t.Errorf("want 401, got %d", w.Code)
    }
}

func TestAuthMiddleware_RejectsMalformedHeader(t *testing.T) {
    cfg := auth.TokenConfig{AccessSecret: []byte("test-secret-32-bytes-exactly-!!!")}
    router := setupAuthMiddlewareRouter(cfg)

    cases := []string{"token-without-bearer", "Basic abc123", "Bearer", ""}
    for _, h := range cases {
        t.Run(h, func(t *testing.T) {
            req := httptest.NewRequest(http.MethodGet, "/protected", nil)
            if h != "" {
                req.Header.Set("Authorization", h)
            }
            w := httptest.NewRecorder()
            router.ServeHTTP(w, req)
            if w.Code != http.StatusUnauthorized {
                t.Errorf("header=%q: want 401, got %d", h, w.Code)
            }
        })
    }
}

func TestAuthMiddleware_InjectsClaimsOnSuccess(t *testing.T) {
    cfg := auth.TokenConfig{
        AccessSecret: []byte("test-secret-32-bytes-exactly-!!!"),
        AccessTTL:    15 * time.Minute,
    }

    // Capture injected claims in a custom handler
    var capturedUserID string
    r := gin.New()
    r.Use(middleware.Auth(cfg, slog.Default()))
    r.GET("/me", func(c *gin.Context) {
        capturedUserID = c.GetString(middleware.UserIDKey)
        c.JSON(http.StatusOK, gin.H{"ok": true})
    })

    token, err := auth.GenerateAccessToken(cfg, "user-xyz", "u@example.com", "user")
    if err != nil {
        t.Fatal(err)
    }

    req := httptest.NewRequest(http.MethodGet, "/me", nil)
    req.Header.Set("Authorization", "Bearer "+token)
    w := httptest.NewRecorder()
    r.ServeHTTP(w, req)

    if w.Code != http.StatusOK {
        t.Fatalf("want 200, got %d; body: %s", w.Code, w.Body)
    }
    if capturedUserID != "user-xyz" {
        t.Errorf("want user_id 'user-xyz' injected by middleware, got %q", capturedUserID)
    }
}
```

---

## Benchmark Tests

Use `Benchmark*` functions to measure performance of hot paths — JSON serialization, handler throughput, service logic. Run with `go test -bench=. -benchmem ./...`.

```go
// internal/handler/user_handler_bench_test.go
package handler_test

import (
    "context"
    "net/http"
    "net/http/httptest"
    "testing"

    "myapp/internal/domain"
)

// BenchmarkUserHandler_GetByID measures handler throughput including
// JSON marshaling and routing overhead.
func BenchmarkUserHandler_GetByID(b *testing.B) {
    svc := &mockUserService{
        getByIDFn: func(_ context.Context, id string) (*domain.User, error) {
            return &domain.User{
                ID:    id,
                Name:  "Alice",
                Email: "alice@example.com",
                Role:  "user",
            }, nil
        },
    }
    router := setupUserRouter(svc)

    b.ReportAllocs() // report allocations per operation
    b.ResetTimer()

    for b.Loop() {
        req := httptest.NewRequest(http.MethodGet, "/users/user-123", nil)
        w := httptest.NewRecorder()
        router.ServeHTTP(w, req)
        if w.Code != http.StatusOK {
            b.Fatalf("unexpected status %d", w.Code)
        }
    }
}

// BenchmarkUserHandler_GetByID_Parallel runs the same benchmark
// with multiple goroutines to expose concurrency bottlenecks.
func BenchmarkUserHandler_GetByID_Parallel(b *testing.B) {
    svc := &mockUserService{
        getByIDFn: func(_ context.Context, id string) (*domain.User, error) {
            return &domain.User{ID: id, Name: "Alice", Email: "alice@example.com", Role: "user"}, nil
        },
    }
    router := setupUserRouter(svc)

    b.ReportAllocs()
    b.ResetTimer()

    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            req := httptest.NewRequest(http.MethodGet, "/users/user-123", nil)
            w := httptest.NewRecorder()
            router.ServeHTTP(w, req)
        }
    })
}
```

Run benchmarks:

```bash
# All benchmarks in the package
go test -bench=. -benchmem ./internal/handler/...

# Specific benchmark, 5 seconds run time
go test -bench=BenchmarkUserHandler_GetByID -benchtime=5s -benchmem ./internal/handler/...

# Compare two versions with benchstat
go test -bench=. -benchmem -count=5 ./... > before.txt
# (make changes)
go test -bench=. -benchmem -count=5 ./... > after.txt
benchstat before.txt after.txt
```

---

## Fuzz Tests

Go 1.18+ native fuzzing (`func FuzzX(f *testing.F)`) finds edge cases in input parsing that table-driven tests miss. Use for request binding, validators, and string-processing utilities.

```go
// internal/handler/user_handler_fuzz_test.go
package handler_test

import (
    "context"
    "net/http"
    "net/http/httptest"
    "strings"
    "testing"

    "myapp/internal/domain"
)

// FuzzUserHandler_Create exercises the Create handler with arbitrary JSON bodies.
// The fuzzer discovers inputs that cause panics or unexpected status codes.
//
// Run the fuzzer: go test -fuzz=FuzzUserHandler_Create ./internal/handler/...
// Run the corpus only (fast, CI-safe): go test ./internal/handler/...
func FuzzUserHandler_Create(f *testing.F) {
    // Seed corpus — representative inputs the fuzzer mutates from
    f.Add(`{"name":"Alice","email":"alice@example.com","password":"secret123"}`)
    f.Add(`{"name":"","email":"bad-email","password":"x"}`)
    f.Add(`{}`)
    f.Add(`not-json`)
    f.Add(`{"name":null,"email":null}`)

    svc := &mockUserService{
        createFn: func(_ context.Context, req domain.CreateUserRequest) (*domain.User, error) {
            return &domain.User{ID: "fuzz-id", Name: req.Name, Email: req.Email, Role: "user"}, nil
        },
    }
    router := setupUserRouter(svc)

    f.Fuzz(func(t *testing.T, body string) {
        req := httptest.NewRequest(http.MethodPost, "/users", strings.NewReader(body))
        req.Header.Set("Content-Type", "application/json")
        w := httptest.NewRecorder()
        router.ServeHTTP(w, req)

        // The handler must NEVER panic (recovered by Gin) and must always
        // return a valid HTTP status — never a 5xx for invalid client input.
        if w.Code == http.StatusInternalServerError {
            t.Errorf("handler returned 500 for input %q; body: %s", body, w.Body)
        }
    })
}
```

**Key rules for fuzz tests:**
- Seed corpus must include valid inputs and known edge cases
- The fuzz function must not have non-deterministic behavior (no random, no time)
- Run in CI with `-fuzz=` omitted (seed corpus only) — fuzzing itself runs locally or on dedicated infrastructure
- Found crash inputs are saved to `testdata/fuzz/FuzzX/` automatically

---

## Golden File / Snapshot Testing

Golden files capture the expected output of a function as a file on disk. On subsequent runs the output is compared to the stored file. Useful for JSON responses, rendered templates, or any output too verbose to inline.

```go
// internal/handler/user_handler_golden_test.go
package handler_test

import (
    "encoding/json"
    "flag"
    "os"
    "path/filepath"
    "testing"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
    "myapp/internal/domain"
)

// Run with -update to regenerate golden files:
//   go test ./internal/handler/... -update
var update = flag.Bool("update", false, "update golden files")

func TestUserResponse_Golden(t *testing.T) {
    user := domain.User{ID: "test-id", Email: "test@example.com", Name: "Test User"}
    body, err := json.MarshalIndent(user, "", "  ")
    require.NoError(t, err)

    golden := filepath.Join("testdata", t.Name()+".golden")
    if *update {
        os.MkdirAll("testdata", 0o755)
        os.WriteFile(golden, body, 0o644)
    }
    expected, err := os.ReadFile(golden)
    require.NoError(t, err)
    assert.JSONEq(t, string(expected), string(body))
}
```

**How it works:**

1. First run: `go test ./... -update` writes `testdata/TestUserResponse_Golden.golden`
2. Subsequent runs compare current output to the stored file
3. Commit golden files to source control — diffs surface unintended response changes
4. When the response shape intentionally changes, re-run with `-update` and commit the new file

Golden files live in `testdata/` next to the test file. The `testdata/` directory is ignored by the Go build toolchain but committed to git.

---

## Test Organization

Go supports two complementary test styles in the same package directory:

### Same-package tests (white-box)

```go
// File: internal/service/user_service_test.go
package service  // same package — can access unexported identifiers
```

Use for: verifying internal state, unexported helper functions, implementation details that should not be part of the public API.

### External-package tests (black-box)

```go
// File: internal/handler/user_handler_test.go
package handler_test  // _test suffix — only exported API visible
```

Use for: handlers, middleware, repositories. Tests the public contract; prevents accidental coupling to internals.

**Rule of thumb:** handlers and services use `package X_test` (black-box). Internal utility packages may use `package X` (white-box) when testing unexported logic.

### Build tags for integration tests

```go
//go:build integration

// Place at the top of every integration test file, before the package declaration.
// Excluded from: go test ./...
// Included in:   go test -tags=integration ./...
```

Use `//go:build integration` for tests that require Docker/network. Use `//go:build e2e` for full end-to-end tests. This keeps `go test ./...` fast for everyday development.

---

## Testify as an Alternative

The standard library `testing` package is sufficient for most cases. In enterprise codebases, [testify](https://github.com/testify-community/testify) (`github.com/stretchr/testify`) is widely adopted for its readable assertions and reduced boilerplate.

```bash
go get github.com/stretchr/testify
```

**Comparison:**

```go
// Standard library
if got.Email != "alice@example.com" {
    t.Errorf("want email 'alice@example.com', got %q", got.Email)
}
if err != nil {
    t.Fatalf("unexpected error: %v", err)
}

// testify/assert — continues test on failure
assert.Equal(t, "alice@example.com", got.Email)
assert.NoError(t, err)

// testify/require — stops test immediately on failure (equivalent to t.Fatalf)
require.NoError(t, err)
require.Equal(t, "alice@example.com", got.Email)
```

**When to use each:**

| Package | Behavior | Use when |
|---|---|---|
| `assert` | non-fatal, test continues | checking multiple independent fields |
| `require` | fatal, stops immediately | precondition must hold for rest of test to make sense |

`require.NoError(t, err)` before accessing the result is the most common pattern — no point checking `got.Email` if `err != nil` crashed the response.

**Trade-off:** testify adds a dependency but significantly reduces assertion noise in large test suites. The standard library is always available with zero dependencies.
</file>

<file path="skills/golang-gin-testing/SKILL.md">
---
name: golang-gin-testing
description: "Test Go Gin REST APIs with unit, integration, and end-to-end tests. Covers httptest patterns, table-driven tests, mocking repositories, testcontainers for real databases, and CI/CD integration. Use when writing tests for Gin handlers, services, middleware, or setting up test infrastructure for a Go API."
license: MIT
metadata:
  author: henriqueatila
  version: 1.0.3
---

# golang-gin-testing — Testing REST APIs

Write confident tests for Gin APIs: unit tests with mocked repositories, integration tests with real PostgreSQL via testcontainers, and e2e tests for critical flows. This skill covers the 80% of testing patterns you need daily.

## When to Use

- Writing tests for Gin handlers (`UserHandler`, `AuthHandler`)
- Testing services with a mocked `UserRepository`
- Setting up integration tests with a real database (testcontainers)
- Testing JWT auth middleware in isolation
- Adding table-driven tests for request validation and error paths
- Setting up `TestMain` for shared test infrastructure

## Table of Contents

1. [Testing Philosophy](#testing-philosophy)
2. [Test Helpers](#test-helpers)
3. [Handler Tests with httptest](#handler-tests-with-httptest)
4. [Table-Driven Handler Tests](#table-driven-handler-tests)
5. [Service Tests with Mocked Repository](#service-tests-with-mocked-repository)
6. [Running Tests](#running-tests)
7. [Reference Files](#reference-files)
8. [Cross-Skill References](#cross-skill-references)

---

## Testing Philosophy

| Layer | Tool | Goal |
|---|---|---|
| Handler | `httptest` + mock service | Verify HTTP contract (status codes, JSON shape) |
| Service | mock repository | Verify business logic, error mapping |
| Repository | testcontainers (real DB) | Verify SQL correctness |
| E2E | running server + real DB | Verify critical user flows end-to-end |

Unit tests run fast and cover most cases. Integration tests verify real database behavior. E2e tests catch wiring bugs. **Never mock what you're testing** — mock the layer below.

---

## Test Helpers

Define reusable helpers in `internal/testutil/` to keep tests DRY.

```go
// internal/testutil/helpers.go
package testutil

import (
    "bytes"
    "encoding/json"
    "net/http"
    "net/http/httptest"
    "testing"

    "github.com/gin-gonic/gin"
)

func init() {
    // Suppress Gin debug output in tests
    gin.SetMode(gin.TestMode)
}

// NewTestRouter creates a bare Gin engine for tests (no Logger/Recovery noise).
func NewTestRouter() *gin.Engine {
    return gin.New()
}

// PerformRequest executes an HTTP request against the given router and returns the recorder.
func PerformRequest(t *testing.T, router *gin.Engine, method, path string, body any, headers map[string]string) *httptest.ResponseRecorder {
    t.Helper()

    var reqBody *bytes.Buffer
    if body != nil {
        b, err := json.Marshal(body)
        if err != nil {
            t.Fatalf("PerformRequest: failed to marshal body: %v", err)
        }
        reqBody = bytes.NewBuffer(b)
    } else {
        reqBody = bytes.NewBuffer(nil)
    }

    req, err := http.NewRequest(method, path, reqBody)
    if err != nil {
        t.Fatalf("PerformRequest: failed to create request: %v", err)
    }

    if body != nil {
        req.Header.Set("Content-Type", "application/json")
    }
    for k, v := range headers {
        req.Header.Set(k, v)
    }

    w := httptest.NewRecorder()
    router.ServeHTTP(w, req)
    return w
}

// AssertJSON unmarshals the recorder body into dst, failing the test on error.
func AssertJSON(t *testing.T, w *httptest.ResponseRecorder, dst any) {
    t.Helper()
    if err := json.Unmarshal(w.Body.Bytes(), dst); err != nil {
        t.Fatalf("AssertJSON: failed to unmarshal %q: %v", w.Body.String(), err)
    }
}

// BearerHeader returns an Authorization header map for JWT-protected routes.
func BearerHeader(token string) map[string]string {
    return map[string]string{"Authorization": "Bearer " + token}
}
```

---

## Handler Tests with httptest

Test handlers by wiring a real router with a **mock service**, then using `httptest.NewRecorder` + `router.ServeHTTP`.

```go
// internal/handler/user_handler_test.go
package handler_test

import (
    "context"
    "log/slog"
    "net/http"
    "testing"
    "time"

    "github.com/gin-gonic/gin"
    "myapp/internal/domain"
    "myapp/internal/handler"
    "myapp/internal/service"
    "myapp/internal/testutil"
)

// mockUserService implements service.UserService for tests.
type mockUserService struct {
    createFn  func(ctx context.Context, req domain.CreateUserRequest) (*domain.User, error)
    getByIDFn func(ctx context.Context, id string) (*domain.User, error)
}

func (m *mockUserService) Create(ctx context.Context, req domain.CreateUserRequest) (*domain.User, error) {
    return m.createFn(ctx, req)
}

func (m *mockUserService) GetByID(ctx context.Context, id string) (*domain.User, error) {
    return m.getByIDFn(ctx, id)
}

func setupUserRouter(svc service.UserService) *gin.Engine {
    r := testutil.NewTestRouter()
    h := handler.NewUserHandler(svc, slog.Default())
    r.POST("/users", h.Create)
    r.GET("/users/:id", h.GetByID)
    return r
}

func TestUserHandler_Create_Success(t *testing.T) {
    now := time.Now()
    svc := &mockUserService{
        createFn: func(ctx context.Context, req domain.CreateUserRequest) (*domain.User, error) {
            return &domain.User{
                ID:        "user-123",
                Name:      req.Name,
                Email:     req.Email,
                Role:      "user",
                CreatedAt: now,
                UpdatedAt: now,
            }, nil
        },
    }

    router := setupUserRouter(svc)
    body := map[string]any{
        "name":     "Alice",
        "email":    "alice@example.com",
        "password": "secret123",
    }

    w := testutil.PerformRequest(t, router, http.MethodPost, "/users", body, nil)

    if w.Code != http.StatusCreated {
        t.Errorf("expected status 201, got %d; body: %s", w.Code, w.Body)
    }

    var got domain.User
    testutil.AssertJSON(t, w, &got)
    if got.ID != "user-123" {
        t.Errorf("expected user ID 'user-123', got %q", got.ID)
    }
}

func TestUserHandler_GetByID_NotFound(t *testing.T) {
    svc := &mockUserService{
        getByIDFn: func(ctx context.Context, id string) (*domain.User, error) {
            return nil, domain.ErrNotFound
        },
    }

    router := setupUserRouter(svc)
    w := testutil.PerformRequest(t, router, http.MethodGet, "/users/missing-id", nil, nil)

    if w.Code != http.StatusNotFound {
        t.Errorf("expected status 404, got %d", w.Code)
    }
}
```

---

## Table-Driven Handler Tests

Table-driven tests cover all request variants (valid, invalid, edge cases) in one function.

```go
// internal/handler/user_handler_table_test.go
package handler_test

import (
    "context"
    "net/http"
    "testing"

    "myapp/internal/domain"
    "myapp/internal/testutil"
)

func TestUserHandler_Create_Validation(t *testing.T) {
    svc := &mockUserService{
        createFn: func(ctx context.Context, req domain.CreateUserRequest) (*domain.User, error) {
            return &domain.User{ID: "1", Name: req.Name, Email: req.Email, Role: "user"}, nil
        },
    }
    router := setupUserRouter(svc)

    tests := []struct {
        name       string
        body       any
        wantStatus int
    }{
        {
            name:       "valid request",
            body:       map[string]any{"name": "Alice", "email": "alice@example.com", "password": "secret123"},
            wantStatus: http.StatusCreated,
        },
        {
            name:       "missing email",
            body:       map[string]any{"name": "Alice", "password": "secret123"},
            wantStatus: http.StatusBadRequest,
        },
        {
            name:       "invalid email format",
            body:       map[string]any{"name": "Alice", "email": "not-an-email", "password": "secret123"},
            wantStatus: http.StatusBadRequest,
        },
        {
            name:       "name too short",
            body:       map[string]any{"name": "A", "email": "alice@example.com", "password": "secret123"},
            wantStatus: http.StatusBadRequest,
        },
        {
            name:       "password too short",
            body:       map[string]any{"name": "Alice", "email": "alice@example.com", "password": "short"},
            wantStatus: http.StatusBadRequest,
        },
        {
            name:       "invalid role value",
            body:       map[string]any{"name": "Alice", "email": "alice@example.com", "password": "secret123", "role": "superadmin"},
            wantStatus: http.StatusBadRequest,
        },
    }

    for _, tc := range tests {
        t.Run(tc.name, func(t *testing.T) {
            t.Parallel()
            w := testutil.PerformRequest(t, router, http.MethodPost, "/users", tc.body, nil)
            if w.Code != tc.wantStatus {
                t.Errorf("want status %d, got %d; body: %s", tc.wantStatus, w.Code, w.Body)
            }
        })
    }
}
```

---

## Service Tests with Mocked Repository

Test business logic without touching the database. The mock implements `domain.UserRepository`.

```go
// internal/service/user_service_test.go
package service_test

import (
    "context"
    "errors"
    "log/slog"
    "testing"

    "myapp/internal/domain"
    "myapp/internal/service"
)

// mockUserRepository implements domain.UserRepository for service tests.
type mockUserRepository struct {
    createFn     func(ctx context.Context, user *domain.User) error
    getByEmailFn func(ctx context.Context, email string) (*domain.User, error)
    getByIDFn    func(ctx context.Context, id string) (*domain.User, error)
}

func (m *mockUserRepository) Create(ctx context.Context, user *domain.User) error {
    return m.createFn(ctx, user)
}
func (m *mockUserRepository) GetByEmail(ctx context.Context, email string) (*domain.User, error) {
    return m.getByEmailFn(ctx, email)
}
func (m *mockUserRepository) GetByID(ctx context.Context, id string) (*domain.User, error) {
    if m.getByIDFn != nil {
        return m.getByIDFn(ctx, id)
    }
    return nil, domain.ErrNotFound
}
func (m *mockUserRepository) List(ctx context.Context, opts domain.ListOptions) ([]domain.User, int64, error) {
    return nil, 0, nil
}
func (m *mockUserRepository) Update(ctx context.Context, user *domain.User) error { return nil }
func (m *mockUserRepository) Delete(ctx context.Context, id string) error         { return nil }

func TestUserService_Create_DuplicateEmail(t *testing.T) {
    repo := &mockUserRepository{
        getByEmailFn: func(ctx context.Context, email string) (*domain.User, error) {
            return &domain.User{Email: email}, nil // email already taken
        },
        createFn: func(ctx context.Context, user *domain.User) error {
            return nil
        },
    }

    svc := service.NewUserService(repo, slog.Default())
    _, err := svc.Create(context.Background(), domain.CreateUserRequest{
        Name:     "Alice",
        Email:    "alice@example.com",
        Password: "secret123",
    })

    // Use errors.As to unwrap *AppError and inspect the HTTP status code.
    // errors.Is works only if AppError implements Is(); errors.As is always safe.
    var appErr *domain.AppError
    if !errors.As(err, &appErr) || appErr.Code != 409 {
        t.Errorf("expected ErrConflict (409 AppError), got %v", err)
    }
}
```

---

## Running Tests

```bash
# All tests with race detector and coverage
go test -v -race -cover ./...

# Specific package
go test -v -race ./internal/handler/...

# Run only unit tests (exclude integration)
go test -v -race -cover -tags='!integration' ./...

# Run integration tests only (requires Docker)
go test -v -race -tags=integration ./internal/repository/...

# Coverage report
go test -race -coverprofile=coverage.out ./...
go tool cover -html=coverage.out
```

---

## Reference Files

Load these when you need deeper detail:

- **[references/unit-tests.md](references/unit-tests.md)** — Handler httptest patterns, testing authenticated routes with mock JWT, middleware isolation tests, mock generation with `gomock`/manual mocks, `t.Helper`/`t.Cleanup`/`t.Parallel`, test fixtures and factories, benchmark tests (`BenchmarkX`), fuzz tests (`FuzzX`), golden file/snapshot testing, test organization (same-package vs external-package, build tags), testify assertions
- **[references/integration-tests.md](references/integration-tests.md)** — testcontainers-go setup, `TestMain` for DB lifecycle, repository integration tests, cleanup between tests, build tags, fixture loading
- **[references/e2e.md](references/e2e.md)** — End-to-end flow testing (register → login → CRUD), docker-compose test setup, GitHub Actions CI/CD, environment configuration, cleanup and idempotency

## Cross-Skill References

- For handler and service implementations being tested: see the **golang-gin-api** skill
- For `UserRepository` interface and GORM/sqlx implementations: see the **golang-gin-database** skill
- For JWT middleware and auth handler test patterns: see the **golang-gin-auth** skill
- **golang-gin-clean-arch** → Architecture: mock strategy (boundaries only), testing by layer, test fixtures

## Official Docs

If this skill doesn't cover your use case, consult the [Go testing package](https://pkg.go.dev/testing), [httptest GoDoc](https://pkg.go.dev/net/http/httptest), or [testcontainers-go docs](https://golang.testcontainers.org/).
</file>

<file path="skills/golang-gin-auth/references/jwt-patterns.md">
# jwt-patterns.md — JWT Token Architecture

Complete reference for JWT access/refresh token patterns in Gin APIs. Covers token lifecycle, refresh endpoint, blacklisting, storage, and algorithm choices.

All patterns use `github.com/golang-jwt/jwt/v5`. These are architectural recommendations, not Gin API.

## Table of Contents

1. [Access + Refresh Token Architecture](#access--refresh-token-architecture)
2. [Custom Claims](#custom-claims)
3. [Token Refresh Endpoint](#token-refresh-endpoint)
4. [Token Blacklisting (Redis)](#token-blacklisting-redis)
5. [Storage Recommendations](#storage-recommendations)
6. [CSRF Protection](#csrf-protection)
7. [RS256 vs HS256](#rs256-vs-hs256)
8. [Complete Auth Flow Example](#complete-auth-flow-example)

---

## Access + Refresh Token Architecture

Two-token strategy separates concerns: short-lived access tokens reduce attack surface; long-lived refresh tokens enable session continuity without re-login.

```
┌──────────┐   POST /auth/login    ┌───────────┐
│  Client  │──────────────────────▶│  Gin API  │
│          │◀──────────────────────│           │
│          │  {access, refresh}    └───────────┘
│          │
│          │   GET /api/v1/users   ┌───────────┐
│          │   Authorization: Bearer <access>
│          │──────────────────────▶│  Gin API  │
│          │◀──────────────────────│           │
│          │   200 OK              └───────────┘
│          │
│          │   POST /auth/refresh  ┌───────────┐
│          │   {refresh_token}     │           │
│          │──────────────────────▶│  Gin API  │
│          │◀──────────────────────│           │
│          │   {new_access}        └───────────┘
└──────────┘
```

| Token         | TTL           | Payload         | Stored where       |
|---------------|---------------|-----------------|--------------------|
| Access token  | 15 minutes    | UserID, Role    | Memory / header    |
| Refresh token | 7–30 days     | UserID only     | httpOnly cookie     |

**Why short-lived access tokens?** If stolen, they expire quickly. The refresh token grants new access tokens — revoking a refresh token immediately locks out the session.

---

## Custom Claims

Embed `jwt.RegisteredClaims` for standard fields. Add only what handlers need — avoid putting large objects in the token.

```go
// internal/auth/claims.go
package auth

import "github.com/golang-jwt/jwt/v5"

// Claims is the JWT payload for access tokens.
type Claims struct {
    jwt.RegisteredClaims            // sub, iat, exp, iss, aud
    UserID   string   `json:"uid"`
    Email    string   `json:"email"`
    Role     string   `json:"role"`
    // For multi-role: Roles    []string `json:"roles"`
    // For multi-tenant: TenantID string `json:"tid"`
}

// RefreshClaims is the minimal payload for refresh tokens.
// Only the subject (UserID) is needed — refresh doesn't need role/email.
type RefreshClaims struct {
    jwt.RegisteredClaims
    TokenID string `json:"jti"` // for blacklisting: store this ID in Redis on logout
}
```

**Why separate RefreshClaims?** Refresh tokens are long-lived — embedding extra data increases exposure. The `jti` (JWT ID) enables per-token revocation without invalidating all sessions.

---

## Token Refresh Endpoint

> **Security note:** In production, never expose raw `err.Error()` to clients. Return generic messages and log the error server-side. See **golang-gin-clean-arch** error handling patterns.

Exchange a valid refresh token for a new access token. Two rotation strategies exist — pick one and document your choice:

- **Option A: Always rotate** — new refresh token on every call. More secure (old token invalidated immediately) but breaks parallel requests that use the same refresh token.
- **Option B: Rotate past half-lifetime (recommended)** — issue a new refresh token only when the current one is more than halfway through its lifetime. Prevents logout on parallel requests; the window of token reuse is bounded.

The code below implements **Option B**.

```go
// internal/handler/auth_handler.go (Refresh method)
package handler

import (
    "fmt"
    "log/slog"
    "net/http"
    "time"

    "github.com/gin-gonic/gin"
    "github.com/golang-jwt/jwt/v5"
    "myapp/internal/auth"
    "myapp/internal/domain"
)

type refreshRequest struct {
    RefreshToken string `json:"refresh_token" binding:"required"`
}

func (h *AuthHandler) Refresh(c *gin.Context) {
    var req refreshRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "invalid request body"})
        return
    }

    // Parse refresh token with its own secret
    token, err := jwt.ParseWithClaims(req.RefreshToken, &auth.RefreshClaims{}, func(t *jwt.Token) (any, error) {
        if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
            return nil, fmt.Errorf("unexpected signing method: %v", t.Header["alg"])
        }
        return h.tokenCfg.RefreshSecret, nil
    })
    if err != nil {
        c.JSON(http.StatusUnauthorized, gin.H{"error": "invalid or expired refresh token"})
        return
    }

    rc, ok := token.Claims.(*auth.RefreshClaims)
    if !ok || !token.Valid {
        c.JSON(http.StatusUnauthorized, gin.H{"error": "invalid refresh token claims"})
        return
    }

    // Re-fetch user to ensure they are still active
    user, err := h.userRepo.GetByID(c.Request.Context(), rc.Subject)
    if err != nil {
        h.logger.Warn("refresh: user lookup failed", "error", err, "subject", rc.Subject)
        c.JSON(http.StatusUnauthorized, gin.H{"error": "invalid or expired refresh token"})
        return
    }

    accessToken, err := auth.GenerateAccessToken(h.tokenCfg, user.ID, user.Email, user.Role)
    if err != nil {
        h.logger.Error("failed to generate access token on refresh", "error", err, "user_id", user.ID)
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
        return
    }

    // Option B: Rotate only when the refresh token is past half its lifetime.
    // This avoids logout on parallel requests while keeping the reuse window bounded.
    // Switch to always-rotate (Option A) if you can guarantee sequential refresh calls.
    resp := gin.H{
        "access_token": accessToken,
        "expires_in":   int(h.tokenCfg.AccessTTL / time.Second),
    }
    halfLife := rc.IssuedAt.Time.Add(h.tokenCfg.RefreshTTL / 2)
    if time.Now().After(halfLife) {
        newRefresh, err := auth.GenerateRefreshToken(h.tokenCfg, user.ID)
        if err != nil {
            h.logger.Error("failed to rotate refresh token", "error", err, "user_id", user.ID)
            c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
            return
        }
        resp["refresh_token"] = newRefresh
    }

    c.JSON(http.StatusOK, resp)
}
```

---

## Token Blacklisting (Redis)

Tokens are stateless — you can't "delete" one. Blacklisting stores revoked token IDs in Redis with TTL matching the token's remaining lifetime. On every request, the middleware checks Redis before accepting the token.

```go
// internal/auth/blacklist.go
package auth

import (
    "context"
    "fmt"
    "time"

    "github.com/redis/go-redis/v9"
)

// Blacklist manages token revocation via Redis.
// Architectural recommendation — not part of the Gin API.
type Blacklist struct {
    rdb    *redis.Client
    prefix string
}

func NewBlacklist(rdb *redis.Client) *Blacklist {
    return &Blacklist{rdb: rdb, prefix: "jwt:revoked:"}
}

// Revoke adds a token ID to the blacklist until its expiry.
func (b *Blacklist) Revoke(ctx context.Context, tokenID string, expiry time.Time) error {
    ttl := time.Until(expiry)
    if ttl <= 0 {
        return nil // already expired — no need to blacklist
    }
    key := b.prefix + tokenID
    if err := b.rdb.Set(ctx, key, "1", ttl).Err(); err != nil {
        return fmt.Errorf("blacklist.Revoke: %w", err)
    }
    return nil
}

// IsRevoked returns true if the token ID has been revoked.
func (b *Blacklist) IsRevoked(ctx context.Context, tokenID string) (bool, error) {
    key := b.prefix + tokenID
    val, err := b.rdb.Exists(ctx, key).Result()
    if err != nil {
        return false, fmt.Errorf("blacklist.IsRevoked: %w", err)
    }
    return val > 0, nil
}
```

**Middleware integration:** After `ParseAccessToken`, check `blacklist.IsRevoked(ctx, claims.ID)` before calling `c.Next()`. The `claims.ID` maps to the `jti` (JWT ID) field in `RegisteredClaims`.

**Logout endpoint pattern:**
```go
func (h *AuthHandler) Logout(c *gin.Context) {
    val, exists := c.Get(middleware.ClaimsKey)
    if !exists {
        c.JSON(http.StatusUnauthorized, gin.H{"error": "unauthorized"})
        return
    }
    claims, ok := val.(*auth.Claims)
    if !ok {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
        return
    }

    expiry := claims.ExpiresAt.Time
    if err := h.blacklist.Revoke(c.Request.Context(), claims.ID, expiry); err != nil {
        h.logger.Error("failed to revoke token", "error", err, "jti", claims.ID)
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
        return
    }

    c.JSON(http.StatusOK, gin.H{"message": "logged out"})
}
```

---

## Storage Recommendations

Where the client stores tokens affects the attack surface:

| Storage          | XSS Risk | CSRF Risk | Notes                                     |
|------------------|----------|-----------|-------------------------------------------|
| httpOnly cookie  | Low      | Medium    | Use `SameSite=Strict` + CSRF token to mitigate |
| localStorage     | High     | Low       | JavaScript-accessible — XSS can steal tokens |
| sessionStorage   | High     | Low       | Same as localStorage, cleared on tab close |
| Memory (JS var)  | Low      | Low       | Lost on refresh — needs silent refresh flow |

**Recommended:** httpOnly, Secure cookie for refresh token; short-lived access token in memory or Authorization header.

Setting a secure cookie in Gin — use `http.SetCookie` directly to set `SameSite`, which `c.SetCookie` does not expose:
```go
// After successful login, set refresh token as httpOnly cookie
http.SetCookie(c.Writer, &http.Cookie{
    Name:     "refresh_token",
    Value:    refreshToken,
    Path:     "/auth",             // restrict to /auth/* endpoints
    HttpOnly: true,                // not accessible by JavaScript
    Secure:   true,                // HTTPS only
    SameSite: http.SameSiteStrictMode, // CSRF mitigation
    MaxAge:   7 * 24 * 3600,      // 7 days in seconds
})
```

**Reading the cookie in the refresh endpoint:**
```go
refreshToken, err := c.Cookie("refresh_token")
if err != nil {
    c.JSON(http.StatusUnauthorized, gin.H{"error": "refresh token not found"})
    return
}
```

---

## CSRF Protection

When using httpOnly cookies for the refresh token, protect state-mutating endpoints from cross-site request forgery. The double-submit cookie pattern: set a non-httpOnly CSRF token cookie on login, require it in a custom header on every mutating request.

```go
// pkg/middleware/csrf.go
package middleware

import (
    "crypto/hmac"
    "net/http"

    "github.com/gin-gonic/gin"
)

// CSRFProtection implements the double-submit cookie pattern.
// Set a "csrf_token" cookie on login (non-httpOnly so JS can read it).
// The client must echo it back in the "X-CSRF-Token" header on every mutating request.
func CSRFProtection() gin.HandlerFunc {
    return func(c *gin.Context) {
        method := c.Request.Method
        // Only check state-mutating methods
        if method == http.MethodGet || method == http.MethodHead || method == http.MethodOptions {
            c.Next()
            return
        }

        csrfHeader := c.GetHeader("X-CSRF-Token")
        cookieToken, err := c.Cookie("csrf_token")
        if err != nil || csrfHeader == "" || !hmac.Equal([]byte(csrfHeader), []byte(cookieToken)) {
            c.AbortWithStatusJSON(http.StatusForbidden, gin.H{"error": "invalid CSRF token"})
            return
        }
        c.Next()
    }
}
```

**Set the CSRF cookie on login** (non-httpOnly so JavaScript can read it):

```go
// internal/handler/auth_handler.go — after successful login
import (
    "crypto/rand"
    "encoding/hex"
    "net/http"
)

func generateCSRFToken() (string, error) {
    b := make([]byte, 32)
    if _, err := rand.Read(b); err != nil {
        return "", err
    }
    return hex.EncodeToString(b), nil
}

// Inside Login, after issuing tokens:
csrfToken, err := generateCSRFToken()
if err != nil {
    h.logger.Error("failed to generate CSRF token", "error", err)
    c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
    return
}
http.SetCookie(c.Writer, &http.Cookie{
    Name:     "csrf_token",
    Value:    csrfToken,
    Path:     "/",
    Secure:   true,
    HttpOnly: false, // must be readable by JS to send as header
    SameSite: http.SameSiteStrictMode,
    MaxAge:   7 * 24 * 3600,
})
```

**Apply the middleware** on all mutating routes:

```go
api := r.Group("/api/v1")
api.Use(middleware.CSRFProtection())
```

> **Note:** CSRF protection is only necessary when using cookies for auth. If you use `Authorization: Bearer` headers exclusively, CSRF is not a risk — browsers do not send custom headers cross-origin without explicit CORS preflight.

---

## RS256 vs HS256

| Aspect          | HS256 (HMAC-SHA256)         | RS256 (RSA-SHA256)                  |
|-----------------|-----------------------------|-------------------------------------|
| Keys            | One shared secret           | Private key (sign) + Public key (verify) |
| Verification    | Needs the secret            | Needs only the public key           |
| Use case        | Single service / monolith   | Microservices, third-party verification |
| Key management  | Simple                      | More complex (cert rotation)        |

**HS256 setup** (shown in SKILL.md) — suitable for most single-service APIs.

**RS256 setup** — use when multiple services or external clients need to verify tokens without the signing secret:

```go
// internal/auth/token_rsa.go
package auth

import (
    "crypto/rsa"
    "fmt"
    "time"

    "github.com/golang-jwt/jwt/v5"
    "github.com/google/uuid"
)

// GenerateAccessTokenRS256 signs with RSA private key.
// Accepts issuer and audience to match the HS256 variant — required for token blacklisting (jti) and multi-service verification.
func GenerateAccessTokenRS256(privateKey *rsa.PrivateKey, userID, email, role, issuer string, audience []string, ttl time.Duration) (string, error) {
    now := time.Now()
    claims := Claims{
        RegisteredClaims: jwt.RegisteredClaims{
            ID:        uuid.NewString(),              // jti — required for token blacklisting
            Subject:   userID,
            Issuer:    issuer,
            Audience:  jwt.ClaimStrings(audience),
            IssuedAt:  jwt.NewNumericDate(now),
            NotBefore: jwt.NewNumericDate(now),
            ExpiresAt: jwt.NewNumericDate(now.Add(ttl)),
        },
        UserID: userID,
        Email:  email,
        Role:   role,
    }
    token := jwt.NewWithClaims(jwt.SigningMethodRS256, claims)
    signed, err := token.SignedString(privateKey)
    if err != nil {
        return "", fmt.Errorf("sign RS256 token: %w", err)
    }
    return signed, nil
}

// ParseAccessTokenRS256 verifies with RSA public key (safe to distribute).
func ParseAccessTokenRS256(publicKey *rsa.PublicKey, tokenStr string) (*Claims, error) {
    token, err := jwt.ParseWithClaims(tokenStr, &Claims{}, func(t *jwt.Token) (any, error) {
        if _, ok := t.Method.(*jwt.SigningMethodRSA); !ok {
            return nil, fmt.Errorf("unexpected signing method: %v", t.Header["alg"])
        }
        return publicKey, nil
    })
    if err != nil {
        return nil, fmt.Errorf("parse RS256 token: %w", err)
    }
    claims, ok := token.Claims.(*Claims)
    if !ok || !token.Valid {
        return nil, fmt.Errorf("invalid token claims")
    }
    return claims, nil
}
```

Load RSA keys from PEM files or environment at startup — never embed keys in source code.

---

## Complete Auth Flow Example

End-to-end wiring of auth routes with all patterns:

```go
// cmd/api/main.go — auth wiring
package main

import (
    "context"
    "log/slog"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"

    "github.com/gin-gonic/gin"
    "github.com/redis/go-redis/v9"
    "myapp/internal/auth"
    "myapp/internal/handler"
    "myapp/internal/repository"
    "myapp/internal/service"
    "myapp/pkg/middleware"
)

func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

    // Load config from environment — never hardcode secrets
    tokenCfg := auth.TokenConfig{
        AccessSecret:  []byte(os.Getenv("JWT_ACCESS_SECRET")),
        RefreshSecret: []byte(os.Getenv("JWT_REFRESH_SECRET")),
        AccessTTL:     15 * time.Minute,
        RefreshTTL:    7 * 24 * time.Hour,
        Issuer:        os.Getenv("JWT_ISSUER"),   // e.g. "myapp"
        Audience:      []string{os.Getenv("JWT_AUDIENCE")}, // e.g. ["api.myapp.com"]
    }

    // Redis for token blacklisting
    rdb := redis.NewClient(&redis.Options{Addr: os.Getenv("REDIS_ADDR")})
    blacklist := auth.NewBlacklist(rdb)

    // DB and repository wiring (see gin-database skill)
    db, err := repository.NewGORMDB(repository.Config{DSN: os.Getenv("DATABASE_URL")}, logger)
    if err != nil {
        logger.Error("database connection failed", "error", err)
        os.Exit(1)
    }
    userRepo := repository.NewUserRepository(db)

    // Handlers
    authHandler := handler.NewAuthHandler(userRepo, tokenCfg, blacklist, logger)
    userSvc := service.NewUserService(userRepo, logger)
    userHandler := handler.NewUserHandler(userSvc, logger)

    r := gin.New()
    r.Use(middleware.Logger(logger))
    r.Use(middleware.Recovery(logger))

    api := r.Group("/api/v1")

    // Public
    api.POST("/auth/login", authHandler.Login)
    api.POST("/auth/refresh", authHandler.Refresh)
    api.POST("/users", userHandler.Create)

    // Protected — Auth middleware validates JWT and injects claims
    protected := api.Group("")
    protected.Use(middleware.Auth(tokenCfg, logger))
    {
        protected.POST("/auth/logout", authHandler.Logout)
        protected.GET("/users/me", userHandler.GetMe)
        protected.GET("/users/:id", userHandler.GetByID)

        // Admin only
        admin := protected.Group("/admin")
        admin.Use(middleware.RequireRole("admin"))
        {
            admin.GET("/users", userHandler.List)
            admin.DELETE("/users/:id", userHandler.Delete)
        }
    }

    srv := &http.Server{
        Addr:         os.Getenv("PORT"),
        Handler:      r,
        ReadTimeout:  10 * time.Second,
        WriteTimeout: 10 * time.Second,
    }
    go func() {
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            logger.Error("server failed", "error", err)
            os.Exit(1)
        }
    }()
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()
    if err := srv.Shutdown(ctx); err != nil {
        logger.Error("graceful shutdown failed", "error", err)
    }
}
```

**Sequence for a protected request:**

1. Client sends `Authorization: Bearer <access_token>`
2. `Auth` middleware extracts and validates the token
3. Middleware checks `blacklist.IsRevoked(ctx, claims.ID)` → 401 if revoked
4. Claims injected via `c.Set(ClaimsKey, claims)` and `c.Set(UserIDKey, claims.UserID)`
5. Handler calls `c.GetString(UserIDKey)` or `c.Get(ClaimsKey)` to read identity
6. Handler passes `c.Request.Context()` to all downstream service/repository calls
</file>

<file path="skills/golang-gin-auth/SKILL.md">
---
name: golang-gin-auth
description: "Implement authentication and authorization in Go Gin APIs. Covers JWT middleware, login/register handlers, role-based access control (RBAC), token refresh, and protected routes. Use when adding auth, login, signup, JWT tokens, user sessions, permissions, or role checks to a Gin application."
license: MIT
metadata:
  author: henriqueatila
  version: 1.0.4
---

# golang-gin-auth — Authentication & Authorization

Add JWT-based authentication and role-based access control to a Gin API. This skill covers the patterns you need for secure APIs: JWT middleware, login handler, token lifecycle, and RBAC.

## When to Use

- Adding JWT authentication to a Gin API
- Implementing login or registration endpoints
- Protecting routes with middleware
- Implementing role-based or permission-based access control (RBAC)
- Handling token refresh and revocation
- Getting the current user in any handler

## Dependencies

```bash
go get github.com/golang-jwt/jwt/v5
go get golang.org/x/crypto
go get github.com/google/uuid
go get golang.org/x/time/rate
```

## Claims Struct

Define custom JWT claims that embed `jwt.RegisteredClaims` (architectural recommendation — uses `github.com/golang-jwt/jwt/v5`):

```go
// internal/auth/claims.go
package auth

import "github.com/golang-jwt/jwt/v5"

// Claims holds the JWT payload for access tokens. Embed RegisteredClaims for standard fields.
// RegisteredClaims.ID carries the jti — required for token blacklisting.
type Claims struct {
    jwt.RegisteredClaims
    UserID string `json:"uid"`
    Email  string `json:"email"`
    Role   string `json:"role"`
    // Add Roles []string for multi-role support, TenantID for multi-tenancy
}

// RefreshClaims is the minimal payload for refresh tokens.
// Only the subject (UserID) is needed — refresh doesn't need role/email.
type RefreshClaims struct {
    jwt.RegisteredClaims
    // RegisteredClaims.ID carries the jti for per-token blacklisting.
}
```

## Token Generation and Validation

```go
// internal/auth/token.go
package auth

import (
    "errors"
    "fmt"
    "time"

    "github.com/golang-jwt/jwt/v5"
    "github.com/google/uuid"
)

// TokenConfig holds secrets and TTLs. Load from environment, never hardcode.
type TokenConfig struct {
    AccessSecret  []byte
    RefreshSecret []byte
    AccessTTL     time.Duration // e.g. 15 * time.Minute
    RefreshTTL    time.Duration // e.g. 7 * 24 * time.Hour
    Issuer        string        // e.g. "myapp"
    Audience      []string      // e.g. ["api.myapp.com"]
}

// GenerateAccessToken creates a signed JWT access token for the given user.
// Architectural recommendation — not part of the Gin API.
func GenerateAccessToken(cfg TokenConfig, userID, email, role string) (string, error) {
    now := time.Now()
    claims := Claims{
        RegisteredClaims: jwt.RegisteredClaims{
            ID:        uuid.NewString(), // jti — required for token blacklisting
            Subject:   userID,
            Issuer:    cfg.Issuer,
            Audience:  jwt.ClaimStrings(cfg.Audience),
            IssuedAt:  jwt.NewNumericDate(now),
            NotBefore: jwt.NewNumericDate(now), // token not valid before issue time
            ExpiresAt: jwt.NewNumericDate(now.Add(cfg.AccessTTL)),
        },
        UserID: userID,
        Email:  email,
        Role:   role,
    }
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    signed, err := token.SignedString(cfg.AccessSecret)
    if err != nil {
        return "", fmt.Errorf("sign access token: %w", err)
    }
    return signed, nil
}

// GenerateRefreshToken creates a longer-lived refresh token (user ID only).
func GenerateRefreshToken(cfg TokenConfig, userID string) (string, error) {
    now := time.Now()
    claims := RefreshClaims{
        RegisteredClaims: jwt.RegisteredClaims{
            ID:        uuid.NewString(), // jti — required for per-token blacklisting
            Subject:   userID,
            Issuer:    cfg.Issuer,
            IssuedAt:  jwt.NewNumericDate(now),
            NotBefore: jwt.NewNumericDate(now),
            ExpiresAt: jwt.NewNumericDate(now.Add(cfg.RefreshTTL)),
        },
    }
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    signed, err := token.SignedString(cfg.RefreshSecret)
    if err != nil {
        return "", fmt.Errorf("sign refresh token: %w", err)
    }
    return signed, nil
}

// ParseAccessToken validates and parses a JWT access token.
func ParseAccessToken(cfg TokenConfig, tokenStr string) (*Claims, error) {
    token, err := jwt.ParseWithClaims(tokenStr, &Claims{}, func(t *jwt.Token) (any, error) {
        if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
            return nil, fmt.Errorf("unexpected signing method: %v", t.Header["alg"])
        }
        return cfg.AccessSecret, nil
    })
    if err != nil {
        if errors.Is(err, jwt.ErrTokenExpired) {
            return nil, fmt.Errorf("token expired: %w", err)
        }
        return nil, fmt.Errorf("invalid token: %w", err)
    }
    claims, ok := token.Claims.(*Claims)
    if !ok || !token.Valid {
        return nil, fmt.Errorf("invalid token claims")
    }
    return claims, nil
}
```

## JWT Middleware

Extracts the Bearer token, validates it, and injects claims into `gin.Context` via `c.Set`. Handlers read claims via `c.Get`.

```go
// pkg/middleware/auth.go
package middleware

import (
    "log/slog"
    "net/http"
    "strings"

    "github.com/gin-gonic/gin"
    "myapp/internal/auth"
)

const (
    ClaimsKey = "claims"   // key used for c.Set / c.Get
    UserIDKey  = "user_id" // convenience string key for c.GetString
)

// Auth returns a Gin middleware that validates JWT Bearer tokens.
// Aborts with 401 if the token is missing, malformed, or expired.
func Auth(cfg auth.TokenConfig, logger *slog.Logger) gin.HandlerFunc {
    return func(c *gin.Context) {
        header := c.GetHeader("Authorization")
        if header == "" {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "authorization header required"})
            return
        }

        parts := strings.SplitN(header, " ", 2)
        if len(parts) != 2 || !strings.EqualFold(parts[0], "Bearer") {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "invalid authorization header format"})
            return
        }

        claims, err := auth.ParseAccessToken(cfg, parts[1])
        if err != nil {
            logger.Warn("jwt validation failed", "error", err)
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "invalid or expired token"})
            return
        }

        // Inject claims and convenience string keys for type-safe access
        c.Set(ClaimsKey, claims)
        c.Set(UserIDKey, claims.UserID)
        c.Next()
    }
}
```

## Registration — Password Hashing

Always hash passwords with bcrypt before storing. Use cost >= 12 for production.

```go
// internal/handler/auth_handler.go (Register method)

type registerRequest struct {
    Email    string `json:"email"    binding:"required,email"`
    Password string `json:"password" binding:"required,min=8"`
}

func (h *AuthHandler) Register(c *gin.Context) {
    var req registerRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "invalid request body"})
        return
    }

    // Hash password with bcrypt — cost 12 minimum for production
    hash, err := bcrypt.GenerateFromPassword([]byte(req.Password), 12)
    if err != nil {
        h.logger.Error("failed to hash password", "error", err)
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
        return
    }

    user, err := h.userRepo.Create(c.Request.Context(), domain.CreateUserRequest{
        Email:        req.Email,
        PasswordHash: string(hash),
    })
    if err != nil {
        h.logger.Error("failed to create user", "error", err)
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
        return
    }

    c.JSON(http.StatusCreated, gin.H{"user_id": user.ID})
}
```

## Login Handler

> **Security note:** Never expose raw `err.Error()` to clients. Return generic messages and log the error server-side. See **golang-gin-clean-arch** error handling patterns.

Validates credentials via `UserRepository`, then returns both tokens. Thin handler — no business logic beyond orchestration.

```go
// internal/handler/auth_handler.go
package handler

import (
    "log/slog"
    "net/http"

    "github.com/gin-gonic/gin"
    "golang.org/x/crypto/bcrypt"
    "myapp/internal/auth"
    "myapp/internal/domain"
)

type AuthHandler struct {
    userRepo domain.UserRepository
    tokenCfg auth.TokenConfig
    logger   *slog.Logger
}

func NewAuthHandler(userRepo domain.UserRepository, cfg auth.TokenConfig, logger *slog.Logger) *AuthHandler {
    return &AuthHandler{userRepo: userRepo, tokenCfg: cfg, logger: logger}
}

type loginRequest struct {
    Email    string `json:"email"    binding:"required,email"`
    Password string `json:"password" binding:"required"`
}

type tokenResponse struct {
    AccessToken  string `json:"access_token"`
    RefreshToken string `json:"refresh_token"`
}

func (h *AuthHandler) Login(c *gin.Context) {
    var req loginRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "invalid request body"})
        return
    }

    user, err := h.userRepo.GetByEmail(c.Request.Context(), req.Email)
    if err != nil {
        // Return generic message — don't leak whether email exists
        c.JSON(http.StatusUnauthorized, gin.H{"error": "invalid credentials"})
        return
    }

    if err := bcrypt.CompareHashAndPassword([]byte(user.PasswordHash), []byte(req.Password)); err != nil {
        c.JSON(http.StatusUnauthorized, gin.H{"error": "invalid credentials"})
        return
    }

    accessToken, err := auth.GenerateAccessToken(h.tokenCfg, user.ID, user.Email, user.Role)
    if err != nil {
        h.logger.Error("failed to generate access token", "error", err, "user_id", user.ID)
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
        return
    }

    refreshToken, err := auth.GenerateRefreshToken(h.tokenCfg, user.ID)
    if err != nil {
        h.logger.Error("failed to generate refresh token", "error", err, "user_id", user.ID)
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
        return
    }

    c.JSON(http.StatusOK, tokenResponse{
        AccessToken:  accessToken,
        RefreshToken: refreshToken,
    })
}
```

## Protected Routes — Applying Auth Middleware

Wire the engine with `gin.New()` + explicit middleware, then apply `Auth` to route groups:

```go
// cmd/api/main.go — engine setup (always gin.New(), never gin.Default() in production)
r := gin.New()
r.Use(middleware.Logger(logger))
r.Use(middleware.Recovery(logger))
registerRoutes(r, authHandler, userHandler, tokenCfg, logger)
```

```go
// cmd/api/main.go (route registration)
func registerRoutes(r *gin.Engine, authHandler *handler.AuthHandler, userHandler *handler.UserHandler, cfg auth.TokenConfig, logger *slog.Logger) {
    r.GET("/health", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{"status": "ok"})
    })

    api := r.Group("/api/v1")

    // Auth routes — rate-limited: 5 requests per minute per IP
    authRoutes := api.Group("/auth")
    authRoutes.Use(middleware.IPRateLimiter(rate.Every(12*time.Second), 5))
    {
        authRoutes.POST("/login", authHandler.Login)
        authRoutes.POST("/register", authHandler.Register)
        authRoutes.POST("/refresh", authHandler.Refresh)
    }
    api.POST("/users", userHandler.Create) // registration (also rate-limit in production)

    // Protected routes — JWT required
    protected := api.Group("")
    protected.Use(middleware.Auth(cfg, logger))
    {
        protected.GET("/users/:id", userHandler.GetByID)
        protected.PUT("/users/:id", userHandler.Update)
        protected.DELETE("/users/:id", userHandler.Delete)

        // Admin-only routes — add RBAC middleware
        admin := protected.Group("/admin")
        admin.Use(middleware.RequireRole("admin"))
        {
            admin.GET("/users", userHandler.List)
        }
    }
}
```

### Rate Limiter Middleware

```go
// pkg/middleware/rate_limiter.go
package middleware

import (
    "net/http"
    "sync"

    "github.com/gin-gonic/gin"
    "golang.org/x/time/rate"
)

// IPRateLimiter limits requests per IP using a token-bucket algorithm.
// r controls how fast tokens refill; b is the burst (max simultaneous requests).
func IPRateLimiter(r rate.Limit, b int) gin.HandlerFunc {
    limiters := make(map[string]*rate.Limiter)
    var mu sync.Mutex

    getLimiter := func(ip string) *rate.Limiter {
        mu.Lock()
        defer mu.Unlock()
        if lim, ok := limiters[ip]; ok {
            return lim
        }
        lim := rate.NewLimiter(r, b)
        limiters[ip] = lim
        return lim
    }

    return func(c *gin.Context) {
        lim := getLimiter(c.ClientIP())
        if !lim.Allow() {
            c.AbortWithStatusJSON(http.StatusTooManyRequests, gin.H{"error": "too many requests"})
            return
        }
        c.Next()
    }
}
```

> **Production note:** The in-process map above works for single-instance deployments. For multi-instance deployments, use a Redis-backed limiter (e.g. `go-redis/redis_rate`) so limits are shared across all pods.

## Getting Current User in Handlers

```go
// Read claims injected by Auth middleware
func (h *UserHandler) GetMe(c *gin.Context) {
    // Option 1: type-safe string shortcut
    userID := c.GetString(middleware.UserIDKey)

    // Option 2: full claims object (for role, email, etc.)
    val, exists := c.Get(middleware.ClaimsKey)
    if !exists {
        c.JSON(http.StatusUnauthorized, gin.H{"error": "unauthorized"})
        return
    }
    claims, ok := val.(*auth.Claims)
    if !ok {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
        return
    }

    user, err := h.svc.GetByID(c.Request.Context(), userID)
    if err != nil {
        handleServiceError(c, err, h.logger)
        return
    }

    c.JSON(http.StatusOK, gin.H{
        "user": user,
        "role": claims.Role,
    })
}
```

## Reference Files

Load these when you need deeper detail:

- **[references/jwt-patterns.md](references/jwt-patterns.md)** — Access + refresh token architecture, token refresh endpoint, token blacklisting (Redis), RS256 vs HS256, custom claims, storage recommendations (httpOnly cookie vs localStorage), CSRF protection, complete auth flow
- **[references/rbac.md](references/rbac.md)** — RequireRole/RequireAnyRole middleware, permission-based access, role hierarchy, multi-tenant authorization, resource-level authorization, complete RBAC example

## Cross-Skill References

- For handler patterns (ShouldBindJSON, error responses, route groups): see the **golang-gin-api** skill
- For `UserRepository` interface and `GetByEmail` implementation: see the **golang-gin-database** skill
- For testing JWT middleware and auth handlers: see the **golang-gin-testing** skill
- **golang-gin-clean-arch** → Architecture: where auth middleware fits (delivery layer only), DI patterns for auth services

## Official Docs

If this skill doesn't cover your use case, consult the [Gin documentation](https://gin-gonic.com/docs/), [golang-jwt GoDoc](https://pkg.go.dev/github.com/golang-jwt/jwt/v5), or [Gin GoDoc](https://pkg.go.dev/github.com/gin-gonic/gin).
</file>

<file path="skills/golang-gin-swagger/SKILL.md">
---
name: golang-gin-swagger
description: "Generate Swagger/OpenAPI documentation for Go Gin APIs using swaggo/swag. Covers annotation syntax, model documentation, Swagger UI setup, security definitions, and CI/CD integration. Use when adding API docs, Swagger UI, OpenAPI spec, endpoint documentation, or auto-generated API reference to a Gin application."
license: MIT
metadata:
  author: henriqueatila
  version: 1.0.3
---

# golang-gin-swagger — Swagger/OpenAPI Documentation

Generate and serve Swagger/OpenAPI documentation for Gin APIs using [swaggo/swag](https://github.com/swaggo/swag). This skill covers the 80% you need daily: setup, handler annotations, model tags, Swagger UI, and doc generation.

## When to Use

- Adding Swagger/OpenAPI documentation to a Gin API
- Documenting endpoints with request/response schemas
- Serving Swagger UI for interactive API exploration
- Generating `swagger.json`/`swagger.yaml` from Go annotations
- Documenting JWT Bearer auth in OpenAPI spec
- Setting up CI/CD to validate docs are up to date

## Dependencies

```bash
# CLI tool (generates docs from annotations)
go install github.com/swaggo/swag/cmd/swag@latest

# Go module dependencies
go get -u github.com/swaggo/gin-swagger
go get -u github.com/swaggo/files
```

Ensure `$(go env GOPATH)/bin` is in your `$PATH` so the `swag` CLI is available.

## General API Annotations

Place these directly before `main()` in `cmd/api/main.go`. Only one annotation block per project.

```go
// @title           My API
// @version         1.0
// @description     Production-grade REST API built with Gin.

// @contact.name    API Support
// @contact.email   support@example.com

// @license.name    MIT
// @license.url     https://opensource.org/licenses/MIT

// @host            localhost:8080
// @BasePath        /api/v1
// @schemes         http https

// @securityDefinitions.apikey  BearerAuth
// @in                          header
// @name                        Authorization
// @description                 Enter: Bearer {token}
func main() { ... }
```

## Serving Swagger UI

```go
package main

import (
    "os"

    "github.com/gin-gonic/gin"
    swaggerFiles "github.com/swaggo/files"
    ginSwagger   "github.com/swaggo/gin-swagger"

    _ "myapp/docs" // CRITICAL: blank import registers the generated spec
)

func main() {
    r := gin.New()

    // Only expose Swagger UI outside production
    if os.Getenv("GIN_MODE") != "release" {
        r.GET("/swagger/*any", ginSwagger.WrapHandler(swaggerFiles.Handler))
    }

    // ... register routes, start server
}
```

Access at: `http://localhost:8080/swagger/index.html`

**Swagger UI options:**

```go
r.GET("/swagger/*any", ginSwagger.WrapHandler(
    swaggerFiles.Handler,
    ginSwagger.URL("/swagger/doc.json"),
    ginSwagger.DocExpansion("list"),           // "list"|"full"|"none"
    ginSwagger.DeepLinking(true),
    ginSwagger.DefaultModelsExpandDepth(1),    // -1 hides models section
    ginSwagger.PersistAuthorization(true),     // retains Bearer token across reloads
    ginSwagger.DefaultModelExpandDepth(1),    // expand depth for example section
    ginSwagger.DefaultModelRendering("example"), // "example"|"model"
))
```

## Dynamic Host Configuration

Override spec values at runtime for multi-environment deploys:

```go
import (
    "os"
    "myapp/docs"
)

func main() {
    docs.SwaggerInfo.Host     = os.Getenv("API_HOST") // e.g. "api.prod.example.com"
    docs.SwaggerInfo.Schemes  = []string{"https"}
    docs.SwaggerInfo.BasePath = "/api/v1"
    // ...
}
```

## Handler Annotations

Annotate each handler to document the endpoint. Always start with a Go doc comment.

```go
// CreateUser godoc
//
// @Summary      Create a new user
// @Description  Register a new user account
// @Tags         users
// @Accept       json
// @Produce      json
// @Param        request  body      domain.CreateUserRequest  true  "Create user payload"
// @Success      201      {object}  domain.User
// @Failure      400      {object}  domain.ErrorResponse
// @Failure      409      {object}  domain.ErrorResponse
// @Failure      500      {object}  domain.ErrorResponse
// @Router       /users [post]
func (h *UserHandler) Create(c *gin.Context) { ... }

// GetByID godoc
//
// @Summary      Get user by ID
// @Description  Retrieve a single user by UUID
// @Tags         users
// @Produce      json
// @Security     BearerAuth
// @Param        id   path      string  true  "User ID (UUID)"
// @Success      200  {object}  domain.User
// @Failure      400  {object}  domain.ErrorResponse
// @Failure      401  {object}  domain.ErrorResponse
// @Failure      404  {object}  domain.ErrorResponse
// @Failure      500  {object}  domain.ErrorResponse
// @Router       /users/{id} [get]
func (h *UserHandler) GetByID(c *gin.Context) { ... }

// List godoc
//
// @Summary      List users
// @Description  List users with pagination and optional role filter
// @Tags         users
// @Produce      json
// @Security     BearerAuth
// @Param        page   query     int     false  "Page number"      default(1) minimum(1)
// @Param        limit  query     int     false  "Items per page"   default(20) minimum(1) maximum(100)
// @Param        role   query     string  false  "Filter by role"   Enums(admin, user)
// @Success      200    {array}   domain.User
// @Failure      400    {object}  domain.ErrorResponse
// @Failure      401    {object}  domain.ErrorResponse
// @Failure      500    {object}  domain.ErrorResponse
// @Router       /users [get]
func (h *UserHandler) List(c *gin.Context) { ... }
```

**Critical rules:**
- `@Router` uses `{id}` (OpenAPI style), not `:id` (Gin style)
- `@Security BearerAuth` must match the name in `@securityDefinitions.apikey BearerAuth`
- Use named structs in `@Success`/`@Failure` — never `gin.H{}` or `map[string]interface{}`

## Model Documentation

> **Architecture note:** In clean architecture, domain entities should not carry `json` or `binding` tags. Use separate request/response DTOs in the delivery layer. See **golang-gin-clean-arch** Golden Rule 4. The tags here are shown for Swagger annotation purposes — in a clean-arch project, apply them to DTO structs, not domain entities.

Add `example`, `format`, and constraint tags to struct fields for rich Swagger docs:

```go
// internal/domain/user.go
package domain

import "time"

// User represents an authenticated system user.
type User struct {
    ID        string    `json:"id"         example:"550e8400-e29b-41d4-a716-446655440000" format:"uuid"`
    Name      string    `json:"name"       example:"Jane Doe"    minLength:"2" maxLength:"100"`
    Email     string    `json:"email"      example:"jane@example.com" format:"email"`
    Role      string    `json:"role"       example:"admin"       enums:"admin,user"`
    CreatedAt time.Time `json:"created_at" example:"2024-01-15T10:30:00Z" format:"date-time"`
    UpdatedAt time.Time `json:"updated_at" example:"2024-01-15T10:30:00Z" format:"date-time"`

    PasswordHash string `json:"-" swaggerignore:"true"`
}

// CreateUserRequest is the payload for POST /users.
type CreateUserRequest struct {
    Name     string `json:"name"     example:"Jane Doe"        binding:"required,min=2,max=100"`
    Email    string `json:"email"    example:"jane@example.com" binding:"required,email"`
    Password string `json:"password" example:"s3cur3P@ss!"      binding:"required,min=8"`
    Role     string `json:"role"     example:"user"             binding:"omitempty,oneof=admin user" enums:"admin,user"`
}

// ErrorResponse is the standard API error envelope.
type ErrorResponse struct {
    Error string `json:"error" example:"resource not found"`
}
```

**Key struct tags for swag:**

| Tag | Purpose | Example |
|-----|---------|---------|
| `example:"..."` | Sample value in Swagger UI | `example:"jane@example.com"` |
| `format:"..."` | OpenAPI format | `format:"uuid"`, `format:"email"`, `format:"date-time"` |
| `enums:"a,b"` | Allowed values | `enums:"admin,user"` |
| `swaggerignore:"true"` | Exclude field from docs | Hide `PasswordHash` |
| `swaggertype:"string"` | Override inferred type | For `time.Time`, `sql.NullInt64` |
| `minimum:` / `maximum:` | Numeric bounds | `minimum:"1" maximum:"100"` |
| `minLength:` / `maxLength:` | String length bounds | `minLength:"2" maxLength:"100"` |
| `default:"..."` | Default value | `default:"20"` |

## Generating Docs

```bash
# Standard cmd/ layout
swag init -g cmd/api/main.go -d ./,./internal/handler,./internal/domain

# Format annotations first (recommended)
swag fmt && swag init -g cmd/api/main.go

# Parse types from internal/ packages
swag init -g cmd/api/main.go --parseInternal

# Output: docs/docs.go, docs/swagger.json, docs/swagger.yaml
```

**Commit the generated `docs/` directory.** Re-run `swag init` after every handler or model change.

## Makefile Integration

```makefile
.PHONY: docs docs-check

docs:
	swag fmt
	swag init -g cmd/api/main.go -d ./,./internal/... --exclude ./vendor

docs-check:
	swag init -g cmd/api/main.go -d ./,./internal/... --exclude ./vendor
	git diff --exit-code docs/
```

## Common Gotchas

| Gotcha | Fix |
|--------|-----|
| `swag` CLI not found | Add `$(go env GOPATH)/bin` to `$PATH` |
| Docs not updating | Re-run `swag init` — no watch mode |
| Blank import `_ "myapp/docs"` missing | Without it, spec is never registered — Swagger UI shows empty |
| `@Router` uses `:id` instead of `{id}` | Use `{id}` in annotations (OpenAPI), `:id` in Gin routes |
| `@Security` name mismatch | Must match `@securityDefinitions.apikey` name exactly |
| `time.Time` rendered as object | Add `swaggertype:"string" format:"date-time"` on field |
| Type not found during parsing | Add `--parseInternal` or `--parseDependency` flag |
| `map[string]interface{}` in response | Replace with a named struct |
| `internal_` prefix on model names | Known bug with `--parseInternal` — use `--useStructName` |

## Excluding Swagger from Production Binary

Use build tags to strip Swagger UI from production builds entirely:

```go
// file: swagger.go
//go:build swagger

package main

import (
    _ "myapp/docs"
    ginSwagger   "github.com/swaggo/gin-swagger"
    swaggerFiles "github.com/swaggo/files"
    "github.com/gin-gonic/gin"
)

func registerSwagger(r *gin.Engine) {
    r.GET("/swagger/*any", ginSwagger.WrapHandler(swaggerFiles.Handler))
}
```

```go
// file: swagger_noop.go
//go:build !swagger

package main

import "github.com/gin-gonic/gin"

func registerSwagger(r *gin.Engine) {} // no-op in production
```

Build with `go build -tags swagger .` for dev/staging, `go build .` for production.

## Reference Files

Load these when you need deeper detail:

- **[references/annotations.md](references/annotations.md)** — Complete annotation reference: all @Param types, file uploads, response headers, enum from constants, model renaming, grouped responses, multiple auth schemes
- **[references/ci-cd.md](references/ci-cd.md)** — GitHub Actions workflow, PR validation, pre-commit hooks, OpenAPI 3.0 conversion, multiple swagger instances, swag init flags reference

## Cross-Skill References

- For handler patterns (ShouldBindJSON, route groups, error handling): see the **golang-gin-api** skill
- For JWT middleware and `@securityDefinitions.apikey BearerAuth`: see the **golang-gin-auth** skill
- For testing annotated handlers: see the **golang-gin-testing** skill
- For adding `swag init` to Docker builds: see the **golang-gin-deploy** skill

## Official Docs

If this skill doesn't cover your use case, consult the [swag GitHub](https://github.com/swaggo/swag), [gin-swagger GoDoc](https://pkg.go.dev/github.com/swaggo/gin-swagger), or [Swagger 2.0 spec](https://swagger.io/specification/v2/).
</file>

<file path="README.md">
# golang-gin-best-practices

Agent Skills for building production-grade REST APIs with Go and the Gin framework.

[![Release](https://img.shields.io/github/v/release/henriqueatila/golang-gin-best-practices?label=release)](https://github.com/henriqueatila/golang-gin-best-practices/releases/latest)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Skills.sh](https://img.shields.io/badge/skills.sh-golang--gin--best--practices-blue?logo=data:image/svg+xml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHZpZXdCb3g9IjAgMCAyNCAyNCIgZmlsbD0id2hpdGUiPjxwYXRoIGQ9Ik0xMiAyTDIgNy41djlMMTIgMjJsMTAtNS41di05TDEyIDJ6Ii8+PC9zdmc+)](https://skills.sh/henriqueatila/golang-gin-best-practices)

---

## Skills

| Skill | Description | Install |
|---|---|---|
| **golang-gin-api** | Core REST API — routing, handlers, binding, error handling, security headers, CORS, project structure | `npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-api` |
| **golang-gin-auth** | JWT auth, RBAC middleware, token lifecycle, CSRF protection, rate limiting, bcrypt, secure cookies | `npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-auth` |
| **golang-gin-database** | PostgreSQL with GORM/sqlx, repository pattern, migrations, connection retry, cursor pagination, transactions | `npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-database` |
| **golang-gin-testing** | Unit/integration/e2e tests, benchmarks, fuzz testing, golden files, testcontainers, coverage thresholds | `npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-testing` |
| **golang-gin-deploy** | Multi-stage Docker, K8s (PDB, NetworkPolicy, HPA), Trivy scanning, CI/CD, observability | `npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-deploy` |
| **golang-gin-swagger** | Swagger/OpenAPI with swaggo/swag, full CRUD + auth annotations, Swagger UI, CI/CD | `npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-swagger` |

---

## Quick Start

### Install all skills at once

```bash
npx skills add henriqueatila/golang-gin-best-practices
```

### Install individual skills

```bash
# Core API only
npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-api

# Add authentication to an existing project
npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-auth

# Add database layer
npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-database

# Add testing infrastructure
npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-testing

# Add deployment configuration
npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-deploy

# Add Swagger/OpenAPI documentation
npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-swagger
```

### Manual installation

Download the zip package for any skill from the `skills/` directory and extract it into your project's `.claude/skills/` folder:

```bash
# Example: download and extract golang-gin-api
curl -L https://github.com/henriqueatila/golang-gin-best-practices/raw/main/skills/golang-gin-api.zip -o golang-gin-api.zip
unzip golang-gin-api.zip -d .claude/skills/golang-gin-api/
```

---

## Architecture

### Progressive Disclosure

Each skill follows a two-level structure:

1. **SKILL.md** — The 80% you need daily. Loaded automatically by the agent. Covers the most common patterns with complete, compilable code examples. Under 500 lines.
2. **references/*.md** — The 20% for specific scenarios. Loaded on demand when you need deeper detail. Each file focuses on one topic.

This means the agent loads minimal context by default and fetches reference files only when needed — keeping token usage low while providing comprehensive coverage.

### Composability

Skills are designed to work together with shared conventions:

- **golang-gin-api** defines the project structure, `AppError` type, and handler patterns that all other skills follow
- **golang-gin-auth** provides JWT middleware consumed by **golang-gin-api** route groups
- **golang-gin-database** provides the `UserRepository` interface and implementations used by **golang-gin-auth** and **golang-gin-api** handlers
- **golang-gin-testing** tests the handlers, services, and middleware defined across all skills
- **golang-gin-deploy** builds the project structure from **golang-gin-api** and wires health checks from **golang-gin-api**
- **golang-gin-swagger** documents the handlers from **golang-gin-api**, the JWT security from **golang-gin-auth**, and integrates with **golang-gin-deploy** Docker builds

You can start with `golang-gin-api` and add skills incrementally as your project grows.

---

## Directory Structure

```
golang-gin-best-practices/
├── README.md
├── CLAUDE.md                        # Agent guidance for this repo
├── AGENTS.md                        # Multi-agent guidance
├── LICENSE
│
├── skills/
│   ├── golang-gin-api/
│   │   ├── SKILL.md                 # Server setup, routing, handlers, binding, errors
│   │   ├── metadata.json
│   │   ├── README.md
│   │   └── references/
│   │       ├── routing.md           # Route groups, versioning, pagination, file uploads
│   │       ├── middleware.md        # CORS, rate limiting, request ID, timeout, recovery
│   │       ├── error-handling.md    # AppError system, validation errors, panic recovery
│   │       ├── websocket.md         # gorilla/websocket, hub pattern, auth, ping/pong
│   │       └── rate-limiting.md     # Token bucket, sliding window, Redis, tiered limits
│   │
│   ├── golang-gin-auth/
│   │   ├── SKILL.md                 # JWT middleware, login handler, RBAC, token lifecycle
│   │   ├── metadata.json
│   │   ├── README.md
│   │   └── references/
│   │       ├── jwt-patterns.md      # Token refresh, blacklisting (Redis), RS256 vs HS256
│   │       └── rbac.md              # RequireRole, permissions, multi-tenant authorization
│   │
│   ├── golang-gin-database/
│   │   ├── SKILL.md                 # Repository pattern, GORM/sqlx, connection pooling, DI
│   │   ├── metadata.json
│   │   ├── README.md
│   │   └── references/
│   │       ├── gorm-patterns.md     # Models, CRUD, soft deletes, transactions, hooks
│   │       ├── sqlx-patterns.md     # Struct scanning, NamedExec, IN clauses, transactions
│   │       └── migrations.md        # golang-migrate CLI, zero-downtime, seeding, rollback
│   │
│   ├── golang-gin-testing/
│   │   ├── SKILL.md                 # httptest, table-driven tests, mock repositories
│   │   ├── metadata.json
│   │   ├── README.md
│   │   └── references/
│   │       ├── unit-tests.md        # Handler tests, middleware isolation, mock generation
│   │       ├── integration-tests.md # testcontainers, TestMain, DB lifecycle, cleanup
│   │       └── e2e.md               # Full flows, docker-compose tests, GitHub Actions CI
│   │
│   ├── golang-gin-deploy/
│   │   ├── SKILL.md                 # Multi-stage Dockerfile, docker-compose, health checks
│   │   ├── metadata.json
│   │   ├── README.md
│   │   └── references/
│   │       ├── dockerfile.md        # Distroless, build args, layer caching, image size
│   │       ├── docker-compose.md    # Air hot reload, pgadmin, networking, integration tests
│   │       ├── kubernetes.md        # Deployment, Service, ConfigMap, HPA, Ingress, Helm
│   │       └── observability.md     # OpenTelemetry tracing, metrics, slog correlation
│   │
│   ├── golang-gin-swagger/
│   │   ├── SKILL.md                 # Swagger UI, annotations, model tags, doc generation
│   │   ├── metadata.json
│   │   ├── README.md
│   │   └── references/
│   │       ├── annotations.md       # Complete annotation ref: @Param, uploads, enums, auth
│   │       └── ci-cd.md             # GitHub Actions, PR validation, OpenAPI 3.0, Docker
│   │
│   ├── golang-gin-api.zip           # Packaged skill (SKILL.md + references/)
│   ├── golang-gin-auth.zip
│   ├── golang-gin-database.zip
│   ├── golang-gin-deploy.zip
│   ├── golang-gin-testing.zip
│   └── golang-gin-swagger.zip
```

---

## Design Principles

**Opinionated patterns** — Each skill teaches one right way, not five options. The patterns are chosen for production correctness, not tutorial simplicity.

**Enterprise-audited** — Every skill passed a multi-agent enterprise audit covering correctness, security, completeness, and production-readiness. All code examples compile, handle errors properly, and follow OWASP security guidelines.

**PostgreSQL default** — All database examples target PostgreSQL. GORM and sqlx are both covered; the repository interface allows swapping implementations without touching business logic.

**Production-first** — Examples use `gin.New()` + explicit middleware instead of `gin.Default()`, `log/slog` instead of `fmt.Println`, `ShouldBind*` instead of `Bind*`, and environment variables instead of hardcoded configuration.

**Go 1.24+** — All code targets Go 1.24 or later. Uses standard library features (`log/slog`, `context`, `errors.As`) and avoids deprecated patterns.

**Thin handlers** — Handlers bind input, call a service, and format the response. Business logic lives in services. Data access lives in repositories. This is enforced consistently across all skills.

**Context propagation** — All blocking calls receive `c.Request.Context()` (handlers) or a propagated `context.Context` (services and repositories). This enables proper cancellation and tracing.

---

## Security

All skills enforce enterprise security patterns:

- **OWASP headers** — `X-Content-Type-Options`, `X-Frame-Options`, `CSP`, `HSTS`, `Permissions-Policy`
- **JWT** — `jti` for token blacklisting, `Issuer`/`Audience` validation, `NotBefore` claims
- **Auth hardening** — bcrypt (cost 12), rate limiting on auth endpoints, CSRF double-submit cookies, secure cookie flags
- **Database** — parameterized queries only, `pgconn.PgError` typed assertions, TLS/sslmode guidance
- **Containers** — non-root, read-only rootfs, dropped capabilities, seccomp, Trivy image scanning
- **No internal error leakage** — all error responses use generic messages; real errors logged server-side via `slog`

---

## Contributing

1. Follow the design principles above — new patterns must be production-ready
2. Code examples must compile and handle errors (no `_` for errors, no `fmt.Println`)
3. SKILL.md files must stay under 500 lines — move detail to reference files
4. Reference files over 300 lines must include a table of contents
5. Use `gin.New()`, `log/slog`, `ShouldBind*`, and `context.Context` consistently
6. Verify all Gin API calls match official documentation before submitting a PR

---

## License

MIT License — see [LICENSE](LICENSE) for details.
</file>

<file path="skills/golang-gin-api/SKILL.md">
---
name: golang-gin-api
description: "Build REST APIs with Go Gin framework. Covers routing, handler patterns, request binding/validation, middleware chains, error handling, and project structure. Use when creating Go web servers, REST endpoints, HTTP handlers, or working with the Gin framework. Also activate when the user mentions Gin routes, middleware, JSON responses, request parsing, or API structure in Go."
license: MIT
metadata:
  author: henriqueatila
  version: 1.0.5
---

# golang-gin-api — Core REST API Development

Build production-grade REST APIs with Go and Gin. This skill covers the 80% of patterns you need daily: server setup, routing, request binding, response formatting, and error handling.

## When to Use

- Creating a new Go REST API or HTTP server
- Adding routes, handlers, or middleware to a Gin app
- Binding and validating incoming JSON/query/URI parameters
- Structuring a Go project with a layered project structure
- Wiring handlers → services → repositories in main.go
- Returning consistent JSON error responses

## Project Structure

```
myapp/
├── cmd/
│   └── api/
│       └── main.go          # Entry point, wiring
├── internal/
│   ├── handler/             # HTTP handlers (thin layer)
│   ├── service/             # Business logic
│   ├── repository/          # Data access
│   └── domain/              # Entities, interfaces, errors
├── pkg/
│   └── middleware/          # Shared middleware
└── go.mod
```

Use `internal/` for code that must not be imported by other modules. Use `pkg/` for reusable middleware and utilities.

## Server Setup with Graceful Shutdown

```go
package main

import (
    "context"
    "log/slog"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"

    "github.com/gin-gonic/gin"
    "gorm.io/gorm"
    "myapp/internal/handler"
    "myapp/internal/service"
    "myapp/internal/repository"
    "myapp/pkg/auth"       // see gin-auth skill
    "myapp/pkg/middleware"
)

func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

    // Production: gin.New() + explicit middleware (NOT gin.Default())
    r := gin.New()
    r.SetTrustedProxies([]string{"10.0.0.0/8", "172.16.0.0/12", "192.168.0.0/16"}) // proxy CIDRs — c.ClientIP() is spoofable without this
    // For CDN (Cloudflare/GAE/Fly.io): use r.TrustedPlatform = gin.PlatformCloudflare instead
    r.Use(middleware.Logger(logger))
    r.Use(middleware.Recovery(logger))

    // Dependency injection (db initialized via gin-database skill)
    var db *gorm.DB // see gin-database skill for initialization
    userRepo := repository.NewUserRepository(db)
    userSvc := service.NewUserService(userRepo)
    userHandler := handler.NewUserHandler(userSvc, logger)
    authHandler := handler.NewAuthHandler(userSvc, logger)     // see gin-auth skill
    tokenCfg := auth.TokenConfig{Secret: os.Getenv("JWT_SECRET")} // see gin-auth skill

    registerRoutes(r, userHandler, authHandler, tokenCfg, logger)

    srv := &http.Server{
        Addr:              ":" + os.Getenv("PORT"),
        Handler:           r,
        ReadHeaderTimeout: 10 * time.Second,  // guards against Slowloris (CWE-400)
        ReadTimeout:       30 * time.Second,
        WriteTimeout:      30 * time.Second,
        IdleTimeout:       120 * time.Second,
    }

    go func() {
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            logger.Error("server failed", "error", err)
            os.Exit(1)
        }
    }()

    // Buffered channel — unbuffered misses signals
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    if err := srv.Shutdown(ctx); err != nil {
        logger.Error("graceful shutdown failed", "error", err)
    }
}
```

## Domain Model

> **Architecture note:** In clean architecture, domain entities should not carry `json` or `binding` tags. Use separate request/response DTOs in the delivery layer. See **golang-gin-clean-arch** Golden Rule 4.

```go
// internal/domain/user.go
package domain

import "time"

type User struct {
    ID           string    `json:"id"`
    Name         string    `json:"name"`
    Email        string    `json:"email"`
    PasswordHash string    `json:"-"` // never exposed via API
    Role         string    `json:"role"`
    CreatedAt    time.Time `json:"created_at"`
    UpdatedAt    time.Time `json:"updated_at"`
}

type CreateUserRequest struct {
    Name     string `json:"name"     binding:"required,min=2,max=100"`
    Email    string `json:"email"    binding:"required,email"`
    Password string `json:"password" binding:"required,min=8"`
    Role     string `json:"role"     binding:"omitempty,oneof=admin user"`
}
```

## Thin Handler Pattern

> **Security note:** In production, never expose raw `err.Error()` to clients. Return generic messages and log the error server-side. See **golang-gin-clean-arch** error handling patterns.

Handlers bind input, call a service, and format the response. No business logic.

```go
// internal/handler/user_handler.go
package handler

import (
    "log/slog"
    "net/http"

    "github.com/gin-gonic/gin"
    "myapp/internal/domain"
    "myapp/internal/service"
)

type UserHandler struct {
    svc    service.UserService
    logger *slog.Logger
}

func NewUserHandler(svc service.UserService, logger *slog.Logger) *UserHandler {
    return &UserHandler{svc: svc, logger: logger}
}

func (h *UserHandler) Create(c *gin.Context) {
    var req domain.CreateUserRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        // validationErrors formats validator messages into field-level errors (see error-handling.md)
        c.JSON(http.StatusBadRequest, gin.H{
            "error":  "validation failed",
            "fields": validationErrors(err),
        })
        return
    }

    user, err := h.svc.Create(c.Request.Context(), req)
    if err != nil {
        handleServiceError(c, err, h.logger)
        return
    }

    c.JSON(http.StatusCreated, user)
}

func (h *UserHandler) GetByID(c *gin.Context) {
    type uriParams struct {
        ID string `uri:"id" binding:"required"`
    }
    var params uriParams
    if err := c.ShouldBindURI(&params); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "invalid request parameters"})
        return
    }

    user, err := h.svc.GetByID(c.Request.Context(), params.ID)
    if err != nil {
        handleServiceError(c, err, h.logger)
        return
    }

    c.JSON(http.StatusOK, user)
}
```

## Route Registration

```go
func registerRoutes(r *gin.Engine, userHandler *handler.UserHandler, authHandler *handler.AuthHandler, tokenCfg auth.TokenConfig, logger *slog.Logger) {
    // Health check — no auth required
    r.GET("/health", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{"status": "ok"})
    })

    api := r.Group("/api/v1")

    // Public routes
    public := api.Group("")
    {
        public.POST("/users", userHandler.Create)
        public.POST("/auth/login", authHandler.Login)
    }

    // Protected routes — for JWT middleware, see gin-auth skill
    protected := api.Group("")
    protected.Use(middleware.Auth(tokenCfg, logger)) // see gin-auth skill
    {
        protected.GET("/users/:id", userHandler.GetByID)
        protected.GET("/users", userHandler.List)
        protected.PUT("/users/:id", userHandler.Update)
        protected.DELETE("/users/:id", userHandler.Delete)
    }
}
```

## Request Binding Patterns

```go
// JSON body
var req domain.CreateUserRequest
if err := c.ShouldBindJSON(&req); err != nil { ... }

// Query string: GET /users?page=1&limit=20&role=admin
type ListQuery struct {
    Page  int    `form:"page"  binding:"min=1"`
    Limit int    `form:"limit" binding:"min=1,max=100"`
    Role  string `form:"role"  binding:"omitempty,oneof=admin user"`
}
var q ListQuery
if err := c.ShouldBindQuery(&q); err != nil { ... }

// URI parameters: GET /users/:id
type URIParams struct {
    ID string `uri:"id" binding:"required"`
}
var params URIParams
if err := c.ShouldBindURI(&params); err != nil { ... }
```

**Critical:** Always use `ShouldBind*` — `Bind*` auto-aborts with 400 and prevents custom error responses.

## Input Sanitization

Struct tags validate format and constraints. Sanitize string fields **after** binding to neutralize injection payloads before they reach services or storage.

```go
import (
    "html"
    "path/filepath"
    "strings"
)

func (h *UserHandler) Create(c *gin.Context) {
    var req domain.CreateUserRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    // Sanitize after bind: trim whitespace, escape HTML entities
    req.Name = strings.TrimSpace(req.Name)
    req.Name = html.EscapeString(req.Name)
    req.Email = strings.TrimSpace(req.Email)
    req.Email = strings.ToLower(req.Email)

    user, err := h.svc.Create(c.Request.Context(), req)
    // ...
}
```

For file uploads, always strip directory components from client-supplied filenames:

```go
safeName := filepath.Base(file.Filename)
dst := filepath.Join("uploads", safeName)
```

## Centralized Error Handling

> **Note:** This `AppError` is a simplified version for this skill's examples. For the canonical domain error pattern with `Detail` field and 5xx guard, see the **golang-gin-clean-arch** skill.

```go
// internal/domain/errors.go
package domain

import "errors"

type AppError struct {
    Code    int
    Message string
    Err     error
}

func (e *AppError) Error() string { return e.Message }
func (e *AppError) Unwrap() error  { return e.Err }

// Is enables errors.Is() to match AppErrors by code or unwrap to check sentinel errors.
func (e *AppError) Is(target error) bool {
    t, ok := target.(*AppError)
    if ok {
        return e.Code == t.Code
    }
    return errors.Is(e.Err, target)
}

var (
    ErrNotFound       = &AppError{Code: 404, Message: "resource not found"}
    ErrUnauthorized   = &AppError{Code: 401, Message: "unauthorized"}
    ErrForbidden      = &AppError{Code: 403, Message: "forbidden"}
    ErrConflict       = &AppError{Code: 409, Message: "resource already exists"}
    ErrValidation     = &AppError{Code: 422, Message: "validation failed"}
)
```

```go
// internal/handler/errors.go
package handler

import (
    "errors"
    "log/slog"
    "net/http"

    "github.com/gin-gonic/gin"
    "myapp/internal/domain"
)

// handleServiceError maps domain errors to HTTP responses.
// Logger is used to record 5xx errors — the actual error is never sent to clients.
func handleServiceError(c *gin.Context, err error, logger *slog.Logger) {
    var appErr *domain.AppError
    if errors.As(err, &appErr) {
        if appErr.Code >= 500 {
            logger.ErrorContext(c.Request.Context(), "service error", "error", err, "path", c.FullPath())
        }
        c.JSON(appErr.Code, gin.H{"error": appErr.Message})
        return
    }
    logger.ErrorContext(c.Request.Context(), "unhandled error", "error", err, "path", c.FullPath())
    c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
}
```

## Goroutine Safety

**Always call `c.Copy()` before passing `*gin.Context` to a goroutine.** The original context is reused by the pool after the request ends.

```go
func (h *UserHandler) CreateWithNotification(c *gin.Context) {
    var req domain.CreateUserRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    user, err := h.svc.Create(c.Request.Context(), req)
    if err != nil {
        handleServiceError(c, err, h.logger)
        return
    }

    // c.Copy() — safe to use in goroutine, original c is NOT
    cCopy := c.Copy()
    go func() {
        h.notifier.SendWelcome(cCopy.Request.Context(), user)
    }()

    c.JSON(http.StatusCreated, user)
}
```

## Reference Files

Load these when you need deeper detail:

- **[references/routing.md](references/routing.md)** — Route groups, API versioning, path parameters, pagination, wildcard routes, file uploads, custom validators, request size limits
- **[references/middleware.md](references/middleware.md)** — CORS, security headers, request logging with slog, rate limiting, request ID, timeout, recovery, custom middleware template
- **[references/error-handling.md](references/error-handling.md)** — Full AppError system, sentinel errors, validation error formatting, panic recovery, consistent JSON error format
- **[references/websocket.md](references/websocket.md)** — WebSocket with gorilla/websocket: upgrade handler, hub pattern, auth before upgrade, ping/pong keepalive, graceful shutdown, JSON messages, testing
- **[references/rate-limiting.md](references/rate-limiting.md)** — Deep-dive rate limiting: token bucket, sliding window, Redis distributed, per-user/API-key quotas, tiered limits, response headers, graceful degradation

## Cross-Skill References

- For JWT middleware to protect routes: see the **golang-gin-auth** skill
- For wiring repositories into services and handlers: see the **golang-gin-database** skill
- For testing handlers and services: see the **golang-gin-testing** skill
- For Dockerizing this project structure: see the **golang-gin-deploy** skill
- **golang-gin-clean-arch** → Architecture: 4-layer separation, dependency injection, error propagation, input sanitization

## Official Docs

If this skill doesn't cover your use case, consult the [Gin documentation](https://gin-gonic.com/docs/) or [Gin GoDoc](https://pkg.go.dev/github.com/gin-gonic/gin).
</file>

<file path="skills/golang-gin-database/SKILL.md">
---
name: golang-gin-database
description: "Integrate PostgreSQL databases with Go Gin APIs using GORM or sqlx. Covers repository pattern, connection pooling, transactions, migrations, and dependency injection. Use when adding database support, creating models, writing queries, implementing repositories, setting up migrations, or wiring database layers into a Gin project. Also activate when the user mentions GORM, sqlx, database connection, SQL queries, repository pattern, or database migrations in a Go/Gin context."
license: MIT
metadata:
  author: henriqueatila
  version: 1.0.4
---

# golang-gin-database — Database Integration

Integrate PostgreSQL with Gin APIs using the repository pattern. Keeps database logic out of handlers and services, and supports swapping GORM ↔ sqlx without touching business logic.

## When to Use

- Adding a PostgreSQL database to a Gin project
- Implementing the repository pattern (interface + concrete implementation)
- Writing GORM or sqlx queries
- Setting up database connection pooling
- Running migrations (golang-migrate)
- Wiring repositories → services → handlers in `main.go`
- Writing context-propagating transactions

## Repository Interface Pattern

**Define the interface in the domain layer; implement it in the repository layer.** This inverts the dependency: services depend on an abstraction, not a concrete database library.

> **Architecture note:** In clean architecture, domain entities should not carry `json` or `binding` tags. Use separate request/response DTOs in the delivery layer. See **golang-gin-clean-arch** Golden Rule 4.

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

## Database Connection Setup

```go
// internal/repository/db.go
package repository

import (
    "fmt"
    "log/slog"
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
}

// NewGORMDB opens a PostgreSQL connection with connection pool settings.
// Architectural recommendation — not part of the Gin API.
func NewGORMDB(cfg Config, appLogger *slog.Logger) (*gorm.DB, error) {
    db, err := gorm.Open(postgres.Open(cfg.DSN), &gorm.Config{
        Logger: logger.Default.LogMode(logger.Warn),
    })
    if err != nil {
        return nil, fmt.Errorf("gorm.Open: %w", err)
    }

    sqlDB, err := db.DB()
    if err != nil {
        return nil, fmt.Errorf("db.DB: %w", err)
    }

    sqlDB.SetMaxOpenConns(cfg.MaxOpenConns)       // e.g. 25
    sqlDB.SetMaxIdleConns(cfg.MaxIdleConns)       // e.g. 5
    sqlDB.SetConnMaxLifetime(cfg.ConnMaxLifetime)  // e.g. 5*time.Minute

    if err := sqlDB.Ping(); err != nil {
        return nil, fmt.Errorf("db.Ping: %w", err)
    }

    appLogger.Info("database connected")
    return db, nil
}

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
        slog.Warn("database connection failed, retrying", "attempt", i+1, "backoff", backoff, "error", err)
        time.Sleep(backoff)
    }
    return nil, fmt.Errorf("failed to connect after %d retries: %w", maxRetries, err)
}
```

**TLS / production DSN:** Always use `sslmode=verify-full` in production to prevent MITM attacks.

```go
// Development (local Docker)
dsn := "host=localhost user=app password=secret dbname=myapp sslmode=disable"

// Production: verify the server certificate
dsn := "host=db.example.com user=app password=*** dbname=myapp sslmode=verify-full sslrootcert=/etc/ssl/certs/rds-ca.pem"
```

For sqlx, see [references/sqlx-patterns.md](references/sqlx-patterns.md#connection-setup).

## GORM Repository Implementation (Brief)

```go
// internal/repository/user_repository_gorm.go
package repository

import (
    "context"
    "errors"

    "gorm.io/gorm"
    "myapp/internal/domain"
)

type gormUserRepository struct {
    db *gorm.DB
}

func NewUserRepository(db *gorm.DB) domain.UserRepository {
    return &gormUserRepository{db: db}
}

func (r *gormUserRepository) GetByID(ctx context.Context, id string) (*domain.User, error) {
    var m UserModel
    if err := r.txFromCtx(ctx).WithContext(ctx).First(&m, "id = ?", id).Error; err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, domain.ErrNotFound.New(err)
        }
        return nil, domain.ErrInternal.New(err)
    }
    return m.ToDomain(), nil
}
```

For full GORM patterns (models, soft deletes, preloading, transactions, hooks): see [references/gorm-patterns.md](references/gorm-patterns.md).

## sqlx Repository Implementation (Brief)

```go
// internal/repository/user_repository_sqlx.go
package repository

import (
    "context"
    "database/sql"
    "errors"

    "github.com/jmoiron/sqlx"
    "myapp/internal/domain"
)

type sqlxUserRepository struct {
    db *sqlx.DB
}

func NewSqlxUserRepository(db *sqlx.DB) domain.UserRepository {
    return &sqlxUserRepository{db: db}
}

func (r *sqlxUserRepository) GetByID(ctx context.Context, id string) (*domain.User, error) {
    var u userRow
    err := r.db.GetContext(ctx, &u, `SELECT id, name, email, role, created_at, updated_at FROM users WHERE id = $1`, id)
    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, domain.ErrNotFound.New(err)
        }
        return nil, domain.ErrInternal.New(err)
    }
    return u.toDomain(), nil
}
```

For full sqlx patterns (struct scanning, NamedExec, IN clauses, transactions): see [references/sqlx-patterns.md](references/sqlx-patterns.md).

## Transaction Pattern (Context-Based)

Pass `*gorm.DB` via context so repositories transparently participate in a transaction. The service orchestrates; repositories just call `txFromCtx`.

```go
// internal/repository/tx.go (GORM approach)
package repository

import (
    "context"

    "gorm.io/gorm"
)

type txKey struct{}

// WithTx stores a transaction in ctx so repositories can use it.
func WithTx(ctx context.Context, tx *gorm.DB) context.Context {
    return context.WithValue(ctx, txKey{}, tx)
}

// txFromCtx returns the transaction from ctx, or the default db.
// Every repository method should call this instead of r.db directly.
func (r *gormUserRepository) txFromCtx(ctx context.Context) *gorm.DB {
    if tx, ok := ctx.Value(txKey{}).(*gorm.DB); ok {
        return tx
    }
    return r.db
}

// Create uses txFromCtx — works both standalone and inside a transaction.
func (r *gormUserRepository) Create(ctx context.Context, user *domain.User) error {
    m := fromDomain(user)
    if err := r.txFromCtx(ctx).WithContext(ctx).Create(m).Error; err != nil {
        return mapGORMError(err)
    }
    user.ID = m.ID
    user.CreatedAt = m.CreatedAt
    user.UpdatedAt = m.UpdatedAt
    return nil
}

// Service-layer usage — the service orchestrates the transaction:
// internal/service/user_service.go
func (s *userService) RegisterWithProfile(ctx context.Context, req domain.CreateUserRequest) error {
    return s.db.WithContext(ctx).Transaction(func(tx *gorm.DB) error {
        txCtx := repository.WithTx(ctx, tx)
        if err := s.userRepo.Create(txCtx, &user); err != nil {
            return err // tx rolled back automatically
        }
        return s.profileRepo.Create(txCtx, &profile)
    })
}
```

## Cursor / Keyset Pagination

Offset pagination (`LIMIT x OFFSET y`) degrades at large offsets because PostgreSQL must skip rows. **Keyset pagination** is O(log n) via an index seek — preferred for large or fast-growing tables.

See [Cursor Pagination in GORM Patterns](references/gorm-patterns.md#cursor--keyset-pagination) for implementation.

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
        MaxIdleConns:    5,
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

    // ... router setup, see gin-api skill
    _ = userHandler
}
```

**Critical:** Read `DATABASE_URL` from environment, never hardcode credentials. See the **golang-gin-deploy** skill for Docker/Kubernetes secrets.

## Reference Files

Load these for deeper detail:

- **[references/gorm-patterns.md](references/gorm-patterns.md)** — GORM model definition, CRUD, soft deletes, scopes, preloading, raw SQL, batch ops, hooks, connection pooling, PostgreSQL-specific features, complete repository implementation
- **[references/sqlx-patterns.md](references/sqlx-patterns.md)** — Connection setup, struct scanning, Get/Select/NamedExec, safe dynamic queries, sqlx.In, null handling, query builder, complete repository implementation
- **[references/migrations.md](references/migrations.md)** — golang-migrate CLI and library usage, file naming, zero-downtime migrations, startup vs CI/CD strategy, seeding, rollback

## Cross-Skill References

- For dependency injection wiring and main.go patterns: see the **golang-gin-api** skill
- For testing repositories with a real database: see the **golang-gin-testing** skill (integration tests)
- For running migrations in Docker containers: see the **golang-gin-deploy** skill
- For user authentication using the UserRepository: see the **golang-gin-auth** skill
- **golang-gin-clean-arch** → Architecture: repository pattern, domain error wrapping, transaction patterns

## Official Docs

If this skill doesn't cover your use case, consult the [GORM documentation](https://gorm.io/docs/), [sqlx GoDoc](https://pkg.go.dev/github.com/jmoiron/sqlx), or [Gin GoDoc](https://pkg.go.dev/github.com/gin-gonic/gin).
</file>

</files>
