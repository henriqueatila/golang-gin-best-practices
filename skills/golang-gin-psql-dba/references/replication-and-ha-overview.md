# Replication and HA — Overview

PostgreSQL replication and high availability strategies for Go/Gin APIs.

## Physical vs Logical Replication

| Feature | Physical (Streaming) | Logical |
|---|---|---|
| Granularity | Entire cluster (all databases) | Per-table, per-publication |
| Cross-version replication | No — identical major version required | Yes — PG 10 to PG 16 supported |
| Standby is read-only | Yes (hot standby) | No — subscriber is writable |
| DDL replication | Yes — everything replicated | No — DDL must be applied manually |
| Sequence replication | Yes | No |
| Overhead | Low — WAL already generated | Moderate — logical decoding CPU |
| Primary use case | HA failover, read scaling | Selective sync, migrations, ETL |

## Streaming Replication (WAL Shipping)

The primary continuously streams Write-Ahead Log (WAL) segments to one or more standbys over a replication connection. Each standby replays WAL records to stay in sync. The standby can serve read-only queries in **hot standby** mode while replay is ongoing.

WAL travels as a stream of binary change records — not SQL. Standbys are byte-for-byte identical to the primary at the replayed LSN (Log Sequence Number).

## Synchronous vs Asynchronous Trade-offs

| Mode | Latency Impact | Data Safety | Use When |
|---|---|---|---|
| **Asynchronous** (default) | None — primary does not wait | Small window of data loss on crash | Read scaling, analytics replicas |
| **Synchronous** (`synchronous_commit = on`) | +1 network RTT per write | Zero data loss — standby confirms before commit | Financial, audit, compliance systems |
| **Remote write** (`synchronous_commit = remote_write`) | +1 RTT, lower than full sync | Standby received WAL but may not have replayed | Balance: near-zero loss, lower latency than full sync |

Set synchronous mode per transaction when needed: `SET LOCAL synchronous_commit = on`. Avoid forcing it globally unless all writes require it.

## Use Cases

- **Read scaling** — route SELECT queries to one or more hot standbys; primary handles writes only.
- **HA failover** — promote a standby if the primary fails; Patroni automates this.
- **Data distribution** — logical replication pushes specific tables to reporting databases, data warehouses, or microservices.
- **Zero-downtime major upgrades** — logical replication from old to new version; switch application endpoint when caught up.

**Related files:**
- Setup details: [replication-and-ha-streaming-setup.md](replication-and-ha-streaming-setup.md)
- Logical replication: [replication-and-ha-logical.md](replication-and-ha-logical.md)
- Read/write splitting in Go: [replication-and-ha-go-readwrite.md](replication-and-ha-go-readwrite.md)
- Lag monitoring: [replication-and-ha-lag-monitoring.md](replication-and-ha-lag-monitoring.md)
- HA clusters (Patroni, pg_auto_failover): [replication-and-ha-clusters.md](replication-and-ha-clusters.md)
- Managed services: [replication-and-ha-managed-services.md](replication-and-ha-managed-services.md)
- Go resilience patterns: [replication-and-ha-go-resilience.md](replication-and-ha-go-resilience.md)
