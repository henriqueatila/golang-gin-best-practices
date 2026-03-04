# Row-Level Security Reference

PostgreSQL Row-Level Security (RLS) lets the database engine enforce data visibility rules at the row level — before any application logic runs. Instead of scattering `WHERE tenant_id = $1` clauses across every repository method, you configure policies once on the table and set a session variable per request. Every `SELECT`, `INSERT`, `UPDATE`, and `DELETE` then filters automatically. This is the right model for multi-tenant SaaS, compliance-driven data isolation (HIPAA, SOC 2), and any scenario where the database must be a hard boundary — not just an advisory one. All Go examples use `sqlx`, `log/slog`, and `context.Context`. No GORM.

## Table of Contents

1. [RLS Overview](#rls-overview)
2. [Basic Setup](#basic-setup)
3. [Policy Types](#policy-types)
4. [Multi-Tenant RLS Pattern](#multi-tenant-rls-pattern)
5. [Role-Based RLS](#role-based-rls)
6. [Go Integration Pattern](#go-integration-pattern)
7. [Performance Considerations](#performance-considerations)
8. [Testing RLS](#testing-rls)
9. [Common Pitfalls](#common-pitfalls)

---

## RLS Overview

RLS (PostgreSQL 9.5+) attaches filter predicates directly to tables. When a query hits an RLS-enabled table, PostgreSQL rewrites it to include the policy `USING` or `WITH CHECK` expression before execution — invisible to the application.

```sql
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON users
    USING (tenant_id = current_setting('app.current_tenant_id')::uuid);
-- Every SELECT/UPDATE/DELETE now behaves as if WHERE tenant_id = $current was appended.
```

**Use RLS when:** multi-tenant SaaS (shared schema, `tenant_id` separation), compliance requirements (HIPAA, SOC 2, GDPR), or audit-critical isolation where the DB must be a hard boundary.

**Skip RLS when:** single-tenant apps (unnecessary complexity), or the app always connects as a superuser (RLS is bypassed — see FORCE below).

---

## Basic Setup

```sql
ALTER TABLE users ENABLE ROW LEVEL SECURITY;   -- non-owners/non-superusers now filtered
ALTER TABLE users FORCE ROW LEVEL SECURITY;    -- also applies to the table owner
```

`FORCE ROW LEVEL SECURITY` is required when the app connects as the schema owner. Without it the owner bypasses all policies.

**Default deny:** with RLS enabled and no policies defined, all access returns zero rows (no error). This is intentional — no accidental leaks during partial policy deployments.

```sql
CREATE POLICY policy_name ON table_name
    [AS { PERMISSIVE | RESTRICTIVE }]
    [FOR { ALL | SELECT | INSERT | UPDATE | DELETE }]
    [TO { role_name | PUBLIC }]
    [USING ( using_expression )]
    [WITH CHECK ( check_expression )];
```

| Clause | Applied to | Purpose |
|---|---|---|
| `USING` | `SELECT`, `UPDATE`, `DELETE` | Filters rows visible to the query |
| `WITH CHECK` | `INSERT`, `UPDATE` | Validates rows being written |

---

## Policy Types

Each command type uses different clauses:

| Command | `USING` | `WITH CHECK` |
|---|---|---|
| `SELECT` | Filters rows returned | — |
| `INSERT` | — | Validates new row before insert |
| `UPDATE` | Filters rows that can be targeted | Validates row after update |
| `DELETE` | Filters rows eligible for deletion | — |
| `ALL` | Applied to SELECT/UPDATE/DELETE | Applied to INSERT/UPDATE |

```sql
-- SELECT: filter visible rows
CREATE POLICY select_tenant ON users FOR SELECT
    USING (tenant_id = current_setting('app.current_tenant_id')::uuid);

-- INSERT: reject inserts for wrong tenant
CREATE POLICY insert_tenant ON users FOR INSERT
    WITH CHECK (tenant_id = current_setting('app.current_tenant_id')::uuid);

-- UPDATE: only target own tenant's rows, and prevent moving to another tenant
CREATE POLICY update_tenant ON users FOR UPDATE
    USING      (tenant_id = current_setting('app.current_tenant_id')::uuid)
    WITH CHECK (tenant_id = current_setting('app.current_tenant_id')::uuid);

-- ALL: shorthand when USING and WITH CHECK expressions are identical
CREATE POLICY tenant_isolation ON users
    USING      (tenant_id = current_setting('app.current_tenant_id')::uuid)
    WITH CHECK (tenant_id = current_setting('app.current_tenant_id')::uuid);
```

### PERMISSIVE vs RESTRICTIVE

| Type | Combines policies with | Default | Use for |
|---|---|---|---|
| `PERMISSIVE` | **OR** | Yes | Normal grants — row visible if ANY policy allows |
| `RESTRICTIVE` | **AND** with all permissive | No | Hard limits — e.g., `deleted_at IS NULL`, tenant boundary |

```sql
CREATE POLICY see_own    AS PERMISSIVE   ON documents FOR SELECT
    USING (owner_id = current_setting('app.current_user_id')::uuid);
CREATE POLICY see_shared AS PERMISSIVE   ON documents FOR SELECT
    USING (shared = true);
CREATE POLICY hide_deleted AS RESTRICTIVE ON documents FOR SELECT
    USING (deleted_at IS NULL);
-- Result: (see_own OR see_shared) AND hide_deleted
```

---

## Multi-Tenant RLS Pattern

### Session variable approach

PostgreSQL `current_setting()` reads a session-local variable set by the application. The variable is scoped to the transaction when set with `SET LOCAL`, which is the correct approach for connection pools where connections are reused.

```sql
-- Set in the transaction (cleared automatically on COMMIT/ROLLBACK)
SET LOCAL app.current_tenant_id = 'a1b2c3d4-e5f6-7890-abcd-ef1234567890';

-- Read in policy expression
current_setting('app.current_tenant_id', true)::uuid
-- Second arg 'true' = return NULL instead of error if variable is not set
```

### Complete DDL: multi-tenant users table

```sql
CREATE ROLE app_user LOGIN PASSWORD 'secret';

CREATE TABLE users (
    id         UUID        NOT NULL DEFAULT gen_random_uuid(),
    tenant_id  UUID        NOT NULL,
    email      TEXT        NOT NULL,
    name       TEXT        NOT NULL,
    role       TEXT        NOT NULL DEFAULT 'member'
                           CHECK (role IN ('admin', 'member', 'viewer')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at TIMESTAMPTZ,
    CONSTRAINT users_pk_id    PRIMARY KEY (id),
    CONSTRAINT users_uq_email UNIQUE (tenant_id, email)
);

CREATE INDEX idx_users_tenant_id     ON users (tenant_id);
CREATE INDEX idx_users_tenant_active ON users (tenant_id, email) WHERE deleted_at IS NULL;

GRANT SELECT, INSERT, UPDATE, DELETE ON users TO app_user;

ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE users FORCE ROW LEVEL SECURITY;

-- Tenant isolation (ALL commands)
CREATE POLICY users_tenant_isolation ON users
    USING  (tenant_id = current_setting('app.current_tenant_id', true)::uuid)
    WITH CHECK (tenant_id = current_setting('app.current_tenant_id', true)::uuid);

-- Soft-delete filter (RESTRICTIVE — always AND'd with permissive policies)
CREATE POLICY users_hide_deleted AS RESTRICTIVE ON users
    FOR SELECT USING (deleted_at IS NULL);
```

### Go repository — transparent with RLS

Omit `tenant_id` from every WHERE clause. RLS applies it automatically at the PostgreSQL level.

```go
// List returns users visible to the current tenant — no WHERE tenant_id needed.
func (r *UserRepository) List(ctx context.Context) ([]User, error) {
    var users []User
    err := r.db.SelectContext(ctx, &users,
        `SELECT id, tenant_id, email, name, role FROM users ORDER BY created_at DESC`)
    if err != nil {
        return nil, fmt.Errorf("repository: list users: %w", err)
    }
    slog.DebugContext(ctx, "users listed", "count", len(users))
    return users, nil
}

// GetByID returns a single user. RLS prevents cross-tenant access by ID lookup.
func (r *UserRepository) GetByID(ctx context.Context, id string) (User, error) {
    var u User
    err := r.db.GetContext(ctx, &u,
        `SELECT id, tenant_id, email, name, role FROM users WHERE id = $1`, id)
    if err != nil {
        return User{}, fmt.Errorf("repository: get user %s: %w", id, err)
    }
    return u, nil
}
```

### Transaction handling: SET LOCAL

Always use `SET LOCAL` inside a transaction — never bare `SET`, which persists on the connection and leaks across pool reuse.

```go
func SetTenantContext(ctx context.Context, tx *sqlx.Tx, tenantID string) error {
    _, err := tx.ExecContext(ctx, `SET LOCAL app.current_tenant_id = $1`, tenantID)
    if err != nil {
        return fmt.Errorf("rls: set tenant context: %w", err)
    }
    return nil
}
```

---

## Role-Based RLS

Combine role-based visibility with tenant isolation by reading a second session variable.

### Combining tenant_id + role

Three policies achieve the full picture: a RESTRICTIVE tenant boundary (always applied), and two PERMISSIVE policies — admin sees all rows within the tenant, member sees only their own row.

```sql
CREATE POLICY tenant_boundary AS RESTRICTIVE ON users
    USING  (tenant_id = current_setting('app.current_tenant_id', true)::uuid)
    WITH CHECK (tenant_id = current_setting('app.current_tenant_id', true)::uuid);

CREATE POLICY admin_full_access AS PERMISSIVE ON users FOR SELECT
    USING (current_setting('app.current_role', true) = 'admin');

CREATE POLICY member_own_row AS PERMISSIVE ON users FOR SELECT
    USING (id = current_setting('app.current_user_id', true)::uuid);
```

Result: `(admin_full_access OR member_own_row) AND tenant_boundary` — neither role can ever cross tenants.

---

## Go Integration Pattern

### Gin middleware: complete request flow

Opens a transaction, calls `SET LOCAL`, stores the tx in Gin context, commits or rolls back on response status. Handlers retrieve the tx from context — never the pool `db` directly.

```go
func RLSMiddleware(db *sqlx.DB) gin.HandlerFunc {
    return func(c *gin.Context) {
        tenantID, ok := c.Get("tenant_id") // set by a preceding JWT middleware
        if !ok {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "tenant context missing"})
            return
        }
        tx, err := db.BeginTxx(c.Request.Context(), nil)
        if err != nil {
            slog.ErrorContext(c.Request.Context(), "rls: begin tx", "error", err)
            c.AbortWithStatusJSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
            return
        }
        if _, err := tx.ExecContext(c.Request.Context(),
            `SET LOCAL app.current_tenant_id = $1`, tenantID.(string)); err != nil {
            _ = tx.Rollback()
            slog.ErrorContext(c.Request.Context(), "rls: set tenant", "error", err)
            c.AbortWithStatusJSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
            return
        }
        c.Set("db_tx", tx)
        c.Next()
        if c.Writer.Status() >= 400 {
            _ = tx.Rollback()
        } else if err := tx.Commit(); err != nil {
            slog.ErrorContext(c.Request.Context(), "rls: commit", "error", err)
        }
    }
}
```

### Error handling: missing tenant context

`current_setting('app.current_tenant_id', true)` with the `true` flag returns `NULL` (not an error) when unset. `tenant_id = NULL` is always `false`, so RLS silently returns zero rows — a silent failure that is hard to diagnose. Detect it early:

```go
// ValidateTenantContext asserts the session variable is set inside the tx.
func ValidateTenantContext(ctx context.Context, tx *sqlx.Tx) error {
    var tid *string
    err := tx.QueryRowContext(ctx,
        `SELECT current_setting('app.current_tenant_id', true)`).Scan(&tid)
    if err != nil {
        return fmt.Errorf("rls: query tenant context: %w", err)
    }
    if tid == nil || *tid == "" {
        return fmt.Errorf("rls: tenant context not set in transaction")
    }
    return nil
}
```

---

## Performance Considerations

### How RLS affects query plans

PostgreSQL inlines the policy expression — it appears in `EXPLAIN` output exactly as if you had written `WHERE tenant_id = $1`. The planner treats it identically, including index usage. With a B-tree index on `tenant_id`, overhead is 1–3% on indexed queries; the cost is the index lookup, not the policy evaluation itself.

```sql
-- Required: composite indexes leading with tenant_id
CREATE INDEX idx_users_tenant_id         ON users (tenant_id);
CREATE INDEX idx_orders_tenant_created   ON orders (tenant_id, created_at DESC);
```

### When RLS becomes expensive

- **Function calls in USING** — `USING (get_tenant())` evaluates per-row if not inlined. Use `current_setting()` directly.
- **Many PERMISSIVE policies** — each adds an OR branch. Keep 1–3 per command type.
- **JOIN-based policies** — join to a membership table triggers a subquery per query. Materialize the value into the session variable in middleware instead.

### Monitoring

Compare `pg_stat_statements` mean execution times between superuser (bypasses RLS) and `app_user` for the same query. Any gap beyond ~3% on indexed columns means the policy is not using the index — confirm with `EXPLAIN (ANALYZE, BUFFERS)` and verify `Index Scan` appears, not `Seq Scan`.

---

## Testing RLS

### Testing policies with SET ROLE and SET LOCAL

```sql
SET ROLE app_user;
BEGIN;
SET LOCAL app.current_tenant_id = 'tenant-a-uuid';
SELECT count(*) FROM users;                                    -- only tenant-A rows
SELECT * FROM users WHERE id = 'tenant-b-user-uuid';           -- must return 0 rows
ROLLBACK;
RESET ROLE;
```

### Go integration test: verify tenant isolation

```go
func TestRLS_TenantIsolation(t *testing.T) {
    // db must connect as app_user (non-superuser) — superuser bypasses RLS
    db := testDB(t)

    tenantA := "aaaaaaaa-0000-0000-0000-000000000001"
    tenantB := "bbbbbbbb-0000-0000-0000-000000000002"
    seedUser(t, db, tenantA, "alice@a.com")
    seedUser(t, db, tenantB, "bob@b.com")

    t.Run("tenant A sees only its rows", func(t *testing.T) {
        tx := beginWithTenant(t, db, tenantA)
        defer tx.Rollback()
        var count int
        if err := tx.QueryRowContext(context.Background(),
            `SELECT count(*) FROM users`).Scan(&count); err != nil {
            t.Fatal(err)
        }
        if count != 1 {
            t.Fatalf("expected 1 row for tenant A, got %d", count)
        }
    })

    t.Run("tenant A cannot read tenant B row by ID", func(t *testing.T) {
        bobID := getUserID(t, db, tenantB, "bob@b.com") // fetched via superuser conn
        tx := beginWithTenant(t, db, tenantA)
        defer tx.Rollback()
        var u struct{ ID string `db:"id"` }
        err := tx.QueryRowxContext(context.Background(),
            `SELECT id FROM users WHERE id = $1`, bobID).StructScan(&u)
        if err == nil {
            t.Fatal("expected no row — RLS isolation breach")
        }
    })
}

func beginWithTenant(t *testing.T, db *sqlx.DB, tenantID string) *sqlx.Tx {
    t.Helper()
    tx, err := db.Beginx()
    if err != nil {
        t.Fatal(err)
    }
    if _, err := tx.ExecContext(context.Background(),
        `SET LOCAL app.current_tenant_id = $1`, tenantID); err != nil {
        t.Fatal(err)
    }
    return tx
}
```

### Using testcontainers for RLS tests

RLS requires a real PostgreSQL instance — no SQLite mock will work. Use `testcontainers-go` to spin up a real Postgres container, apply migrations (which include `CREATE POLICY`), and connect as `app_user` (not the superuser). The `gin-testing` skill has the full testcontainers wiring.

Critical: if the test connection uses the `postgres` superuser, RLS is bypassed and isolation tests pass trivially without actually enforcing anything.

---

## Common Pitfalls

### Forgetting FORCE ROW LEVEL SECURITY

If the application connects as the table owner or a superuser, `ENABLE ROW LEVEL SECURITY` alone does nothing. The owner is exempt.

```sql
-- Without this, the table owner bypasses all policies
ALTER TABLE users FORCE ROW LEVEL SECURITY;
```

Verify your app's database role is NOT a superuser and is NOT the table owner.

### Not setting the session variable (silent empty results)

If `SET LOCAL app.current_tenant_id` is never called, `current_setting(..., true)` returns `NULL` and `tenant_id = NULL` is always false — the query silently returns zero rows instead of an error. Detect this in middleware using `ValidateTenantContext` (see above) and write a negative test that confirms a request without the session variable set returns zero rows, not data.

### PERMISSIVE vs RESTRICTIVE confusion

Multiple `PERMISSIVE` policies are OR'd together. Adding a "security" policy as PERMISSIVE widens access, not restricts it. Use `AS RESTRICTIVE` for must-always-apply filters like `deleted_at IS NULL` or the tenant boundary.

```sql
-- WRONG: this WIDENS access (OR logic)
CREATE POLICY hide_deleted AS PERMISSIVE ON users
    USING (deleted_at IS NULL);

-- CORRECT: this NARROWS access (AND with all permissive policies)
CREATE POLICY hide_deleted AS RESTRICTIVE ON users
    USING (deleted_at IS NULL);
```

### Backup and restore

`pg_dump` includes RLS policies — restoring re-creates them. No special handling needed.

- Never use `pg_dump --no-policies` on RLS-protected schemas.
- After restore, confirm `FORCE ROW LEVEL SECURITY` is still set and the app role is non-superuser.

### Migrations on RLS-protected tables

DDL (ALTER TABLE, CREATE INDEX) is not filtered by RLS — run migrations as the schema owner freely. But backfill DML run as `app_user` will be filtered by the active policy. For backfills, either run as the schema owner or temporarily disable RLS:

```sql
ALTER TABLE users DISABLE ROW LEVEL SECURITY;
UPDATE users SET new_column = derive_value(id);
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE users FORCE ROW LEVEL SECURITY;
```

Drop policies before dropping columns they reference to avoid dependency errors:

```sql
DROP POLICY IF EXISTS users_tenant_isolation ON users;
ALTER TABLE users DROP COLUMN tenant_id;
```
