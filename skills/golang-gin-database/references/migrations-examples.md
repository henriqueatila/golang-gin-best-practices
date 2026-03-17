# Migrations — Seeding, Rollbacks, and Example SQL Files

Data seeding strategies, rollback commands, and complete example migrations for users, roles, and triggers.

> **Architectural recommendation:** golang-migrate is not part of the Gin framework. These are mainstream Go community patterns.

## Seeding Data

**Option A: Dedicated seed migration (idempotent)**

```sql
-- 000099_seed_admin_user.up.sql
INSERT INTO users (id, name, email, password_hash, role)
VALUES ('a0000000-0000-0000-0000-000000000001',
    'Admin', 'admin@example.com', '$2a$12$...bcrypt-hash...', 'admin')
ON CONFLICT (email) DO NOTHING;
```

**Option B: Go seed script (more control)**

```go
// cmd/seed/main.go — run with: go run ./cmd/seed
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
        logger.Error("connect failed", "error", err); os.Exit(1)
    }
    defer db.Close()
    if err := seedAdminUser(context.Background(), db, logger); err != nil {
        logger.Error("seed failed", "error", err); os.Exit(1)
    }
    logger.Info("seed complete")
}

func seedAdminUser(ctx context.Context, db *sqlx.DB, logger *slog.Logger) error {
    _, err := db.ExecContext(ctx, `
        INSERT INTO users (id, name, email, password_hash, role)
        VALUES ($1, $2, $3, $4, $5) ON CONFLICT (email) DO NOTHING`,
        "a0000000-0000-0000-0000-000000000001", "Admin",
        os.Getenv("ADMIN_EMAIL"), os.Getenv("ADMIN_PASSWORD_HASH"), "admin",
    )
    if err != nil {
        return fmt.Errorf("seed admin: %w", err)
    }
    logger.Info("admin user seeded")
    return nil
}
```

**Critical:** Never hardcode credentials. Read from environment variables.

## Rollback Strategies

```bash
migrate -path db/migrations -database "$DATABASE_URL" down 1     # roll back 1
migrate -path db/migrations -database "$DATABASE_URL" version    # check version
migrate -path db/migrations -database "$DATABASE_URL" force <N>  # clear dirty state
```

| Environment | Strategy |
|-------------|----------|
| Local dev | `down 1` freely |
| Staging | `down` only to recover from failed deploy |
| Production | Never roll back schema — deploy a fix migration |

## Example Migrations

### 000001: Create Users Table

```sql
-- db/migrations/000001_create_users_table.up.sql
CREATE EXTENSION IF NOT EXISTS "pgcrypto";
CREATE TABLE users (
    id            UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
    name          VARCHAR(100) NOT NULL,
    email         VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role          VARCHAR(50)  NOT NULL DEFAULT 'user',
    created_at    TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at    TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    deleted_at    TIMESTAMPTZ
);
CREATE UNIQUE INDEX idx_users_email ON users(email) WHERE deleted_at IS NULL;
CREATE INDEX idx_users_role ON users(role);
```

```sql
-- db/migrations/000001_create_users_table.down.sql
DROP TABLE IF EXISTS users;
DROP EXTENSION IF EXISTS "pgcrypto";
```

For migrations 000002 (roles), 000003 (user_roles junction), and 000004 (updated_at trigger): see [migrations-schema-examples.md](migrations-schema-examples.md).
