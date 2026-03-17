# Schema Design — Soft Delete and Multi-Tenancy

## Soft Delete Pattern

Soft delete records by setting `deleted_at` rather than physically removing rows. Preserves audit history and allows recovery.

### DDL

```sql
ALTER TABLE users ADD COLUMN deleted_at TIMESTAMPTZ;

-- Partial unique index: enforce uniqueness only among active (non-deleted) rows
CREATE UNIQUE INDEX users_uq_email_active
    ON users (email) WHERE deleted_at IS NULL;

-- Efficient active-row filter
CREATE INDEX idx_users_active ON users (id) WHERE deleted_at IS NULL;
```

The partial unique index is critical: without it, re-registering a previously deleted email fails on the plain `UNIQUE (email)` constraint.

### Go Repository

```go
const (
    listActiveUsersSQL = `
        SELECT id, name, email, role, created_at, updated_at
        FROM users WHERE deleted_at IS NULL
        ORDER BY created_at DESC LIMIT $1 OFFSET $2`

    softDeleteUserSQL = `
        UPDATE users SET deleted_at = now()
        WHERE id = $1 AND deleted_at IS NULL`
)

func (r *sqlxUserRepository) Delete(ctx context.Context, id string) error {
    result, err := r.db.ExecContext(ctx, softDeleteUserSQL, id)
    if err != nil {
        return fmt.Errorf("soft delete user: %w", err)
    }
    n, _ := result.RowsAffected()
    if n == 0 {
        return domain.ErrNotFound.New(fmt.Errorf("user %s not found or already deleted", id))
    }
    return nil
}

// HardDelete permanently removes a user — use only for GDPR erasure.
func (r *sqlxUserRepository) HardDelete(ctx context.Context, id string) error {
    _, err := r.db.ExecContext(ctx, `DELETE FROM users WHERE id = $1`, id)
    return err
}
```

For the complete users table DDL (with constraints, triggers, RLS), see [schema-design-complete-example.md](schema-design-complete-example.md).

---

## Multi-Tenancy Patterns

### Pattern A — Shared Schema with tenant_id

Add `tenant_id UUID NOT NULL` to every tenant-scoped table. Simple; all tenants share the same tables.

```sql
ALTER TABLE orders ADD COLUMN tenant_id UUID NOT NULL;
ALTER TABLE orders
    ADD CONSTRAINT orders_fk_tenant_id FOREIGN KEY (tenant_id) REFERENCES tenants (id) ON DELETE RESTRICT;

-- Composite index: tenant_id first
CREATE INDEX idx_orders_tenant_id_created_at ON orders (tenant_id, created_at DESC);
```

**Every query must include `WHERE tenant_id = $1`** — the application enforces isolation.

### Pattern B — Row-Level Security (RLS)

RLS enforces tenant isolation at the database level — even if application code omits the filter.

```sql
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders FORCE ROW LEVEL SECURITY;

CREATE POLICY orders_tenant_isolation ON orders
    USING (tenant_id = current_setting('app.current_tenant_id')::uuid);
```

The middleware reads `X-Tenant-ID` header, opens a `db.Conn`, calls `SET LOCAL app.current_tenant_id = $1` (parameterized — never format directly into SQL), stores the conn in context, then calls `c.Next()`. Validate the tenant ID as a UUID before use.

> For the complete Gin RLS middleware with transaction lifecycle, see [row-level-security-go-integration-and-pitfalls.md](row-level-security-go-integration-and-pitfalls.md).

### Pattern C — Schema-per-Tenant

Each tenant gets a dedicated PostgreSQL schema (`tenant_abc.orders`). Provides complete isolation. Trade-offs: schema proliferation (100+ schemas), complex migrations (must run against every schema), higher operational overhead. Use only when strong data isolation is a contractual or compliance requirement.

---

## Audit Trail Pattern

### Audit Log Table

```sql
CREATE TABLE audit_log (
    id         BIGINT      GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    table_name TEXT        NOT NULL,
    record_id  UUID        NOT NULL,
    operation  TEXT        NOT NULL CHECK (operation IN ('INSERT', 'UPDATE', 'DELETE')),
    changed_by UUID,
    changed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    old_values JSONB,
    new_values JSONB
);

CREATE INDEX idx_audit_log_table_record ON audit_log (table_name, record_id);
CREATE INDEX idx_audit_log_changed_at   ON audit_log (changed_at DESC);
```

### Trigger for Automatic Audit Logging

```sql
CREATE OR REPLACE FUNCTION record_audit_log() RETURNS TRIGGER AS $$
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

CREATE TRIGGER orders_audit
    AFTER INSERT OR UPDATE OR DELETE ON orders
    FOR EACH ROW EXECUTE FUNCTION record_audit_log();
```

**Caution:** `to_jsonb(NEW)` captures all columns including sensitive fields like `password_hash`. Exclude sensitive tables or sanitize in the trigger.
