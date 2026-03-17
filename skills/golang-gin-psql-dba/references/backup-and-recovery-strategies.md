# Backup and Recovery — Strategies, pg_dump, and pg_basebackup

## Strategy Comparison

| Strategy | Type | RPO | RTO | Best For |
|---|---|---|---|---|
| **pg_dump** | Logical | Hours (frequency-dependent) | Minutes–Hours | Dev/staging, schema migration safety, selective restores |
| **pg_basebackup** | Physical | Hours (frequency-dependent) | Minutes–Hours | Full cluster restore, replica seed, disaster recovery baseline |
| **WAL archiving** | Physical + Continuous | Near-zero (seconds) | Minutes–Hours | Production PITR, "undo accidental DELETE" |
| **Managed snapshots** | Physical | Varies (hourly–daily) | Minutes | Managed PaaS (RDS, Cloud SQL, Supabase) — zero operational overhead |

**RPO** = Recovery Point Objective: how much data you can afford to lose.
**RTO** = Recovery Time Objective: how long the restore can take.

### Decision Tree

```
Is this a managed PaaS (RDS / Cloud SQL / Supabase)?
  YES → Use managed snapshots + enable PITR on Pro/Premium plan.
  NO (self-hosted) → continue:

    Is production data? (data loss measured in seconds is unacceptable)
      YES → pg_basebackup baseline + WAL archiving → PITR.
      NO (staging/dev) → pg_dump on a schedule is sufficient.

    Do you need selective table restores?
      YES → pg_dump (custom format) — physical backups restore the full cluster only.

    Do you need to seed a read replica or spin up a standby?
      YES → pg_basebackup (streaming replication seed).
```

**WAL archiving** alone is not enough — combine with a base backup. WAL replays forward from the last base backup. **pg_dump** cannot replay transactions after the dump completed — RPO equals the dump interval. **pg_basebackup** restore is faster than pg_dump for full cluster restores.

---

## pg_dump / pg_restore

### Backup Formats

| Format | Flag | Notes |
|---|---|---|
| Plain SQL | `-Fp` (default) | Human-readable, no parallel restore, large files |
| Custom | `-Fc` | Compressed, supports parallel restore with `-j`, best for production |
| Directory | `-Fd` | One file per table, supports parallel dump and restore |
| Tar | `-Ft` | Single tarball, no parallel restore |

**Prefer custom format (`-Fc`) for anything larger than a few MB.**

### Common Commands

```bash
# Full database dump — custom format, compressed
pg_dump -Fc -d "postgres://user:pass@host:5432/mydb" -f mydb_$(date +%Y%m%d_%H%M%S).dump

# Schema only (no data) — useful before migrations
pg_dump -Fc --schema-only -d "postgres://user:pass@host:5432/mydb" -f schema.dump

# Single table
pg_dump -Fc -t public.orders -d "postgres://user:pass@host:5432/mydb" -f orders.dump

# Parallel dump — 4 workers, directory format required
pg_dump -Fd -j 4 -d "postgres://user:pass@host:5432/mydb" -f mydb_dir/
```

### Restore Commands

```bash
# Full restore — drop & recreate target database first
dropdb mydb && createdb mydb
pg_restore -d "postgres://user:pass@host:5432/mydb" mydb.dump

# Parallel restore — 8 workers (custom or directory format)
pg_restore -j 8 -d "postgres://user:pass@host:5432/mydb" mydb.dump

# Selective table restore from a full dump
pg_restore -t orders -d "postgres://user:pass@host:5432/mydb" mydb.dump
```

### Automated Daily Dump Script

```bash
#!/usr/bin/env bash
set -euo pipefail

BACKUP_DIR="${BACKUP_DIR:-/var/backups/postgres}"
DB_URL="${DATABASE_URL:?DATABASE_URL must be set}"
KEEP_DAYS="${KEEP_DAYS:-7}"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="${BACKUP_DIR}/mydb_${TIMESTAMP}.dump"

mkdir -p "$BACKUP_DIR"
echo "[$(date -u +%Y-%m-%dT%H:%M:%SZ)] Starting backup → $BACKUP_FILE"
pg_dump -Fc -j 4 -d "$DB_URL" -f "$BACKUP_FILE"
FILESIZE=$(du -sh "$BACKUP_FILE" | cut -f1)
echo "[$(date -u +%Y-%m-%dT%H:%M:%SZ)] Backup complete — size: $FILESIZE"
find "$BACKUP_DIR" -name "*.dump" -mtime "+${KEEP_DAYS}" -delete
```

Add to root crontab: `0 2 * * * DATABASE_URL="postgres://..." /usr/local/bin/pg-backup.sh >> /var/log/pg-backup.log 2>&1`

---

## pg_basebackup

Physical backup of the entire PostgreSQL cluster. Required as the base for WAL archiving / PITR.

### Key Flags

| Flag | Purpose |
|---|---|
| `-D /path/to/backup` | Output directory |
| `-Ft -z` | Tar format + gzip compress |
| `-Xs` | Include WAL via streaming |
| `-P` | Show progress |
| `-c fast` | Force immediate checkpoint |
| `-R` | Write `standby.signal` for replica setup |

### Example Commands

```bash
# Backup to local directory — streaming WAL, progress, fast checkpoint
pg_basebackup \
  -h localhost -p 5432 -U replicator \
  -D /var/backups/postgres/basebackup_$(date +%Y%m%d) \
  -Xs -P -c fast

# Seed a read replica (writes recovery config automatically)
pg_basebackup \
  -h primary-host -p 5432 -U replicator \
  -D /var/lib/postgresql/16/main \
  -Xs -P -R -c fast
```

```sql
-- Replication user requirement
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'strongpassword';
```
