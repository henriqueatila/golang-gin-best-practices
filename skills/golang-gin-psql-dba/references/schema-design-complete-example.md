# Schema Design — Complete E-Commerce Schema Example

Full DDL for tenants, users, products, orders, order_items with all conventions applied: naming, constraints, indexes, triggers, soft delete, and audit logging.

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;
CREATE EXTENSION IF NOT EXISTS btree_gist;
CREATE OR REPLACE FUNCTION update_updated_at() RETURNS TRIGGER AS $$
BEGIN NEW.updated_at = now(); RETURN NEW; END;
$$ LANGUAGE plpgsql;

-- Tenants
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
    BEFORE UPDATE ON tenants FOR EACH ROW EXECUTE FUNCTION update_updated_at();

-- Users
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
    CONSTRAINT users_pk_id        PRIMARY KEY (id),
    CONSTRAINT users_fk_tenant_id FOREIGN KEY (tenant_id) REFERENCES tenants (id) ON DELETE RESTRICT,
    CONSTRAINT users_ck_role      CHECK (role IN ('admin', 'user', 'viewer')),
    CONSTRAINT users_ck_name_len  CHECK (char_length(name) BETWEEN 1 AND 200),
    CONSTRAINT users_ck_email_fmt CHECK (char_length(email) BETWEEN 3 AND 255)
);
CREATE UNIQUE INDEX users_uq_tenant_email_active ON users (tenant_id, email) WHERE deleted_at IS NULL;
CREATE INDEX idx_users_tenant_id                 ON users (tenant_id)        WHERE deleted_at IS NULL;
CREATE INDEX idx_users_created_at                ON users (created_at DESC);
CREATE TRIGGER users_update_updated_at
    BEFORE UPDATE ON users FOR EACH ROW EXECUTE FUNCTION update_updated_at();

-- Products
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
    CONSTRAINT products_pk_id             PRIMARY KEY (id),
    CONSTRAINT products_fk_tenant_id      FOREIGN KEY (tenant_id) REFERENCES tenants (id) ON DELETE RESTRICT,
    CONSTRAINT products_ck_price_positive CHECK (price >= 0),
    CONSTRAINT products_ck_stock_positive CHECK (stock >= 0),
    CONSTRAINT products_ck_name_len       CHECK (char_length(name) BETWEEN 1 AND 300),
    CONSTRAINT products_ck_slug_fmt       CHECK (slug ~ '^[a-z0-9-]+$')
);
CREATE UNIQUE INDEX products_uq_tenant_slug_active ON products (tenant_id, slug) WHERE deleted_at IS NULL;
CREATE INDEX idx_products_tenant_active            ON products (tenant_id)       WHERE deleted_at IS NULL;
CREATE INDEX idx_products_tags                     ON products USING gin (tags);
CREATE INDEX idx_products_metadata                 ON products USING gin (metadata jsonb_path_ops);
CREATE TRIGGER products_update_updated_at
    BEFORE UPDATE ON products FOR EACH ROW EXECUTE FUNCTION update_updated_at();

-- Orders
CREATE TABLE orders (
    id         UUID          NOT NULL,
    tenant_id  UUID          NOT NULL,
    user_id    UUID          NOT NULL,
    status     TEXT          NOT NULL DEFAULT 'pending',
    total      NUMERIC(19,4) NOT NULL DEFAULT 0,
    coupon_id  UUID,
    notes      TEXT,
    created_at TIMESTAMPTZ   NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ   NOT NULL DEFAULT now(),
    deleted_at TIMESTAMPTZ,
    CONSTRAINT orders_pk_id        PRIMARY KEY (id),
    CONSTRAINT orders_fk_tenant_id FOREIGN KEY (tenant_id) REFERENCES tenants (id) ON DELETE RESTRICT,
    CONSTRAINT orders_fk_user_id   FOREIGN KEY (user_id)   REFERENCES users   (id) ON DELETE RESTRICT,
    CONSTRAINT orders_fk_coupon_id FOREIGN KEY (coupon_id) REFERENCES coupons (id) ON DELETE SET NULL,
    CONSTRAINT orders_ck_status    CHECK (status IN ('pending','confirmed','shipped','delivered','cancelled')),
    CONSTRAINT orders_ck_total_pos CHECK (total >= 0)
);
CREATE INDEX idx_orders_tenant_user   ON orders (tenant_id, user_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_orders_tenant_status ON orders (tenant_id, status)  WHERE deleted_at IS NULL;
CREATE INDEX idx_orders_created_at    ON orders (created_at DESC);
CREATE TRIGGER orders_audit
    AFTER INSERT OR UPDATE OR DELETE ON orders
    FOR EACH ROW EXECUTE FUNCTION record_audit_log();
CREATE TRIGGER orders_update_updated_at
    BEFORE UPDATE ON orders FOR EACH ROW EXECUTE FUNCTION update_updated_at();

-- Order Items
CREATE TABLE order_items (
    id         UUID          NOT NULL,
    order_id   UUID          NOT NULL,
    product_id UUID          NOT NULL,
    quantity   INT           NOT NULL,
    unit_price NUMERIC(19,4) NOT NULL,  -- snapshotted at purchase time
    CONSTRAINT order_items_pk_id           PRIMARY KEY (id),
    CONSTRAINT order_items_fk_order_id     FOREIGN KEY (order_id)   REFERENCES orders   (id) ON DELETE CASCADE,
    CONSTRAINT order_items_fk_product_id   FOREIGN KEY (product_id) REFERENCES products (id) ON DELETE RESTRICT,
    CONSTRAINT order_items_ck_qty_positive CHECK (quantity > 0),
    CONSTRAINT order_items_ck_price_pos    CHECK (unit_price >= 0)
);
CREATE INDEX idx_order_items_order_id   ON order_items (order_id);
CREATE INDEX idx_order_items_product_id ON order_items (product_id);
```

**What this demonstrates:**
- All naming conventions applied consistently
- `TIMESTAMPTZ` everywhere, never `TIMESTAMP`
- `NUMERIC(19,4)` for all money columns
- Partial unique indexes for soft-delete-aware uniqueness
- `ON DELETE` clause on every foreign key — intent explicit
- Snapshotted `unit_price` in `order_items` — intentional denormalization
- `JSONB` with GIN index for flexible metadata and preferences
- Triggers wired for `updated_at` and audit logging
- `tags TEXT[]` with GIN index for array containment queries
- RLS-ready `tenant_id` column on every tenant-scoped table
