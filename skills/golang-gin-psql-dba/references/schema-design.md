# Schema Design Reference

This file covers PostgreSQL schema design decisions for Go/Gin APIs: naming conventions, data type selection, constraint patterns, normalization guidance, soft delete, multi-tenancy (row-level and RLS), audit trail, and a complete e-commerce DDL example with all conventions applied. Use when designing a new schema, reviewing an existing schema for correctness, or choosing between competing data type or structural options. All Go examples use sqlx with raw SQL — no ORM.

> **Architectural note:** These are PostgreSQL and Go community patterns, not part of the Gin framework API.

## Table of Contents

1. [Naming Conventions](#naming-conventions)
2. [Data Type Selection](#data-type-selection)
3. [Constraints](#constraints)
4. [Normalization Guidance](#normalization-guidance)
5. [Soft Delete Pattern](#soft-delete-pattern)
6. [Multi-Tenancy Patterns](#multi-tenancy-patterns)
7. [Audit Trail Pattern](#audit-trail-pattern)
8. [Complete Schema Example](#complete-schema-example)

---

## Naming Conventions

Consistent naming lets agents, developers, and tooling (migrate, pgAdmin, pg_stat_statements) understand the schema without reading every table definition.

### Tables

- **Plural snake_case:** `users`, `orders`, `order_items`, `product_categories`
- Junction tables: combine both sides in alphabetical order — `order_items`, `user_roles`, `product_tags`
- Avoid abbreviations unless they are universal (`url`, `ip`, `uuid`)

### Columns

- **Singular snake_case:** `user_id`, `created_at`, `is_verified` (reserved for Go-managed booleans — prefer `NOT NULL DEFAULT false` columns without `is_` prefix when the column name is self-evident, e.g., `active`, `verified`)
- Foreign key columns: `{referenced_table_singular}_id` — e.g., `user_id`, `product_id`, `tenant_id`
- Timestamps: always `created_at`, `updated_at`, `deleted_at` — never `createdAt`, `CreatedAt`, or `create_time`

### Indexes

Pattern: `idx_{table}_{columns}`

```sql
CREATE INDEX idx_users_email          ON users (email);
CREATE INDEX idx_orders_user_id       ON orders (user_id);
CREATE INDEX idx_orders_created_at    ON orders (created_at DESC);
CREATE INDEX idx_order_items_order_id ON order_items (order_id);

-- Multi-column: list columns left to right in selectivity order (most selective first)
CREATE INDEX idx_orders_tenant_status ON orders (tenant_id, status);
```

### Constraints

Pattern: `{table}_{type}_{columns}`

| Type | Suffix | Example |
|------|--------|---------|
| Primary key | `_pk_id` | `users_pk_id` |
| Unique | `_uq_{col}` | `users_uq_email` |
| Foreign key | `_fk_{col}` | `orders_fk_user_id` |
| Check | `_ck_{col}` | `products_ck_price_positive` |
| Not null (via check) | `_nn_{col}` | `users_nn_email` |

```sql
ALTER TABLE users
    ADD CONSTRAINT users_pk_id        PRIMARY KEY (id),
    ADD CONSTRAINT users_uq_email     UNIQUE (email),
    ADD CONSTRAINT users_ck_role      CHECK (role IN ('admin', 'user', 'viewer'));

ALTER TABLE orders
    ADD CONSTRAINT orders_fk_user_id  FOREIGN KEY (user_id) REFERENCES users (id) ON DELETE RESTRICT;
```

### Functions and Triggers

Pattern: `{verb}_{noun}` — describes what the function does, not what it is.

```sql
CREATE FUNCTION update_updated_at()   -- verb: update, noun: updated_at
CREATE FUNCTION set_tenant_context()  -- verb: set, noun: tenant_context
CREATE TRIGGER orders_update_updated_at BEFORE UPDATE ON orders ...
```

---

## Data Type Selection

Choosing the wrong type costs you correctness (float rounding, timezone bugs) or performance (oversized columns, full table scans). These are the decisions that are hard to fix later.

### UUID vs BIGINT Primary Keys

| Concern | UUID (`gen_random_uuid()`) | UUIDv7 (`uuid_generate_v7()`) | BIGINT IDENTITY |
|---------|--------------------------|-------------------------------|-----------------|
| Sortable by creation time | No (v4 random) | Yes (time-prefixed) | Yes |
| B-tree index fragmentation | High (random inserts) | Low (monotonic prefix) | Low |
| Distributed ID generation | Yes — no coordination | Yes — no coordination | Requires sequence coordination |
| Expose sequential IDs to clients | No | No | Yes (enumerable) |
| Storage | 16 bytes | 16 bytes | 8 bytes |
| Go import | `github.com/google/uuid` | `github.com/google/uuid` (v7) | none |

**Recommendation:** Use UUIDv7 when you need non-enumerable IDs with good index performance. Use `BIGINT GENERATED ALWAYS AS IDENTITY` for internal tables never exposed via API (audit logs, event queues). Avoid random UUIDv4 for large tables — random inserts fragment B-tree indexes.

```sql
-- UUIDv7 (time-sortable, better index locality)
id UUID PRIMARY KEY DEFAULT gen_random_uuid() -- replace with uuid_generate_v7() if extension available

-- BIGINT IDENTITY (internal tables, simpler)
id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY
```

```go
// internal/repository/id.go
package repository

import "github.com/google/uuid"

// newID returns a UUIDv7 string for use as primary key.
// UUIDv7 is time-sortable, which reduces B-tree fragmentation.
func newID() string {
    id, err := uuid.NewV7()
    if err != nil {
        // UUIDv7 generation only fails on clock issues — fall back to v4
        return uuid.Must(uuid.NewRandom()).String()
    }
    return id.String()
}
```

### Timestamps — Always TIMESTAMPTZ

Never use `TIMESTAMP` (no timezone). `TIMESTAMPTZ` stores UTC and converts to the session timezone on read.

```sql
-- Correct
created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
deleted_at TIMESTAMPTZ  -- nullable: NULL = active

-- Wrong
created_at TIMESTAMP NOT NULL DEFAULT now()  -- ambiguous timezone — never use
```

```go
// Scan TIMESTAMPTZ into time.Time — the Go postgres driver returns UTC automatically
type orderRow struct {
    ID        string    `db:"id"`
    CreatedAt time.Time `db:"created_at"` // always UTC from TIMESTAMPTZ
    DeletedAt sql.NullTime `db:"deleted_at"`
}
```

### TEXT vs VARCHAR(n)

PostgreSQL stores `TEXT` and `VARCHAR(n)` identically. Prefer `TEXT` with an explicit `CHECK` constraint — constraints can be altered without rewriting the column; length limits on `VARCHAR(n)` require `ALTER COLUMN TYPE` (full table rewrite on older PG versions).

```sql
-- Preferred: TEXT + CHECK
name  TEXT NOT NULL CHECK (char_length(name)  BETWEEN 1 AND 200),
email TEXT NOT NULL CHECK (char_length(email) BETWEEN 3 AND 255 AND email LIKE '%@%'),
slug  TEXT NOT NULL CHECK (slug ~ '^[a-z0-9-]+$'),

-- Acceptable: VARCHAR(n) for well-known fixed formats
phone VARCHAR(20),

-- Avoid: unbounded TEXT on columns that should be limited
description TEXT  -- fine — no reasonable upper bound for free text
```

### NUMERIC for Money

Never store money in `FLOAT`, `REAL`, or `DOUBLE PRECISION` — IEEE 754 binary floats introduce rounding errors (0.1 + 0.2 ≠ 0.3).

```sql
price       NUMERIC(19,4) NOT NULL DEFAULT 0,  -- up to 999,999,999,999,999.9999
tax_rate    NUMERIC(5,4)  NOT NULL DEFAULT 0,  -- 0.0000 to 9.9999 (e.g., 0.2000 = 20%)
total       NUMERIC(19,4) GENERATED ALWAYS AS (price * (1 + tax_rate)) STORED
```

```go
// Use shopspring/decimal in Go to match NUMERIC precision
import "github.com/shopspring/decimal"

type productRow struct {
    ID    string          `db:"id"`
    Price decimal.Decimal `db:"price"`
}
```

### JSONB for Semi-Structured Data

`JSONB` stores a binary-parsed representation; supports GIN indexes and containment operators (`@>`, `<@`, `?`).

**Use JSONB when:**
- The shape varies per row (user preferences, plugin config, webhook payloads)
- You need to query inside the document with `@>` or `jsonb_path_query`
- The data is read-mostly and updated as a unit

**Do NOT use JSONB when:**
- You filter, sort, or join on a field regularly — extract it into a proper column
- The field appears in WHERE clauses with equality — a B-tree on a generated column is better
- The structure is fixed and well-known — use proper columns

```sql
-- Good: preferences shape varies per user
preferences JSONB NOT NULL DEFAULT '{}',

-- Good: GIN index for containment queries
CREATE INDEX idx_users_preferences ON users USING gin (preferences jsonb_path_ops);

-- Bad: status field in JSONB when you filter by status constantly
-- Extract it: status TEXT NOT NULL CHECK (status IN ('active', 'suspended'))
```

### Network Types

```sql
ip_address   INET,   -- single IPv4 or IPv6 address (e.g., '192.168.1.1')
network_cidr CIDR,   -- network address (e.g., '192.168.1.0/24') — host bits must be zero
```

```go
// INET scans as string in database/sql with lib/pq
type eventRow struct {
    IPAddress string `db:"ip_address"`
}
```

### PostgreSQL Arrays

Use arrays for small, bounded, query-able lists of a single scalar type. Common use case: tags.

```sql
-- Tags on a post: small, bounded, queried with && (overlap) or @> (contains)
tags TEXT[] NOT NULL DEFAULT '{}',

-- GIN index for array operators
CREATE INDEX idx_posts_tags ON posts USING gin (tags);
```

```go
import "github.com/lib/pq"

type postRow struct {
    Tags pq.StringArray `db:"tags"` // pq.StringArray implements sql.Scanner
}
```

**Avoid arrays for:** relationships between entities (use a junction table), large or unbounded lists, or anything that needs foreign key constraints.

### Boolean Columns

Always `NOT NULL DEFAULT false`. Nullable booleans introduce three-valued logic (`TRUE`, `FALSE`, `NULL`) which causes subtle WHERE clause bugs.

```sql
active    BOOLEAN NOT NULL DEFAULT false,
verified  BOOLEAN NOT NULL DEFAULT false,
published BOOLEAN NOT NULL DEFAULT false,

-- Never:
active BOOLEAN  -- NULL is not "inactive", it's "unknown" — avoid
```

---

## Constraints

### Constraint Types and DDL

```sql
CREATE TABLE users (
    -- Primary key
    id            UUID        NOT NULL,
    -- Not null
    email         TEXT        NOT NULL,
    name          TEXT        NOT NULL,
    role          TEXT        NOT NULL DEFAULT 'user',
    active        BOOLEAN     NOT NULL DEFAULT false,
    -- Nullable
    bio           TEXT,
    phone         TEXT,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at    TIMESTAMPTZ,

    -- Named constraints
    CONSTRAINT users_pk_id        PRIMARY KEY (id),
    CONSTRAINT users_uq_email     UNIQUE (email),
    CONSTRAINT users_ck_role      CHECK (role IN ('admin', 'user', 'viewer')),
    CONSTRAINT users_ck_name_len  CHECK (char_length(name) BETWEEN 1 AND 200),
    CONSTRAINT users_ck_email_fmt CHECK (char_length(email) BETWEEN 3 AND 255)
);
```

### ON DELETE Decision Table

| Scenario | Clause | Effect |
|----------|--------|--------|
| Child rows are meaningless without parent (order_items → orders) | `ON DELETE CASCADE` | Delete child rows automatically |
| Child rows must not be orphaned; parent deletion blocked if children exist (users → tenant) | `ON DELETE RESTRICT` | Error if parent has children |
| Child rows remain valid without parent; FK nullable (orders → promo_code) | `ON DELETE SET NULL` | Set FK column to NULL |
| Child rows remain valid with a default parent | `ON DELETE SET DEFAULT` | Set FK column to default value |

```sql
-- order_items deleted when the order is deleted
CONSTRAINT order_items_fk_order_id
    FOREIGN KEY (order_id) REFERENCES orders (id) ON DELETE CASCADE,

-- Cannot delete a user who has orders
CONSTRAINT orders_fk_user_id
    FOREIGN KEY (user_id) REFERENCES users (id) ON DELETE RESTRICT,

-- Coupon optional; if coupon deleted, column becomes NULL
CONSTRAINT orders_fk_coupon_id
    FOREIGN KEY (coupon_id) REFERENCES coupons (id) ON DELETE SET NULL
```

**Always specify `ON DELETE`**. Omitting it defaults to `RESTRICT` but is ambiguous to readers — make intent explicit.

### Exclusion Constraints for Ranges

Prevent overlapping date ranges (booking systems, subscription periods):

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

CREATE TABLE room_bookings (
    id          UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
    room_id     UUID         NOT NULL,
    period      TSTZRANGE    NOT NULL,   -- e.g., '[2026-03-01, 2026-03-05)'
    CONSTRAINT room_bookings_no_overlap
        EXCLUDE USING gist (room_id WITH =, period WITH &&)
);
```

---

## Normalization Guidance

### When to Normalize (3NF)

Normalize by default. A table is in Third Normal Form (3NF) when:
1. Every non-key column depends on the whole primary key (not a subset)
2. No non-key column depends on another non-key column (no transitive dependency)

Normalize when:
- The same data appears in multiple rows (city/state repeated on every user)
- Data is updated independently (product name stored redundantly in orders)
- Referential integrity matters (roles defined in a table, not free-text strings)

### When to Denormalize

Denormalize intentionally and document why:
- **Read-heavy aggregates** that are expensive to recompute on every query (running totals, materialized counts)
- **Historical snapshots** that must not change (order `unit_price` at purchase time, copied from products table)
- **JSONB for flexible attributes** where shape varies per row (product metadata per category)

```sql
-- Intentional denormalization: snapshot price at order time
-- product price may change, but order history must not
CREATE TABLE order_items (
    id           UUID          PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id     UUID          NOT NULL,
    product_id   UUID          NOT NULL,
    quantity     INT           NOT NULL CHECK (quantity > 0),
    unit_price   NUMERIC(19,4) NOT NULL,  -- snapshotted at purchase, not a FK lookup
    CONSTRAINT order_items_fk_order_id   FOREIGN KEY (order_id)   REFERENCES orders   (id) ON DELETE CASCADE,
    CONSTRAINT order_items_fk_product_id FOREIGN KEY (product_id) REFERENCES products (id) ON DELETE RESTRICT
);
```

### Junction Tables for Many-to-Many

Never store multiple FKs in an array column. Use a junction table with a composite PK.

```sql
CREATE TABLE product_tags (
    product_id UUID NOT NULL,
    tag_id     UUID NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT product_tags_pk            PRIMARY KEY (product_id, tag_id),
    CONSTRAINT product_tags_fk_product_id FOREIGN KEY (product_id) REFERENCES products (id) ON DELETE CASCADE,
    CONSTRAINT product_tags_fk_tag_id     FOREIGN KEY (tag_id)     REFERENCES tags     (id) ON DELETE CASCADE
);
```

---

## Soft Delete Pattern

Soft delete records by setting `deleted_at` rather than physically removing rows. This preserves audit history and allows recovery.

### DDL

```sql
ALTER TABLE users ADD COLUMN deleted_at TIMESTAMPTZ;

-- Partial unique index: enforce uniqueness only among active (non-deleted) rows
CREATE UNIQUE INDEX users_uq_email_active
    ON users (email)
    WHERE deleted_at IS NULL;

-- Index for efficient soft-delete filter (most queries include WHERE deleted_at IS NULL)
CREATE INDEX idx_users_active ON users (id) WHERE deleted_at IS NULL;
```

The partial unique index is critical: without it, re-registering a previously deleted email would fail on the plain `UNIQUE (email)` constraint.

### Go Repository Query

```go
// internal/repository/user_repository_sqlx.go
package repository

import (
    "context"
    "database/sql"
    "errors"
    "fmt"
    "time"

    "github.com/jmoiron/sqlx"
    "myapp/internal/domain"
)

const (
    listActiveUsersSQL = `
        SELECT id, name, email, role, created_at, updated_at
        FROM users
        WHERE deleted_at IS NULL
        ORDER BY created_at DESC
        LIMIT $1 OFFSET $2
    `

    softDeleteUserSQL = `
        UPDATE users
        SET deleted_at = now()
        WHERE id = $1 AND deleted_at IS NULL
    `
)

func (r *sqlxUserRepository) List(ctx context.Context, limit, offset int) ([]domain.User, error) {
    var rows []userRow
    if err := r.db.SelectContext(ctx, &rows, listActiveUsersSQL, limit, offset); err != nil {
        return nil, fmt.Errorf("list users: %w", err)
    }
    users := make([]domain.User, len(rows))
    for i, row := range rows {
        users[i] = *row.toDomain()
    }
    return users, nil
}

func (r *sqlxUserRepository) Delete(ctx context.Context, id string) error {
    result, err := r.db.ExecContext(ctx, softDeleteUserSQL, id)
    if err != nil {
        return fmt.Errorf("soft delete user: %w", err)
    }
    n, err := result.RowsAffected()
    if err != nil {
        return fmt.Errorf("rows affected: %w", err)
    }
    if n == 0 {
        return domain.ErrNotFound.New(fmt.Errorf("user %s not found or already deleted", id))
    }
    return nil
}

// HardDelete permanently removes a user — use only for GDPR erasure.
func (r *sqlxUserRepository) HardDelete(ctx context.Context, id string) error {
    _, err := r.db.ExecContext(ctx, `DELETE FROM users WHERE id = $1`, id)
    if err != nil {
        return fmt.Errorf("hard delete user: %w", err)
    }
    return nil
}
```

### Full DDL Example (Users with Soft Delete)

```sql
CREATE TABLE users (
    id            UUID        NOT NULL,
    email         TEXT        NOT NULL,
    name          TEXT        NOT NULL,
    role          TEXT        NOT NULL DEFAULT 'user',
    active        BOOLEAN     NOT NULL DEFAULT false,
    bio           TEXT,
    preferences   JSONB       NOT NULL DEFAULT '{}',
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at    TIMESTAMPTZ,

    CONSTRAINT users_pk_id        PRIMARY KEY (id),
    CONSTRAINT users_ck_role      CHECK (role IN ('admin', 'user', 'viewer')),
    CONSTRAINT users_ck_name_len  CHECK (char_length(name) BETWEEN 1 AND 200),
    CONSTRAINT users_ck_email_fmt CHECK (char_length(email) BETWEEN 3 AND 255)
);

-- Unique email among active rows only
CREATE UNIQUE INDEX users_uq_email_active ON users (email) WHERE deleted_at IS NULL;

-- Efficient active-row filter
CREATE INDEX idx_users_active     ON users (id)         WHERE deleted_at IS NULL;
CREATE INDEX idx_users_role       ON users (role)        WHERE deleted_at IS NULL;
CREATE INDEX idx_users_created_at ON users (created_at DESC);

-- Auto-update updated_at
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = now();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER users_update_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION update_updated_at();
```

---

## Multi-Tenancy Patterns

### Pattern A — Shared Schema with tenant_id (Row-Level Isolation)

Add `tenant_id UUID NOT NULL` to every tenant-scoped table. Simple to implement; all tenants share the same tables.

```sql
ALTER TABLE orders ADD COLUMN tenant_id UUID NOT NULL;
ALTER TABLE orders
    ADD CONSTRAINT orders_fk_tenant_id FOREIGN KEY (tenant_id) REFERENCES tenants (id) ON DELETE RESTRICT;

-- Composite index: tenant_id first (partitions data per tenant in B-tree)
CREATE INDEX idx_orders_tenant_id_created_at ON orders (tenant_id, created_at DESC);
```

**Every query must include `WHERE tenant_id = $1`** — the application enforces isolation, not the database. Easy to miss; see Pattern B for database-enforced isolation.

### Pattern B — Row-Level Security (RLS)

RLS enforces tenant isolation at the database level. Even if application code omits the filter, PostgreSQL rejects cross-tenant reads.

```sql
-- Enable RLS on the table
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- Policy: users can only see rows belonging to their tenant
-- app.current_tenant_id must be set via SET LOCAL before every query
CREATE POLICY orders_tenant_isolation ON orders
    USING (tenant_id = current_setting('app.current_tenant_id')::uuid);

-- Bypass for the migration/admin superuser role
ALTER TABLE orders FORCE ROW LEVEL SECURITY; -- applies to table owner too
```

```go
// internal/middleware/tenant_middleware.go
package middleware

import (
    "fmt"
    "net/http"

    "github.com/gin-gonic/gin"
    "github.com/jmoiron/sqlx"
)

// TenantContext sets app.current_tenant_id in the PostgreSQL session
// so RLS policies can enforce tenant isolation.
func TenantContext(db *sqlx.DB) gin.HandlerFunc {
    return func(c *gin.Context) {
        tenantID := c.GetHeader("X-Tenant-ID")
        if tenantID == "" {
            c.AbortWithStatusJSON(http.StatusBadRequest, gin.H{"error": "X-Tenant-ID header required"})
            return
        }

        // Acquire a connection and set the tenant context for this request
        conn, err := db.Conn(c.Request.Context())
        if err != nil {
            c.AbortWithStatusJSON(http.StatusInternalServerError, gin.H{"error": "db connection"})
            return
        }
        defer conn.Close()

        _, err = conn.ExecContext(c.Request.Context(),
            fmt.Sprintf("SET LOCAL app.current_tenant_id = '%s'", tenantID),
        )
        if err != nil {
            c.AbortWithStatusJSON(http.StatusInternalServerError, gin.H{"error": "tenant context"})
            return
        }

        c.Set("tenant_id", tenantID)
        c.Set("db_conn", conn)
        c.Next()
    }
}
```

**Security note:** Never interpolate `tenantID` directly into SQL without validation. Validate it is a valid UUID before the `SET LOCAL` call.

### Pattern C — Schema-per-Tenant (Brief)

Each tenant gets a dedicated PostgreSQL schema (`tenant_abc.orders`, `tenant_xyz.orders`). Provides complete isolation and easy per-tenant backups. Trade-offs: schema proliferation (100+ schemas), complex migrations (must run against every schema), and higher operational overhead. Suitable only when strong data isolation is a contractual or compliance requirement.

---

## Audit Trail Pattern

### Audit Log Table

```sql
CREATE TABLE audit_log (
    id           BIGINT      GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    table_name   TEXT        NOT NULL,
    record_id    UUID        NOT NULL,
    operation    TEXT        NOT NULL CHECK (operation IN ('INSERT', 'UPDATE', 'DELETE')),
    changed_by   UUID,        -- NULL for system/migration operations
    changed_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    old_values   JSONB,       -- NULL on INSERT
    new_values   JSONB        -- NULL on DELETE
);

-- Do not soft-delete audit_log rows — they are immutable history
CREATE INDEX idx_audit_log_table_record ON audit_log (table_name, record_id);
CREATE INDEX idx_audit_log_changed_at   ON audit_log (changed_at DESC);
CREATE INDEX idx_audit_log_changed_by   ON audit_log (changed_by) WHERE changed_by IS NOT NULL;
```

### Trigger for Automatic Audit Logging

```sql
CREATE OR REPLACE FUNCTION record_audit_log()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO audit_log (table_name, record_id, operation, new_values)
        VALUES (TG_TABLE_NAME, NEW.id, 'INSERT', to_jsonb(NEW));
        RETURN NEW;
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO audit_log (table_name, record_id, operation, old_values, new_values)
        VALUES (TG_TABLE_NAME, NEW.id, 'UPDATE', to_jsonb(OLD), to_jsonb(NEW));
        RETURN NEW;
    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO audit_log (table_name, record_id, operation, old_values)
        VALUES (TG_TABLE_NAME, OLD.id, 'DELETE', to_jsonb(OLD));
        RETURN OLD;
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

-- Apply to any audited table
CREATE TRIGGER orders_audit
    AFTER INSERT OR UPDATE OR DELETE ON orders
    FOR EACH ROW EXECUTE FUNCTION record_audit_log();
```

**Caution:** `to_jsonb(NEW)` captures all columns, including sensitive fields like `password_hash`. Either exclude sensitive tables from trigger-based audit, or sanitize the JSONB in the trigger function.

### Go Helper for Manual Audit Entries

Use when the trigger approach is insufficient (e.g., bulk operations, application-level context such as the acting user ID).

```go
// internal/repository/audit_repository.go
package repository

import (
    "context"
    "encoding/json"
    "fmt"
    "time"

    "github.com/jmoiron/sqlx"
)

type AuditEntry struct {
    TableName string
    RecordID  string
    Operation string // INSERT, UPDATE, DELETE
    ChangedBy string // user ID — empty string for system ops
    OldValues any    // struct or map; nil for INSERT
    NewValues any    // struct or map; nil for DELETE
}

const insertAuditSQL = `
    INSERT INTO audit_log (table_name, record_id, operation, changed_by, changed_at, old_values, new_values)
    VALUES ($1, $2, $3, NULLIF($4, '')::uuid, $5, $6, $7)
`

func WriteAuditEntry(ctx context.Context, db *sqlx.DB, entry AuditEntry) error {
    oldJSON, err := marshalNullable(entry.OldValues)
    if err != nil {
        return fmt.Errorf("audit marshal old: %w", err)
    }
    newJSON, err := marshalNullable(entry.NewValues)
    if err != nil {
        return fmt.Errorf("audit marshal new: %w", err)
    }

    _, err = db.ExecContext(ctx, insertAuditSQL,
        entry.TableName,
        entry.RecordID,
        entry.Operation,
        entry.ChangedBy,
        time.Now().UTC(),
        oldJSON,
        newJSON,
    )
    if err != nil {
        return fmt.Errorf("write audit entry: %w", err)
    }
    return nil
}

func marshalNullable(v any) ([]byte, error) {
    if v == nil {
        return nil, nil
    }
    return json.Marshal(v)
}
```

---

## Complete Schema Example

Full DDL for an e-commerce subset: tenants, users, products, orders, order_items. All naming conventions, constraints, indexes, triggers, and soft delete applied.

```sql
-- ============================================================
-- Extensions
-- ============================================================
CREATE EXTENSION IF NOT EXISTS pgcrypto;   -- gen_random_uuid()
CREATE EXTENSION IF NOT EXISTS btree_gist; -- exclusion constraints (optional)

-- ============================================================
-- Tenants
-- ============================================================
CREATE TABLE tenants (
    id         UUID        NOT NULL,
    name       TEXT        NOT NULL,
    slug       TEXT        NOT NULL,
    active     BOOLEAN     NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at TIMESTAMPTZ,

    CONSTRAINT tenants_pk_id       PRIMARY KEY (id),
    CONSTRAINT tenants_ck_name_len CHECK (char_length(name) BETWEEN 1 AND 200),
    CONSTRAINT tenants_ck_slug_fmt CHECK (slug ~ '^[a-z0-9-]+$')
);

CREATE UNIQUE INDEX tenants_uq_slug_active ON tenants (slug) WHERE deleted_at IS NULL;
CREATE INDEX idx_tenants_active            ON tenants (id)   WHERE deleted_at IS NULL;

CREATE TRIGGER tenants_update_updated_at
    BEFORE UPDATE ON tenants
    FOR EACH ROW EXECUTE FUNCTION update_updated_at();

-- ============================================================
-- Users
-- ============================================================
CREATE TABLE users (
    id            UUID        NOT NULL,
    tenant_id     UUID        NOT NULL,
    email         TEXT        NOT NULL,
    name          TEXT        NOT NULL,
    password_hash TEXT        NOT NULL,
    role          TEXT        NOT NULL DEFAULT 'user',
    verified      BOOLEAN     NOT NULL DEFAULT false,
    preferences   JSONB       NOT NULL DEFAULT '{}',
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at    TIMESTAMPTZ,

    CONSTRAINT users_pk_id         PRIMARY KEY (id),
    CONSTRAINT users_fk_tenant_id  FOREIGN KEY (tenant_id) REFERENCES tenants (id) ON DELETE RESTRICT,
    CONSTRAINT users_ck_role       CHECK (role IN ('admin', 'user', 'viewer')),
    CONSTRAINT users_ck_name_len   CHECK (char_length(name) BETWEEN 1 AND 200),
    CONSTRAINT users_ck_email_fmt  CHECK (char_length(email) BETWEEN 3 AND 255)
);

-- Unique email per tenant among active rows
CREATE UNIQUE INDEX users_uq_tenant_email_active ON users (tenant_id, email) WHERE deleted_at IS NULL;
CREATE INDEX idx_users_tenant_id                 ON users (tenant_id)         WHERE deleted_at IS NULL;
CREATE INDEX idx_users_created_at                ON users (created_at DESC);

CREATE TRIGGER users_update_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION update_updated_at();

-- ============================================================
-- Products
-- ============================================================
CREATE TABLE products (
    id          UUID          NOT NULL,
    tenant_id   UUID          NOT NULL,
    name        TEXT          NOT NULL,
    slug        TEXT          NOT NULL,
    description TEXT,
    price       NUMERIC(19,4) NOT NULL DEFAULT 0,
    stock       INT           NOT NULL DEFAULT 0,
    tags        TEXT[]        NOT NULL DEFAULT '{}',
    metadata    JSONB         NOT NULL DEFAULT '{}',
    active      BOOLEAN       NOT NULL DEFAULT false,
    created_at  TIMESTAMPTZ   NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ   NOT NULL DEFAULT now(),
    deleted_at  TIMESTAMPTZ,

    CONSTRAINT products_pk_id              PRIMARY KEY (id),
    CONSTRAINT products_fk_tenant_id       FOREIGN KEY (tenant_id) REFERENCES tenants (id) ON DELETE RESTRICT,
    CONSTRAINT products_ck_price_positive  CHECK (price >= 0),
    CONSTRAINT products_ck_stock_positive  CHECK (stock >= 0),
    CONSTRAINT products_ck_name_len        CHECK (char_length(name) BETWEEN 1 AND 300),
    CONSTRAINT products_ck_slug_fmt        CHECK (slug ~ '^[a-z0-9-]+$')
);

CREATE UNIQUE INDEX products_uq_tenant_slug_active ON products (tenant_id, slug) WHERE deleted_at IS NULL;
CREATE INDEX idx_products_tenant_active            ON products (tenant_id)       WHERE deleted_at IS NULL;
CREATE INDEX idx_products_tags                     ON products USING gin (tags);
CREATE INDEX idx_products_metadata                 ON products USING gin (metadata jsonb_path_ops);

CREATE TRIGGER products_update_updated_at
    BEFORE UPDATE ON products
    FOR EACH ROW EXECUTE FUNCTION update_updated_at();

-- ============================================================
-- Orders
-- ============================================================
CREATE TABLE orders (
    id          UUID          NOT NULL,
    tenant_id   UUID          NOT NULL,
    user_id     UUID          NOT NULL,
    status      TEXT          NOT NULL DEFAULT 'pending',
    total       NUMERIC(19,4) NOT NULL DEFAULT 0,
    coupon_id   UUID,
    notes       TEXT,
    created_at  TIMESTAMPTZ   NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ   NOT NULL DEFAULT now(),
    deleted_at  TIMESTAMPTZ,

    CONSTRAINT orders_pk_id          PRIMARY KEY (id),
    CONSTRAINT orders_fk_tenant_id   FOREIGN KEY (tenant_id) REFERENCES tenants (id) ON DELETE RESTRICT,
    CONSTRAINT orders_fk_user_id     FOREIGN KEY (user_id)   REFERENCES users   (id) ON DELETE RESTRICT,
    CONSTRAINT orders_fk_coupon_id   FOREIGN KEY (coupon_id) REFERENCES coupons (id) ON DELETE SET NULL,
    CONSTRAINT orders_ck_status      CHECK (status IN ('pending', 'confirmed', 'shipped', 'delivered', 'cancelled')),
    CONSTRAINT orders_ck_total_pos   CHECK (total >= 0)
);

CREATE INDEX idx_orders_tenant_user      ON orders (tenant_id, user_id)      WHERE deleted_at IS NULL;
CREATE INDEX idx_orders_tenant_status    ON orders (tenant_id, status)       WHERE deleted_at IS NULL;
CREATE INDEX idx_orders_created_at       ON orders (created_at DESC);

CREATE TRIGGER orders_audit
    AFTER INSERT OR UPDATE OR DELETE ON orders
    FOR EACH ROW EXECUTE FUNCTION record_audit_log();

CREATE TRIGGER orders_update_updated_at
    BEFORE UPDATE ON orders
    FOR EACH ROW EXECUTE FUNCTION update_updated_at();

-- ============================================================
-- Order Items
-- ============================================================
CREATE TABLE order_items (
    id          UUID          NOT NULL,
    order_id    UUID          NOT NULL,
    product_id  UUID          NOT NULL,
    quantity    INT           NOT NULL,
    unit_price  NUMERIC(19,4) NOT NULL, -- snapshotted at purchase time

    CONSTRAINT order_items_pk_id           PRIMARY KEY (id),
    CONSTRAINT order_items_fk_order_id     FOREIGN KEY (order_id)   REFERENCES orders   (id) ON DELETE CASCADE,
    CONSTRAINT order_items_fk_product_id   FOREIGN KEY (product_id) REFERENCES products (id) ON DELETE RESTRICT,
    CONSTRAINT order_items_ck_qty_positive CHECK (quantity > 0),
    CONSTRAINT order_items_ck_price_pos    CHECK (unit_price >= 0)
);

CREATE INDEX idx_order_items_order_id   ON order_items (order_id);
CREATE INDEX idx_order_items_product_id ON order_items (product_id);
```

**What this example demonstrates:**
- All naming conventions applied consistently (tables plural, columns singular, constraints `{table}_{type}_{col}`)
- `TIMESTAMPTZ` everywhere, never `TIMESTAMP`
- `NUMERIC(19,4)` for all money columns
- Partial unique indexes for soft-delete-aware uniqueness
- `ON DELETE` clause on every foreign key — intent explicit
- Snapshotted `unit_price` in `order_items` — intentional denormalization documented
- `JSONB` with GIN index for flexible product metadata and user preferences
- Triggers wired for `updated_at` and audit logging
- `tags TEXT[]` with GIN index for array containment queries
- RLS-ready `tenant_id` column on every tenant-scoped table
