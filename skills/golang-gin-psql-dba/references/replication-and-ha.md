# Replication and High Availability Reference

This reference covers PostgreSQL replication and high availability strategies for Go/Gin APIs: physical versus logical replication, streaming replication setup, logical publication/subscription, read/write splitting in Go with separate `sqlx` connection pools, replication lag monitoring, Patroni and pg_auto_failover clusters, managed cloud HA services, and application-level resilience patterns including retry backoff and circuit breakers. All Go examples use `sqlx` with raw SQL, `log/slog`, and `context.Context`.

## Table of Contents

1. [Replication Overview](#replication-overview)
2. [Streaming Replication Setup](#streaming-replication-setup)
3. [Logical Replication](#logical-replication)
4. [Read/Write Splitting in Go](#readwrite-splitting-in-go)
5. [Replication Lag Monitoring](#replication-lag-monitoring)
6. [High Availability with Patroni](#high-availability-with-patroni)
7. [pg_auto_failover](#pg_auto_failover)
8. [Managed HA Services](#managed-ha-services)
9. [Go Application Resilience](#go-application-resilience)
10. [Checklist](#checklist)

---

## Replication Overview

### Physical vs Logical Replication

| Feature | Physical (Streaming) | Logical |
|---|---|---|
| Granularity | Entire cluster (all databases) | Per-table, per-publication |
| Cross-version replication | No — identical major version required | Yes — PG 10 to PG 16 supported |
| Standby is read-only | Yes (hot standby) | No — subscriber is writable |
| DDL replication | Yes — everything replicated | No — DDL must be applied manually |
| Sequence replication | Yes | No |
| Overhead | Low — WAL already generated | Moderate — logical decoding CPU |
| Primary use case | HA failover, read scaling | Selective sync, migrations, ETL |

### Streaming Replication (WAL Shipping)

The primary continuously streams Write-Ahead Log (WAL) segments to one or more standbys over a replication connection. Each standby replays WAL records to stay in sync. The standby can serve read-only queries in **hot standby** mode while replay is ongoing.

WAL travels as a stream of binary change records — not SQL. Standbys are byte-for-byte identical to the primary at the replayed LSN (Log Sequence Number).

### Synchronous vs Asynchronous Trade-offs

| Mode | Latency Impact | Data Safety | Use When |
|---|---|---|---|
| **Asynchronous** (default) | None — primary does not wait | Small window of data loss on crash | Read scaling, analytics replicas |
| **Synchronous** (`synchronous_commit = on`) | +1 network RTT per write | Zero data loss — standby confirms before commit | Financial, audit, compliance systems |
| **Remote write** (`synchronous_commit = remote_write`) | +1 RTT, lower than full sync | Standby received WAL but may not have replayed | Balance: near-zero loss, lower latency than full sync |

Set synchronous mode per transaction when needed: `SET LOCAL synchronous_commit = on`. Avoid forcing it globally unless all writes require it.

### Use Cases

- **Read scaling** — route SELECT queries to one or more hot standbys; primary handles writes only.
- **HA failover** — promote a standby if the primary fails; Patroni automates this.
- **Data distribution** — logical replication pushes specific tables to reporting databases, data warehouses, or microservices.
- **Zero-downtime major upgrades** — logical replication from old to new version; switch application endpoint when caught up.

---

## Streaming Replication Setup

### Primary: postgresql.conf

```ini
# postgresql.conf on primary
wal_level            = replica        # minimum for streaming; use 'logical' if logical replication also needed
max_wal_senders      = 5              # max concurrent standby connections (include headroom for pg_basebackup)
wal_keep_size        = 512MB          # WAL retained on primary; protects against slow standbys
hot_standby          = on             # allows standbys to serve read-only queries (default on since PG 14)

# For synchronous replication (optional)
# synchronous_standby_names = 'standby1'
# synchronous_commit = on
```

Reload with `SELECT pg_reload_conf();` — no restart needed for most of these except `wal_level` and `max_wal_senders`.

### Replication User

```sql
-- Run on primary
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'strong_password_here';
```

In `pg_hba.conf` on primary, allow the standby to connect:

```
# pg_hba.conf — allow standby IP to connect for replication
host  replication  replicator  10.0.0.11/32  scram-sha-256
```

### Standby: Initial Base Backup

```bash
# Run on the standby host — copies primary data directory
pg_basebackup \
  --host=10.0.0.10 \
  --username=replicator \
  --pgdata=/var/lib/postgresql/data \
  --wal-method=stream \
  --checkpoint=fast \
  --progress \
  --verbose
```

### Standby: postgresql.conf + standby.signal (PG 12+)

```ini
# postgresql.conf on standby
primary_conninfo = 'host=10.0.0.10 port=5432 user=replicator password=strong_password_here application_name=standby1'
hot_standby      = on
```

```bash
# Create empty file to signal standby mode (replaces recovery.conf from PG 11 and earlier)
touch /var/lib/postgresql/data/standby.signal
```

Start the standby. Monitor replication on the primary:

```sql
SELECT application_name, state, sync_state, replay_lag
FROM pg_stat_replication;
```

### Docker Compose: Primary + Replica

```yaml
# docker-compose.yml
version: "3.9"

services:
  primary:
    image: postgres:16
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: mydb
    volumes:
      - primary_data:/var/lib/postgresql/data
      - ./pg-init/primary.sh:/docker-entrypoint-initdb.d/01-replication.sh
    ports:
      - "5432:5432"

  replica:
    image: postgres:16
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      PGPASSWORD: strong_password_here
    volumes:
      - replica_data:/var/lib/postgresql/data
      - ./pg-init/replica-entrypoint.sh:/entrypoint.sh
    entrypoint: ["/entrypoint.sh"]
    depends_on:
      - primary
    ports:
      - "5433:5432"

volumes:
  primary_data:
  replica_data:
```

```bash
# pg-init/primary.sh — runs inside primary container on first start
psql -U app -c "CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'strong_password_here';"
echo "host replication replicator all scram-sha-256" >> /var/lib/postgresql/data/pg_hba.conf
psql -U app -c "SELECT pg_reload_conf();"
```

```bash
# pg-init/replica-entrypoint.sh
#!/bin/bash
set -e

# Wait for primary to be ready
until pg_isready -h primary -p 5432 -U app; do
  echo "waiting for primary..." && sleep 2
done

# Base backup if data directory is empty
if [ ! -f /var/lib/postgresql/data/PG_VERSION ]; then
  pg_basebackup -h primary -U replicator -D /var/lib/postgresql/data \
    --wal-method=stream --checkpoint=fast
  cat >> /var/lib/postgresql/data/postgresql.conf <<EOF
primary_conninfo = 'host=primary port=5432 user=replicator password=strong_password_here'
hot_standby = on
EOF
  touch /var/lib/postgresql/data/standby.signal
fi

exec docker-entrypoint.sh postgres
```

---

## Logical Replication

### When to Use

- Replicate only selected tables (not the entire cluster)
- Replicate between different PostgreSQL major versions (upgrade path)
- Feed changes to a different database system (Kafka, ClickHouse, read replica with different schema)
- Keep a writable subscriber (analytics database enriched with additional data)

### Publication and Subscription Model

The **publisher** (source) creates a publication that lists which tables to stream. The **subscriber** (destination) creates a subscription pointing at the publisher's connection string.

### CREATE PUBLICATION

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

### CREATE SUBSCRIPTION

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

### Limitations

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

---

## Read/Write Splitting in Go

### Connection Struct

```go
// internal/db/connections.go
package db

import (
    "fmt"
    "log/slog"
    "time"

    "github.com/jmoiron/sqlx"
    _ "github.com/lib/pq"
)

// Connections holds separate pools for the primary (writes) and replica (reads).
type Connections struct {
    primary *sqlx.DB
    replica *sqlx.DB
    logger  *slog.Logger
}

// Primary returns the primary connection pool.
// Use for: INSERT, UPDATE, DELETE, and reads requiring strong consistency.
func (c *Connections) Primary() *sqlx.DB { return c.primary }

// Replica returns the replica connection pool.
// Use for: SELECT queries that tolerate a small replication lag.
func (c *Connections) Replica() *sqlx.DB { return c.replica }

// NewConnections opens and configures separate pools for primary and replica.
func NewConnections(primaryDSN, replicaDSN string, logger *slog.Logger) (*Connections, error) {
    primary, err := sqlx.Connect("postgres", primaryDSN)
    if err != nil {
        return nil, fmt.Errorf("primary connect: %w", err)
    }
    primary.SetMaxOpenConns(20)
    primary.SetMaxIdleConns(5)
    primary.SetConnMaxLifetime(5 * time.Minute)
    primary.SetConnMaxIdleTime(1 * time.Minute)

    replica, err := sqlx.Connect("postgres", replicaDSN)
    if err != nil {
        primary.Close()
        return nil, fmt.Errorf("replica connect: %w", err)
    }
    replica.SetMaxOpenConns(40) // replicas can absorb more concurrent reads
    replica.SetMaxIdleConns(10)
    replica.SetConnMaxLifetime(5 * time.Minute)
    replica.SetConnMaxIdleTime(1 * time.Minute)

    logger.Info("db connections established",
        "primary", primaryDSN,
        "replica", replicaDSN,
    )
    return &Connections{primary: primary, replica: replica, logger: logger}, nil
}

// Close closes both pools.
func (c *Connections) Close() {
    c.primary.Close()
    c.replica.Close()
}
```

### Repository Pattern: Routing by Operation

```go
// internal/repository/order_repo.go
package repository

import (
    "context"
    "fmt"

    "github.com/jmoiron/sqlx"
    appdb "myapp/internal/db"
)

type OrderRepo struct {
    conns *appdb.Connections
}

func NewOrderRepo(conns *appdb.Connections) *OrderRepo {
    return &OrderRepo{conns: conns}
}

// Create writes to primary — always consistent.
func (r *OrderRepo) Create(ctx context.Context, o *Order) error {
    const q = `
        INSERT INTO orders (id, user_id, total, status, created_at)
        VALUES (:id, :user_id, :total, :status, now())
        RETURNING created_at`
    rows, err := r.conns.Primary().NamedQueryContext(ctx, q, o)
    if err != nil {
        return fmt.Errorf("order create: %w", err)
    }
    defer rows.Close()
    if rows.Next() {
        return rows.Scan(&o.CreatedAt)
    }
    return nil
}

// List reads from replica — acceptable lag for list views.
func (r *OrderRepo) List(ctx context.Context, userID string, limit int) ([]Order, error) {
    const q = `
        SELECT id, user_id, total, status, created_at
        FROM orders
        WHERE user_id = $1
        ORDER BY created_at DESC
        LIMIT $2`
    var orders []Order
    if err := r.conns.Replica().SelectContext(ctx, &orders, q, userID, limit); err != nil {
        return nil, fmt.Errorf("order list: %w", err)
    }
    return orders, nil
}

// GetByID supports read-your-writes: pass strong=true after a write
// to ensure the caller sees what they just created.
func (r *OrderRepo) GetByID(ctx context.Context, id string, strong bool) (*Order, error) {
    db := r.conns.Replica()
    if strong {
        db = r.conns.Primary()
    }
    var o Order
    if err := db.GetContext(ctx, &o, `SELECT * FROM orders WHERE id = $1`, id); err != nil {
        return nil, fmt.Errorf("order get %s: %w", id, err)
    }
    return &o, nil
}
```

### Read-Your-Writes via Context Flag

Pass a context key so any layer can signal "use primary for this request":

```go
// internal/db/ctx.go
package db

import "context"

type ctxKey struct{}

// WithStrongRead marks the context to force primary reads.
func WithStrongRead(ctx context.Context) context.Context {
    return context.WithValue(ctx, ctxKey{}, true)
}

// IsStrongRead returns true if the context requires primary read.
func IsStrongRead(ctx context.Context) bool {
    v, _ := ctx.Value(ctxKey{}).(bool)
    return v
}
```

In the repository, replace the `strong bool` parameter:

```go
func (r *OrderRepo) pickDB(ctx context.Context) *sqlx.DB {
    if appdb.IsStrongRead(ctx) {
        return r.conns.Primary()
    }
    return r.conns.Replica()
}
```

In the handler, after a POST, tag the redirect context:

```go
ctx := appdb.WithStrongRead(c.Request.Context())
order, err := repo.GetByID(ctx, newOrderID, false)
```

---

## Replication Lag Monitoring

### Query on Primary: pg_stat_replication

```sql
-- Shows lag per connected standby
SELECT
    application_name,
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    write_lag,
    flush_lag,
    replay_lag,
    pg_size_pretty(pg_wal_lsn_diff(sent_lsn, replay_lsn)) AS lag_bytes
FROM pg_stat_replication
ORDER BY replay_lag DESC NULLS LAST;
```

`replay_lag` is wall-clock time since the primary last confirmed the standby replayed that WAL segment. Alert when it exceeds 30 seconds.

### Query on Replica

```sql
-- Transaction replay lag (most human-readable)
SELECT now() - pg_last_xact_replay_timestamp() AS replication_lag;

-- LSN-based lag in bytes (run on replica, compare to primary sent_lsn)
SELECT
    pg_last_wal_receive_lsn()  AS received_lsn,
    pg_last_wal_replay_lsn()   AS replayed_lsn,
    pg_wal_lsn_diff(
        pg_last_wal_receive_lsn(),
        pg_last_wal_replay_lsn()
    )                           AS receive_vs_replay_bytes,
    pg_is_in_recovery()        AS is_replica;
```

### Go Lag Monitor

```go
// internal/db/replication_monitor.go
package db

import (
    "context"
    "fmt"
    "log/slog"
    "time"

    "github.com/jmoiron/sqlx"
)

const lagWarningThreshold = 30 * time.Second

type ReplicationStatus struct {
    ApplicationName string        `db:"application_name"`
    State           string        `db:"state"`
    ReplayLag       time.Duration // parsed from interval
    LagBytes        int64         `db:"lag_bytes"`
    BehindThreshold bool
}

// CheckReplicationLag queries pg_stat_replication on the primary and logs
// a warning for any standby that exceeds the threshold.
func CheckReplicationLag(ctx context.Context, primary *sqlx.DB, logger *slog.Logger) error {
    const q = `
        SELECT
            application_name,
            state,
            EXTRACT(EPOCH FROM replay_lag)::bigint            AS replay_lag_seconds,
            pg_wal_lsn_diff(sent_lsn, replay_lsn)            AS lag_bytes
        FROM pg_stat_replication`

    type row struct {
        ApplicationName  string `db:"application_name"`
        State            string `db:"state"`
        ReplayLagSeconds int64  `db:"replay_lag_seconds"`
        LagBytes         int64  `db:"lag_bytes"`
    }

    var rows []row
    if err := primary.SelectContext(ctx, &rows, q); err != nil {
        return fmt.Errorf("pg_stat_replication query: %w", err)
    }

    if len(rows) == 0 {
        logger.WarnContext(ctx, "no standbys connected to primary")
        return nil
    }

    for _, r := range rows {
        lag := time.Duration(r.ReplayLagSeconds) * time.Second
        if lag > lagWarningThreshold {
            logger.WarnContext(ctx, "replication lag exceeds threshold",
                "standby", r.ApplicationName,
                "state", r.State,
                "lag", lag.String(),
                "lag_bytes", r.LagBytes,
            )
        } else {
            logger.InfoContext(ctx, "replication healthy",
                "standby", r.ApplicationName,
                "lag", lag.String(),
            )
        }
    }
    return nil
}
```

When `ReplayLagSeconds` exceeds the threshold, route reads to primary until lag recovers:

```go
// ShouldUsePrimary returns true when replica lag is too high.
func ShouldUsePrimary(ctx context.Context, replica *sqlx.DB) bool {
    var lagSeconds float64
    err := replica.QueryRowContext(ctx,
        `SELECT EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp()))`,
    ).Scan(&lagSeconds)
    if err != nil || lagSeconds > 30 {
        return true
    }
    return false
}
```

---

## High Availability with Patroni

### What Patroni Does

Patroni is a Python daemon that wraps each PostgreSQL node with:
- **Leader election** via a Distributed Configuration Store (DCS): etcd, Consul, or ZooKeeper
- **Automatic failover** — promotes the most up-to-date standby when the primary is unreachable
- **Configuration management** — applies postgresql.conf changes cluster-wide via the DCS
- **REST API** — `/primary`, `/replica`, `/health` endpoints consumed by load balancers

Each node runs Patroni alongside PostgreSQL. Patroni owns the `postgresql.conf` and controls when PostgreSQL starts/stops.

### Architecture

```
Application
    |
    v
HAProxy (reads /primary endpoint)
    |        \
    v          v
Patroni+PG   Patroni+PG   Patroni+PG
(primary)    (standby)    (standby)
    |              |            |
    +------etcd cluster---------+
```

etcd (or Consul) holds the leader key. If the primary's Patroni fails to renew its lease, a standby acquires the key and promotes itself.

### Docker Compose: 3-Node Patroni Cluster with etcd

```yaml
# docker-compose.yml
version: "3.9"

networks:
  patroni_net:
    driver: bridge

services:
  etcd:
    image: quay.io/coreos/etcd:v3.5.9
    environment:
      ETCD_NAME: etcd0
      ETCD_LISTEN_CLIENT_URLS: http://0.0.0.0:2379
      ETCD_ADVERTISE_CLIENT_URLS: http://etcd:2379
      ETCD_LISTEN_PEER_URLS: http://0.0.0.0:2380
      ETCD_INITIAL_ADVERTISE_PEER_URLS: http://etcd:2380
      ETCD_INITIAL_CLUSTER: etcd0=http://etcd:2380
      ETCD_INITIAL_CLUSTER_TOKEN: patroni-cluster
      ETCD_INITIAL_CLUSTER_STATE: new
      ETCD_DATA_DIR: /etcd-data
    volumes:
      - etcd_data:/etcd-data
    networks: [patroni_net]

  patroni1:
    image: patroni:latest               # build from github.com/zalando/patroni
    environment:
      PATRONI_NAME: patroni1
      PATRONI_POSTGRESQL_CONNECT_ADDRESS: patroni1:5432
      PATRONI_RESTAPI_CONNECT_ADDRESS: patroni1:8008
      PATRONI_ETCD_URL: http://etcd:2379
      PATRONI_POSTGRESQL_DATA_DIR: /data/patroni
      PATRONI_SUPERUSER_USERNAME: postgres
      PATRONI_SUPERUSER_PASSWORD: secret
      PATRONI_REPLICATION_USERNAME: replicator
      PATRONI_REPLICATION_PASSWORD: rep_secret
    volumes:
      - patroni1_data:/data/patroni
    networks: [patroni_net]

  patroni2:
    image: patroni:latest
    environment:
      PATRONI_NAME: patroni2
      PATRONI_POSTGRESQL_CONNECT_ADDRESS: patroni2:5432
      PATRONI_RESTAPI_CONNECT_ADDRESS: patroni2:8008
      PATRONI_ETCD_URL: http://etcd:2379
      PATRONI_POSTGRESQL_DATA_DIR: /data/patroni
      PATRONI_SUPERUSER_USERNAME: postgres
      PATRONI_SUPERUSER_PASSWORD: secret
      PATRONI_REPLICATION_USERNAME: replicator
      PATRONI_REPLICATION_PASSWORD: rep_secret
    volumes:
      - patroni2_data:/data/patroni
    networks: [patroni_net]

  patroni3:
    image: patroni:latest
    environment:
      PATRONI_NAME: patroni3
      PATRONI_POSTGRESQL_CONNECT_ADDRESS: patroni3:5432
      PATRONI_RESTAPI_CONNECT_ADDRESS: patroni3:8008
      PATRONI_ETCD_URL: http://etcd:2379
      PATRONI_POSTGRESQL_DATA_DIR: /data/patroni
      PATRONI_SUPERUSER_USERNAME: postgres
      PATRONI_SUPERUSER_PASSWORD: secret
      PATRONI_REPLICATION_USERNAME: replicator
      PATRONI_REPLICATION_PASSWORD: rep_secret
    volumes:
      - patroni3_data:/data/patroni
    networks: [patroni_net]

  haproxy:
    image: haproxy:2.9
    ports:
      - "5432:5432"   # primary (writes)
      - "5433:5433"   # replicas (reads)
      - "7000:7000"   # HAProxy stats
    volumes:
      - ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro
    networks: [patroni_net]

volumes:
  etcd_data:
  patroni1_data:
  patroni2_data:
  patroni3_data:
```

```ini
# haproxy.cfg
frontend pg_primary
    bind *:5432
    default_backend pg_primary_backend

frontend pg_replica
    bind *:5433
    default_backend pg_replica_backend

backend pg_primary_backend
    option httpchk GET /primary
    http-check expect status 200
    server patroni1 patroni1:5432 check port 8008 inter 2s fall 3 rise 2
    server patroni2 patroni2:5432 check port 8008 inter 2s fall 3 rise 2
    server patroni3 patroni3:5432 check port 8008 inter 2s fall 3 rise 2

backend pg_replica_backend
    option httpchk GET /replica
    http-check expect status 200
    balance roundrobin
    server patroni1 patroni1:5432 check port 8008 inter 2s fall 3 rise 2
    server patroni2 patroni2:5432 check port 8008 inter 2s fall 3 rise 2
    server patroni3 patroni3:5432 check port 8008 inter 2s fall 3 rise 2
```

### Application Connection String

With HAProxy in front, the application uses two static DSNs regardless of which physical node is primary:

```
PRIMARY_DSN=postgres://app:secret@haproxy:5432/mydb
REPLICA_DSN=postgres://app:secret@haproxy:5433/mydb
```

Failover is transparent — HAProxy reroutes connections to the new primary within seconds of Patroni completing promotion.

---

## pg_auto_failover

### What It Is

`pg_auto_failover` is a PostgreSQL extension and service by Citus Data (Microsoft). It provides automatic failover for a **2-node** setup (primary + secondary) managed by a dedicated **monitor node** that tracks health and coordinates promotion.

### Monitor Node Concept

A third lightweight PostgreSQL instance running the `pgautofailover` extension acts as the monitor. It does not store application data — it only tracks cluster state via health checks and TCP connections to both nodes.

```
Monitor (pgautofailover extension)
    |               |
    v               v
Primary            Secondary
(read/write)       (read-only hot standby)
```

### Setup Overview

```bash
# On monitor host
pg_autoctl create monitor --pgdata /data/monitor --auth trust --ssl-self-signed

# On primary host
pg_autoctl create postgres \
  --pgdata /data/primary \
  --monitor postgres://autoctl_node@monitor:5432/pg_auto_failover \
  --name primary \
  --auth trust

# On secondary host
pg_autoctl create postgres \
  --pgdata /data/secondary \
  --monitor postgres://autoctl_node@monitor:5432/pg_auto_failover \
  --name secondary \
  --auth trust

# Start both nodes
pg_autoctl run --pgdata /data/primary &
pg_autoctl run --pgdata /data/secondary &
```

Check status:

```bash
pg_autoctl show state --pgdata /data/monitor
```

### When to Use pg_auto_failover vs Patroni

| Concern | pg_auto_failover | Patroni |
|---|---|---|
| Setup complexity | Low — 3 commands | High — YAML config + DCS |
| External dependencies | None beyond PostgreSQL | etcd / Consul / ZooKeeper |
| Node count | 2 nodes + 1 monitor | 3+ nodes recommended |
| Failover time | ~30 seconds default | ~10–30 seconds tunable |
| Multi-datacenter | Limited | Full support |
| Community/ecosystem | Smaller, Microsoft-backed | Larger, Zalando-originated |
| Best for | Simple 2-node setups | Production clusters with DCS already available |

---

## Managed HA Services

### Comparison Table

| Service | Failover Time | Read Replicas | Managed Backups | Cost Tier |
|---|---|---|---|---|
| **AWS RDS Multi-AZ** | ~60–120 s (DNS failover) | Yes (separate read replica instances) | Yes — automated, point-in-time | Medium |
| **AWS Aurora PostgreSQL** | ~30 s | Up to 15 replicas, < 10 ms lag | Yes — continuous to S3 | High |
| **Google Cloud SQL HA** | ~60 s (regional failover) | Yes — read replicas in same/different region | Yes — automated | Medium |
| **Google AlloyDB** | ~60 s | Yes — up to 20 read pool nodes | Yes | High |
| **Supabase** | ~60 s (via Fly.io primary) | Yes — read replicas (paid plans) | Yes | Low–Medium |
| **Neon** | Near-instant (branching architecture) | Serverless replicas | Yes | Low–Medium |

### AWS RDS Multi-AZ

RDS Multi-AZ creates a synchronous standby in a second Availability Zone. Failover is automatic: RDS updates the CNAME DNS record to point to the standby. The application must handle the ~60 second DNS propagation gap and connection reset.

```go
// Use short ConnMaxLifetime so connections re-resolve the DNS after failover
primary.SetConnMaxLifetime(1 * time.Minute)
```

Read replicas on RDS are separate instances at an additional cost. They use asynchronous replication and have independent endpoints — route reads to them explicitly.

### Google Cloud SQL HA

Cloud SQL HA uses a regional instance with a standby in a second zone. Failover is automatic and managed entirely by Google. Connection via the Cloud SQL Auth Proxy handles reconnection transparently.

```bash
# Cloud SQL Auth Proxy — handles failover reconnection automatically
cloud-sql-proxy myproject:us-central1:myinstance
```

### Supabase Read Replicas

Supabase provides read replicas on paid plans. The replica endpoint is a separate host:

```
PRIMARY: db.project-ref.supabase.co:5432
REPLICA: db.project-ref.read.supabase.co:5432
```

Use the same `Connections` pattern from Section 4.

---

## Go Application Resilience

### Connection Retry with Exponential Backoff

```go
// internal/db/wait.go
package db

import (
    "context"
    "fmt"
    "log/slog"
    "time"

    "github.com/jmoiron/sqlx"
    _ "github.com/lib/pq"
)

// WaitForConnection attempts to connect to dsn with exponential backoff.
// It returns the connected *sqlx.DB or an error after maxAttempts.
func WaitForConnection(ctx context.Context, dsn string, maxAttempts int, logger *slog.Logger) (*sqlx.DB, error) {
    var db *sqlx.DB
    var err error
    delay := 500 * time.Millisecond

    for attempt := 1; attempt <= maxAttempts; attempt++ {
        db, err = sqlx.ConnectContext(ctx, "postgres", dsn)
        if err == nil {
            logger.InfoContext(ctx, "database connected", "attempt", attempt)
            return db, nil
        }

        logger.WarnContext(ctx, "database connection failed",
            "attempt", attempt,
            "max", maxAttempts,
            "delay", delay.String(),
            "err", err,
        )

        select {
        case <-ctx.Done():
            return nil, fmt.Errorf("context cancelled waiting for db: %w", ctx.Err())
        case <-time.After(delay):
        }

        // Exponential backoff capped at 30 seconds
        delay *= 2
        if delay > 30*time.Second {
            delay = 30 * time.Second
        }
    }

    return nil, fmt.Errorf("database unavailable after %d attempts: %w", maxAttempts, err)
}
```

### Health Check Endpoint with Replication Status

```go
// internal/handler/ha_health_handler.go
package handler

import (
    "net/http"
    "time"

    "github.com/gin-gonic/gin"
    "github.com/jmoiron/sqlx"
    "log/slog"
)

type HAHealthHandler struct {
    primary *sqlx.DB
    replica *sqlx.DB
    logger  *slog.Logger
}

func NewHAHealthHandler(primary, replica *sqlx.DB, logger *slog.Logger) *HAHealthHandler {
    return &HAHealthHandler{primary: primary, replica: replica, logger: logger}
}

type haHealthResponse struct {
    Primary    dbNodeStatus `json:"primary"`
    Replica    dbNodeStatus `json:"replica"`
    ReplicaLag string       `json:"replica_lag"`
}

type dbNodeStatus struct {
    OK      bool   `json:"ok"`
    Latency string `json:"latency_ms"`
}

// GET /internal/health/ha
func (h *HAHealthHandler) Check(c *gin.Context) {
    ctx := c.Request.Context()
    resp := haHealthResponse{}

    // Ping primary
    start := time.Now()
    if err := h.primary.PingContext(ctx); err != nil {
        h.logger.WarnContext(ctx, "primary ping failed", "err", err)
        resp.Primary = dbNodeStatus{OK: false}
    } else {
        resp.Primary = dbNodeStatus{OK: true, Latency: time.Since(start).String()}
    }

    // Ping replica
    start = time.Now()
    if err := h.replica.PingContext(ctx); err != nil {
        h.logger.WarnContext(ctx, "replica ping failed", "err", err)
        resp.Replica = dbNodeStatus{OK: false}
    } else {
        resp.Replica = dbNodeStatus{OK: true, Latency: time.Since(start).String()}
    }

    // Replication lag from replica
    if resp.Replica.OK {
        var lagSeconds float64
        err := h.replica.QueryRowContext(ctx,
            `SELECT COALESCE(EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp())), 0)`,
        ).Scan(&lagSeconds)
        if err == nil {
            resp.ReplicaLag = time.Duration(lagSeconds * float64(time.Second)).String()
        }
    }

    status := http.StatusOK
    if !resp.Primary.OK {
        status = http.StatusServiceUnavailable
    }
    c.JSON(status, resp)
}
```

### Circuit Breaker for Replica Unavailability

When the replica is down, fall back to primary automatically:

```go
// internal/db/circuit.go
package db

import (
    "context"
    "log/slog"
    "sync/atomic"
    "time"

    "github.com/jmoiron/sqlx"
)

// ReplicaCircuit tracks replica availability with a simple open/closed state.
// When open (tripped), all reads fall back to primary.
type ReplicaCircuit struct {
    tripped    atomic.Bool
    primary    *sqlx.DB
    replica    *sqlx.DB
    resetAfter time.Duration
    logger     *slog.Logger
}

func NewReplicaCircuit(primary, replica *sqlx.DB, resetAfter time.Duration, logger *slog.Logger) *ReplicaCircuit {
    return &ReplicaCircuit{
        primary:    primary,
        replica:    replica,
        resetAfter: resetAfter,
        logger:     logger,
    }
}

// DB returns the replica if healthy, or the primary if the circuit is tripped.
func (rc *ReplicaCircuit) DB(ctx context.Context) *sqlx.DB {
    if rc.tripped.Load() {
        return rc.primary
    }
    return rc.replica
}

// RecordFailure trips the circuit and schedules an automatic reset.
func (rc *ReplicaCircuit) RecordFailure(ctx context.Context) {
    if rc.tripped.CompareAndSwap(false, true) {
        rc.logger.WarnContext(ctx, "replica circuit tripped — routing reads to primary",
            "reset_after", rc.resetAfter.String(),
        )
        go func() {
            time.Sleep(rc.resetAfter)
            if err := rc.replica.PingContext(context.Background()); err == nil {
                rc.tripped.Store(false)
                rc.logger.Info("replica circuit reset — replica is healthy again")
            }
        }()
    }
}
```

Usage in a repository:

```go
func (r *OrderRepo) ListWithCircuit(ctx context.Context, userID string) ([]Order, error) {
    db := r.circuit.DB(ctx)
    var orders []Order
    if err := db.SelectContext(ctx, &orders,
        `SELECT * FROM orders WHERE user_id = $1 ORDER BY created_at DESC LIMIT 100`, userID,
    ); err != nil {
        r.circuit.RecordFailure(ctx)
        // Retry on primary
        if err2 := r.conns.Primary().SelectContext(ctx, &orders,
            `SELECT * FROM orders WHERE user_id = $1 ORDER BY created_at DESC LIMIT 100`, userID,
        ); err2 != nil {
            return nil, fmt.Errorf("order list fallback: %w", err2)
        }
    }
    return orders, nil
}
```

### Graceful Handling of Failover

During a Patroni or RDS failover, existing connections receive `driver: bad connection`. The standard `database/sql` pool retries transparently on the next request, but long-lived idle connections may need eviction:

```go
// After detecting a failover event (e.g., from a health probe or error count spike),
// close all idle connections to force re-dial against the new primary.
func ForceReconnect(db *sqlx.DB) {
    db.SetMaxIdleConns(0)               // evict all idle connections immediately
    time.Sleep(100 * time.Millisecond)
    db.SetMaxIdleConns(5)               // restore normal idle pool
}
```

Set `ConnMaxLifetime` to a value shorter than the failover window (1–2 minutes) so connections naturally recycle before the next failover attempt.

---

## Checklist

- [ ] Streaming replication configured: `wal_level = replica`, `max_wal_senders` set, `replicator` role created
- [ ] Standby verified with `pg_stat_replication` showing active standby in `streaming` state
- [ ] Replication lag monitored: alert threshold set at 30 seconds (`replay_lag`)
- [ ] Go application uses separate `primary` and `replica` connection pools
- [ ] Read-your-writes pattern implemented for post-write reads (context flag or explicit `strong` parameter)
- [ ] Failover tested: promote standby manually, verify application reconnects within `ConnMaxLifetime`
- [ ] Application handles `bad connection` errors gracefully — no panics on failover
- [ ] Circuit breaker or fallback logic routes reads to primary when replica is unreachable
- [ ] Backups configured independently of replication (`pg_basebackup`, `pgbackrest`, or managed service snapshots)
- [ ] WAL archiving enabled if point-in-time recovery is required (`archive_mode = on`, `archive_command`)

**Cross-references:**

- Connection pool sizing and PgBouncer: [query-performance.md](query-performance.md)
- Schema design and DDL conventions: [schema-design.md](schema-design.md)
- Extension ecosystem (pg_cron, pgstattuple): [extensions-toolkit.md](extensions-toolkit.md)
- Zero-downtime migrations alongside HA: [migration-impact-analysis.md](migration-impact-analysis.md)
