# Replication and HA — Managed HA Services

## Comparison Table

| Service | Failover Time | Read Replicas | Managed Backups | Cost Tier |
|---|---|---|---|---|
| **AWS RDS Multi-AZ** | ~60–120 s (DNS failover) | Yes (separate read replica instances) | Yes — automated, point-in-time | Medium |
| **AWS Aurora PostgreSQL** | ~30 s | Up to 15 replicas, < 10 ms lag | Yes — continuous to S3 | High |
| **Google Cloud SQL HA** | ~60 s (regional failover) | Yes — read replicas in same/different region | Yes — automated | Medium |
| **Google AlloyDB** | ~60 s | Yes — up to 20 read pool nodes | Yes | High |
| **Supabase** | ~60 s (via Fly.io primary) | Yes — read replicas (paid plans) | Yes | Low–Medium |
| **Neon** | Near-instant (branching architecture) | Serverless replicas | Yes | Low–Medium |

## AWS RDS Multi-AZ

RDS Multi-AZ creates a synchronous standby in a second Availability Zone. Failover is automatic: RDS updates the CNAME DNS record to point to the standby. The application must handle the ~60 second DNS propagation gap and connection reset.

```go
// Use short ConnMaxLifetime so connections re-resolve the DNS after failover
primary.SetConnMaxLifetime(1 * time.Minute)
```

Read replicas on RDS are separate instances at an additional cost. They use asynchronous replication and have independent endpoints — route reads to them explicitly.

## Google Cloud SQL HA

Cloud SQL HA uses a regional instance with a standby in a second zone. Failover is automatic and managed entirely by Google. Connection via the Cloud SQL Auth Proxy handles reconnection transparently.

```bash
# Cloud SQL Auth Proxy — handles failover reconnection automatically
cloud-sql-proxy myproject:us-central1:myinstance
```

## Supabase Read Replicas

Supabase provides read replicas on paid plans. The replica endpoint is a separate host:

```
PRIMARY: db.project-ref.supabase.co:5432
REPLICA: db.project-ref.read.supabase.co:5432
```

Use the same `Connections` pattern from [replication-and-ha-go-readwrite.md](replication-and-ha-go-readwrite.md).
