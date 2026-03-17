# Row-Level Security — Setup, Policies, and Multi-Tenant Pattern

PostgreSQL RLS lets the database engine enforce data visibility rules at the row level — before any application logic runs. Instead of scattering `WHERE tenant_id = $1` across every repository method, you configure policies once on the table and set a session variable per request.

> All Go examples use `sqlx`, `log/slog`, and `context.Context`. No GORM.

## RLS Overview

RLS (PostgreSQL 9.5+) attaches filter predicates directly to tables. PostgreSQL rewrites every query to include the policy `USING` or `WITH CHECK` expression before execution.

```sql
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON users
    USING (tenant_id = current_setting('app.current_tenant_id')::uuid);
-- Every SELECT/UPDATE/DELETE now behaves as if WHERE tenant_id = $current was appended.
```

**Use RLS when:** multi-tenant SaaS (shared schema), compliance requirements (HIPAA, SOC 2, GDPR), or audit-critical isolation where the DB must be a hard boundary.

**Skip RLS when:** single-tenant apps, or the app always connects as a superuser (RLS is bypassed — see FORCE below).

---

## Basic Setup

```sql
ALTER TABLE users ENABLE ROW LEVEL SECURITY;   -- non-owners/non-superusers now filtered
ALTER TABLE users FORCE ROW LEVEL SECURITY;    -- also applies to the table owner
```

`FORCE ROW LEVEL SECURITY` is required when the app connects as the schema owner. Without it the owner bypasses all policies.

**Default deny:** with RLS enabled and no policies defined, all access returns zero rows (no error).

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

| Command | `USING` | `WITH CHECK` |
|---|---|---|
| `SELECT` | Filters rows returned | — |
| `INSERT` | — | Validates new row before insert |
| `UPDATE` | Filters rows that can be targeted | Validates row after update |
| `DELETE` | Filters rows eligible for deletion | — |
| `ALL` | Applied to SELECT/UPDATE/DELETE | Applied to INSERT/UPDATE |

```sql
-- FOR SELECT uses USING; FOR INSERT uses WITH CHECK; ALL (shorthand): both same expression
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

```sql
-- Set in the transaction (cleared automatically on COMMIT/ROLLBACK)
SET LOCAL app.current_tenant_id = 'a1b2c3d4-e5f6-7890-abcd-ef1234567890';

-- Read in policy expression — 'true' = return NULL instead of error if unset
current_setting('app.current_tenant_id', true)::uuid
```

### Complete DDL: multi-tenant users table

```sql
CREATE ROLE app_user LOGIN PASSWORD 'secret';
-- users table: id UUID PK, tenant_id UUID NOT NULL, email TEXT, name TEXT, role TEXT, timestamps, deleted_at
-- See schema-design-complete-example.md for full column list

CREATE INDEX idx_users_tenant_id     ON users (tenant_id);
CREATE INDEX idx_users_tenant_active ON users (tenant_id, email) WHERE deleted_at IS NULL;
GRANT SELECT, INSERT, UPDATE, DELETE ON users TO app_user;
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE users FORCE ROW LEVEL SECURITY;

CREATE POLICY users_tenant_isolation ON users
    USING  (tenant_id = current_setting('app.current_tenant_id', true)::uuid)
    WITH CHECK (tenant_id = current_setting('app.current_tenant_id', true)::uuid);

-- RESTRICTIVE: always AND'd with permissive policies
CREATE POLICY users_hide_deleted AS RESTRICTIVE ON users FOR SELECT USING (deleted_at IS NULL);
```

### Go repository — transparent with RLS

```go
// List returns users visible to the current tenant — no WHERE tenant_id needed.
func (r *UserRepository) List(ctx context.Context) ([]User, error) {
    var users []User
    err := r.db.SelectContext(ctx, &users,
        `SELECT id, tenant_id, email, name, role FROM users ORDER BY created_at DESC`)
    if err != nil {
        return nil, fmt.Errorf("repository: list users: %w", err)
    }
    return users, nil
}
```

### Role-Based RLS: combining tenant_id + role

```sql
CREATE POLICY tenant_boundary AS RESTRICTIVE ON users
    USING  (tenant_id = current_setting('app.current_tenant_id', true)::uuid)
    WITH CHECK (tenant_id = current_setting('app.current_tenant_id', true)::uuid);

CREATE POLICY admin_full_access AS PERMISSIVE ON users FOR SELECT
    USING (current_setting('app.current_role', true) = 'admin');

CREATE POLICY member_own_row AS PERMISSIVE ON users FOR SELECT
    USING (id = current_setting('app.current_user_id', true)::uuid);
-- Result: (admin_full_access OR member_own_row) AND tenant_boundary
```

> For RLS testing patterns (SQL SET ROLE, Go isolation tests, testcontainers wiring): see [row-level-security-go-integration-and-pitfalls.md](row-level-security-go-integration-and-pitfalls.md).
