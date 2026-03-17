# Row-Level Security — Go Integration, Testing, and Common Pitfalls

## Go Integration Pattern

### Gin middleware: complete request flow

Opens a transaction, calls `SET LOCAL`, stores the tx in Gin context, commits or rolls back on response status.

```go
func RLSMiddleware(db *sqlx.DB) gin.HandlerFunc {
    return func(c *gin.Context) {
        tenantID, ok := c.Get("tenant_id") // set by preceding JWT middleware
        if !ok { c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "tenant context missing"}); return }
        tx, err := db.BeginTxx(c.Request.Context(), nil)
        if err != nil {
            slog.ErrorContext(c.Request.Context(), "rls: begin tx", "error", err)
            c.AbortWithStatusJSON(http.StatusInternalServerError, gin.H{"error": "internal error"}); return
        }
        if _, err := tx.ExecContext(c.Request.Context(), `SET LOCAL app.current_tenant_id = $1`, tenantID.(string)); err != nil {
            _ = tx.Rollback()
            slog.ErrorContext(c.Request.Context(), "rls: set tenant", "error", err)
            c.AbortWithStatusJSON(http.StatusInternalServerError, gin.H{"error": "internal error"}); return
        }
        c.Set("db_tx", tx)
        c.Next()
        if c.Writer.Status() >= 400 { _ = tx.Rollback() } else if err := tx.Commit(); err != nil {
            slog.ErrorContext(c.Request.Context(), "rls: commit", "error", err)
        }
    }
}
```

### Always use SET LOCAL — never bare SET

```go
func SetTenantContext(ctx context.Context, tx *sqlx.Tx, tenantID string) error {
    _, err := tx.ExecContext(ctx, `SET LOCAL app.current_tenant_id = $1`, tenantID)
    if err != nil {
        return fmt.Errorf("rls: set tenant context: %w", err)
    }
    return nil
}
```

Bare `SET` persists on the connection and leaks across connection pool reuse.

### Error handling: detect missing tenant context early

`current_setting('app.current_tenant_id', true)` returns `NULL` when unset. `tenant_id = NULL` is always false — silently returns zero rows.

```go
func ValidateTenantContext(ctx context.Context, tx *sqlx.Tx) error {
    var tid *string
    if err := tx.QueryRowContext(ctx, `SELECT current_setting('app.current_tenant_id', true)`).Scan(&tid); err != nil {
        return fmt.Errorf("rls: query tenant context: %w", err)
    }
    if tid == nil || *tid == "" { return fmt.Errorf("rls: tenant context not set in transaction") }
    return nil
}
```

---

## Performance Considerations

### How RLS affects query plans

PostgreSQL inlines the policy expression — visible in `EXPLAIN` as a `WHERE tenant_id = $1` filter. With a B-tree index on `tenant_id`, overhead is 1–3%. Required: composite indexes leading with `tenant_id` (e.g., `ON orders (tenant_id, created_at DESC)`).

### When RLS becomes expensive

- **Function calls in USING** — `USING (get_tenant())` evaluates per-row if not inlined. Use `current_setting()` directly.
- **Many PERMISSIVE policies** — each adds an OR branch. Keep 1–3 per command type.
- **JOIN-based policies** — join to a membership table triggers a subquery per query. Materialize the value into the session variable in middleware instead.

---

## Testing RLS

### SQL: SET ROLE and SET LOCAL

```sql
SET ROLE app_user;
BEGIN;
SET LOCAL app.current_tenant_id = 'tenant-a-uuid';
SELECT count(*) FROM users;                            -- only tenant-A rows
SELECT * FROM users WHERE id = 'tenant-b-user-uuid';  -- must return 0 rows
ROLLBACK;
RESET ROLE;
```

### Go: verify tenant isolation

```go
func TestRLS_TenantIsolation(t *testing.T) {
    db := testDB(t) // connects as app_user (non-superuser)
    seedUser(t, db, tenantA, "alice@a.com")
    seedUser(t, db, tenantB, "bob@b.com")

    t.Run("tenant A sees only its rows", func(t *testing.T) {
        tx := beginWithTenant(t, db, tenantA); defer tx.Rollback()
        var count int
        tx.QueryRowContext(context.Background(), `SELECT count(*) FROM users`).Scan(&count)
        if count != 1 { t.Fatalf("expected 1 row, got %d", count) }
    })
    t.Run("tenant A cannot read tenant B row by ID", func(t *testing.T) {
        bobID := getUserID(t, db, tenantB, "bob@b.com")
        tx := beginWithTenant(t, db, tenantA); defer tx.Rollback()
        var u struct{ ID string `db:"id"` }
        if err := tx.QueryRowxContext(context.Background(),
            `SELECT id FROM users WHERE id = $1`, bobID).StructScan(&u); err == nil {
            t.Fatal("RLS isolation breach")
        }
    })
}

func beginWithTenant(t *testing.T, db *sqlx.DB, tenantID string) *sqlx.Tx {
    t.Helper(); tx, _ := db.Beginx()
    tx.ExecContext(context.Background(), `SET LOCAL app.current_tenant_id = $1`, tenantID)
    return tx
}
```

**Critical:** superuser connections bypass RLS entirely — always use a non-superuser test connection.

## Common Pitfalls

### Forgetting FORCE ROW LEVEL SECURITY

```sql
-- Without this, the table owner bypasses all policies
ALTER TABLE users FORCE ROW LEVEL SECURITY;
```

### PERMISSIVE vs RESTRICTIVE confusion

```sql
-- WRONG: this WIDENS access (OR logic)
CREATE POLICY hide_deleted AS PERMISSIVE ON users USING (deleted_at IS NULL);

-- CORRECT: this NARROWS access (AND with all permissive policies)
CREATE POLICY hide_deleted AS RESTRICTIVE ON users USING (deleted_at IS NULL);
```

### Backup, restore, and migrations

`pg_dump` includes RLS policies — after restore, confirm `FORCE ROW LEVEL SECURITY` is still set. For migrations: run DDL as schema owner; for backfill DML, temporarily `DISABLE ROW LEVEL SECURITY`, run UPDATE, then `ENABLE` and `FORCE`. Drop policies before dropping referenced columns.
