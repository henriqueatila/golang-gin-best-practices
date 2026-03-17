# Replication and HA — Patroni and pg_auto_failover

## High Availability with Patroni

Patroni is a Python daemon wrapping each PostgreSQL node with:
- **Leader election** via DCS: etcd, Consul, or ZooKeeper
- **Automatic failover** — promotes the most up-to-date standby
- **Configuration management** — applies postgresql.conf changes cluster-wide
- **REST API** — `/primary`, `/replica`, `/health` endpoints for load balancers

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

### Docker Compose: 3-Node Patroni Cluster

Services: `etcd` (v3.5, DCS), `patroni1/2/3` (image `patroni:latest` from zalando/patroni — env: `PATRONI_NAME`, `PATRONI_POSTGRESQL_CONNECT_ADDRESS`, `PATRONI_RESTAPI_CONNECT_ADDRESS`, `PATRONI_ETCD_URL`), `haproxy:2.9` (ports 5432=primary, 5433=replicas, 7000=stats). All on `patroni_net` bridge network.

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

Application DSNs (static, HAProxy handles routing):

```
PRIMARY_DSN=postgres://app:secret@haproxy:5432/mydb
REPLICA_DSN=postgres://app:secret@haproxy:5433/mydb
```

---

## pg_auto_failover

`pg_auto_failover` provides automatic failover for a **2-node** setup via a dedicated monitor node.

```
Monitor (pgautofailover extension)
    |               |
    v               v
Primary            Secondary
(read/write)       (read-only hot standby)
```

### Setup

```bash
# On monitor host
pg_autoctl create monitor --pgdata /data/monitor --auth trust --ssl-self-signed

# On primary host
pg_autoctl create postgres \
  --pgdata /data/primary \
  --monitor postgres://autoctl_node@monitor:5432/pg_auto_failover \
  --name primary --auth trust

# On secondary host
pg_autoctl create postgres \
  --pgdata /data/secondary \
  --monitor postgres://autoctl_node@monitor:5432/pg_auto_failover \
  --name secondary --auth trust

pg_autoctl run --pgdata /data/primary &
pg_autoctl run --pgdata /data/secondary &

# Check status
pg_autoctl show state --pgdata /data/monitor
```

### pg_auto_failover vs Patroni

| Concern | pg_auto_failover | Patroni |
|---|---|---|
| Setup complexity | Low — 3 commands | High — YAML config + DCS |
| External dependencies | None beyond PostgreSQL | etcd / Consul / ZooKeeper |
| Node count | 2 nodes + 1 monitor | 3+ nodes recommended |
| Failover time | ~30 seconds default | ~10–30 seconds tunable |
| Multi-datacenter | Limited | Full support |
| Best for | Simple 2-node setups | Production clusters with DCS already available |
