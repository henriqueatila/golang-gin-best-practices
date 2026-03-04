# gin-psql-dba

PostgreSQL DBA and architect skill for Go Gin APIs — the "how to think" complement to gin-database's "how to connect."

## What's Included

- **SKILL.md** — Schema design rules, index selection, migration safety, extension guide, EXPLAIN cheat sheet
- **references/schema-design.md** — Normalization, types, constraints, naming conventions, audit trails
- **references/migration-impact-analysis.md** — Lock analysis, zero-downtime ALTER TABLE patterns
- **references/index-strategy.md** — B-tree, GIN, GiST, BRIN, EXPLAIN ANALYZE deep dive
- **references/query-performance.md** — pg_stat_statements, autovacuum, bloat, pool sizing
- **references/extensions-toolkit.md** — pg_cron, pg_partman, pg_trgm, pgcrypto, pg_audit
- **references/paradedb-full-text-search.md** — BM25 search, hybrid search, analytics
- **references/pgvector-embeddings.md** — Vector storage, HNSW/IVFFlat, similarity queries
- **references/postgis-geospatial.md** — Spatial types, distance queries, GiST indexes
- **references/timescaledb-time-series.md** — Hypertables, compression, retention, aggregates
- **references/row-level-security.md** — RLS policies, multi-tenant isolation, Go middleware
- **references/backup-and-recovery.md** — pg_dump, WAL archiving, PITR, disaster recovery
- **references/replication-and-ha.md** — Streaming replication, read/write split, Patroni, failover

## Categories

`go` `gin` `postgresql` `dba` `schema-design` `migrations` `indexes` `query-optimization` `paradedb` `pgvector` `postgis` `timescaledb` `rls` `backup` `replication`

## Install

```bash
npx skills add henriqueatila/golang-gin-best-practices --skill gin-psql-dba
```
