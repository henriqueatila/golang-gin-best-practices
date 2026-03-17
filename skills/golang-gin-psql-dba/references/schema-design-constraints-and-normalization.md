# Schema Design — Constraints and Normalization

## Constraints

### Full Constraint DDL Example

```sql
CREATE TABLE users (
    id            UUID        NOT NULL,
    email         TEXT        NOT NULL,
    name          TEXT        NOT NULL,
    role          TEXT        NOT NULL DEFAULT 'user',
    active        BOOLEAN     NOT NULL DEFAULT false,
    bio           TEXT,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at    TIMESTAMPTZ,

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
| Parent deletion blocked if children exist (users → tenant) | `ON DELETE RESTRICT` | Error if parent has children |
| Child rows remain valid without parent; FK nullable | `ON DELETE SET NULL` | Set FK column to NULL |
| Child rows remain valid with a default parent | `ON DELETE SET DEFAULT` | Set FK column to default value |

```sql
-- order_items deleted when order is deleted
CONSTRAINT order_items_fk_order_id
    FOREIGN KEY (order_id) REFERENCES orders (id) ON DELETE CASCADE,

-- Cannot delete a user who has orders
CONSTRAINT orders_fk_user_id
    FOREIGN KEY (user_id) REFERENCES users (id) ON DELETE RESTRICT,

-- Coupon optional; if coupon deleted, column becomes NULL
CONSTRAINT orders_fk_coupon_id
    FOREIGN KEY (coupon_id) REFERENCES coupons (id) ON DELETE SET NULL
```

**Always specify `ON DELETE`** — omitting it defaults to RESTRICT but is ambiguous to readers.

### Exclusion Constraints for Ranges

Prevent overlapping date ranges (booking systems, subscription periods):

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

CREATE TABLE room_bookings (
    id      UUID      PRIMARY KEY DEFAULT gen_random_uuid(),
    room_id UUID      NOT NULL,
    period  TSTZRANGE NOT NULL,
    CONSTRAINT room_bookings_no_overlap
        EXCLUDE USING gist (room_id WITH =, period WITH &&)
);
```

---

## Normalization Guidance

### When to Normalize (3NF)

Normalize by default. Normalize when:
- The same data appears in multiple rows (city/state repeated on every user)
- Data is updated independently (product name stored redundantly in orders)
- Referential integrity matters (roles defined in a table, not free-text strings)

### When to Denormalize

Denormalize intentionally and document why:
- **Read-heavy aggregates** that are expensive to recompute (running totals, materialized counts)
- **Historical snapshots** that must not change (order `unit_price` at purchase time)
- **JSONB for flexible attributes** where shape varies per row

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
