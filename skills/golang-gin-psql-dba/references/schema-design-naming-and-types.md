# Schema Design — Naming Conventions and Data Types

## Naming Conventions

### Tables
- **Plural snake_case:** `users`, `orders`, `order_items`, `product_categories`
- Junction tables: combine both sides alphabetically — `order_items`, `user_roles`, `product_tags`
- Avoid abbreviations unless universal (`url`, `ip`, `uuid`)

### Columns
- **Singular snake_case:** `user_id`, `created_at`, `is_verified`
- Foreign key columns: `{referenced_table_singular}_id` — e.g., `user_id`, `tenant_id`
- Timestamps: always `created_at`, `updated_at`, `deleted_at`

### Indexes
Pattern: `idx_{table}_{columns}`

```sql
CREATE INDEX idx_users_email          ON users (email);
CREATE INDEX idx_orders_user_id       ON orders (user_id);
CREATE INDEX idx_orders_tenant_status ON orders (tenant_id, status);  -- most selective first
```

### Constraints
Pattern: `{table}_{type}_{columns}`

| Type | Suffix | Example |
|------|--------|---------|
| Primary key | `_pk_id` | `users_pk_id` |
| Unique | `_uq_{col}` | `users_uq_email` |
| Foreign key | `_fk_{col}` | `orders_fk_user_id` |
| Check | `_ck_{col}` | `products_ck_price_positive` |

```sql
ALTER TABLE users
    ADD CONSTRAINT users_pk_id    PRIMARY KEY (id),
    ADD CONSTRAINT users_uq_email UNIQUE (email),
    ADD CONSTRAINT users_ck_role  CHECK (role IN ('admin', 'user', 'viewer'));
```

---

## Data Type Selection

### UUID vs BIGINT Primary Keys

| Concern | UUID (v4) | UUIDv7 | BIGINT IDENTITY |
|---------|-----------|--------|-----------------|
| Sortable by creation time | No (random) | Yes (time-prefixed) | Yes |
| B-tree index fragmentation | High | Low | Low |
| Expose sequential IDs to clients | No | No | Yes (enumerable) |
| Storage | 16 bytes | 16 bytes | 8 bytes |

**Recommendation:** Use UUIDv7 for non-enumerable IDs with good index performance. Use `BIGINT GENERATED ALWAYS AS IDENTITY` for internal tables (audit logs, event queues). Avoid random UUIDv4 for large tables.

```sql
-- UUIDv7 (time-sortable)
id UUID PRIMARY KEY DEFAULT gen_random_uuid()

-- BIGINT IDENTITY (internal tables)
id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY
```

```go
func newID() string {
    id, err := uuid.NewV7()
    if err != nil {
        return uuid.Must(uuid.NewRandom()).String()
    }
    return id.String()
}
```

### Timestamps — Always TIMESTAMPTZ

Never use `TIMESTAMP` (no timezone). `TIMESTAMPTZ` stores UTC and converts on read.

```sql
created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
deleted_at TIMESTAMPTZ  -- nullable: NULL = active
```

### TEXT vs VARCHAR(n)

Prefer `TEXT` with a `CHECK` constraint — constraints can be altered without rewriting the column.

```sql
name  TEXT NOT NULL CHECK (char_length(name)  BETWEEN 1 AND 200),
email TEXT NOT NULL CHECK (char_length(email) BETWEEN 3 AND 255 AND email LIKE '%@%'),
slug  TEXT NOT NULL CHECK (slug ~ '^[a-z0-9-]+$'),
```

### NUMERIC for Money

Never store money in `FLOAT`, `REAL`, or `DOUBLE PRECISION` — binary floats introduce rounding errors.

```sql
price    NUMERIC(19,4) NOT NULL DEFAULT 0,
tax_rate NUMERIC(5,4)  NOT NULL DEFAULT 0,
```

```go
import "github.com/shopspring/decimal"

type productRow struct {
    Price decimal.Decimal `db:"price"`
}
```

### JSONB

Use when shape varies per row or you need containment queries (`@>`, `<@`, `?`).

```sql
preferences JSONB NOT NULL DEFAULT '{}',
CREATE INDEX idx_users_preferences ON users USING gin (preferences jsonb_path_ops);
```

**Do NOT use JSONB** when you filter/sort/join on a field regularly — extract it into a proper column.

### Other Types

```sql
-- Boolean: always NOT NULL DEFAULT false
active BOOLEAN NOT NULL DEFAULT false,

-- Network
ip_address   INET,
network_cidr CIDR,

-- Arrays (small, bounded lists)
tags TEXT[] NOT NULL DEFAULT '{}',
CREATE INDEX idx_posts_tags ON posts USING gin (tags);
```

```go
import "github.com/lib/pq"
type postRow struct {
    Tags pq.StringArray `db:"tags"`
}
```
