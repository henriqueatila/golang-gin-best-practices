# Replication and HA — Logical Replication

## When to Use

- Replicate only selected tables (not the entire cluster)
- Replicate between different PostgreSQL major versions (upgrade path)
- Feed changes to a different database system (Kafka, ClickHouse, read replica with different schema)
- Keep a writable subscriber (analytics database enriched with additional data)

## Publication and Subscription Model

The **publisher** (source) creates a publication that lists which tables to stream. The **subscriber** (destination) creates a subscription pointing at the publisher's connection string.

## CREATE PUBLICATION

```sql
-- On the source database (publisher)

-- Replicate only the orders table
CREATE PUBLICATION orders_pub FOR TABLE orders;

-- Replicate multiple tables
CREATE PUBLICATION app_pub FOR TABLE users, orders, products;

-- Replicate all current and future tables (use cautiously)
-- CREATE PUBLICATION all_pub FOR ALL TABLES;
```

Set `wal_level = logical` in the publisher's `postgresql.conf` (requires restart).

## CREATE SUBSCRIPTION

```sql
-- On the destination database (subscriber)

-- Tables must already exist with compatible schema
CREATE TABLE orders (
    id          UUID        PRIMARY KEY,
    user_id     UUID        NOT NULL,
    total       NUMERIC(19,4) NOT NULL,
    status      TEXT        NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL
);

CREATE SUBSCRIPTION orders_sub
    CONNECTION 'host=10.0.0.10 port=5432 dbname=mydb user=replicator password=strong_password_here'
    PUBLICATION orders_pub;
```

## Limitations

| Limitation | Detail |
|---|---|
| No DDL replication | `ALTER TABLE`, `CREATE INDEX` must be run manually on subscriber |
| No sequence sync | Sequences on subscriber are independent; IDs may diverge |
| Initial data copy | `pg_dump`/`COPY` happens at subscription creation — can be slow for large tables |
| Subscriber is writable | Local writes to subscribed tables can cause conflicts |
| Truncate support | `TRUNCATE` is replicated only if `publish = 'insert, update, delete, truncate'` (default) |

Monitor subscription status:

```sql
-- On subscriber
SELECT subname, received_lsn, latest_end_lsn, latest_end_time
FROM pg_stat_subscription;
```
