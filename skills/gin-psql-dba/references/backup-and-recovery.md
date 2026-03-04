# Backup and Recovery Reference

PostgreSQL backup and recovery reference for Go/Gin APIs. Covers the four backup strategies (pg_dump, pg_basebackup, WAL archiving, managed snapshots), their RPO/RTO trade-offs, and how to choose between them. Includes shell scripts for automated daily dumps, a Go helper that shells out to pg_dump with progress logging, step-by-step Point-in-Time Recovery (PITR) procedures, automated restore validation with row-count checks, managed-service comparison across AWS RDS / Cloud SQL / Supabase, Docker development backup patterns, schedule recommendations by environment, and a disaster recovery checklist. All Go examples use `sqlx`, `log/slog`, and `context.Context`.

## Table of Contents

1. [Backup Strategy Overview](#backup-strategy-overview)
2. [pg_dump / pg_restore](#pg_dump--pg_restore)
3. [pg_basebackup](#pg_basebackup)
4. [WAL Archiving and PITR](#wal-archiving-and-pitr)
5. [Backup Validation](#backup-validation)
6. [Managed Database Backups](#managed-database-backups)
7. [Docker Development Backups](#docker-development-backups)
8. [Backup Schedule Recommendations](#backup-schedule-recommendations)
9. [Disaster Recovery Checklist](#disaster-recovery-checklist)

---

## Backup Strategy Overview

### Strategy Comparison

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

### RPO/RTO Considerations

- **WAL archiving** alone is not enough — combine with a base backup. WAL replays forward from the last base backup.
- **pg_dump** cannot replay transactions that happened after the dump completed — RPO equals the dump interval.
- **Restore time** for pg_dump scales with data volume and parallelism (`-j` flag). A 100 GB database takes 15–60 minutes with `-j 8`.
- **pg_basebackup** restore is faster than pg_dump for full cluster restores — no SQL parsing, direct file copy.

---

## pg_dump / pg_restore

### Backup Formats

| Format | Flag | Notes |
|---|---|---|
| Plain SQL | `-Fp` (default) | Human-readable, no parallel restore, large files |
| Custom | `-Fc` | Compressed, supports parallel restore with `-j`, best for production |
| Directory | `-Fd` | One file per table, supports parallel dump and restore |
| Tar | `-Ft` | Single tarball, no parallel restore |

**Prefer custom format (`-Fc`) for anything larger than a few MB.** It is smaller, faster to restore, and enables selective table restores.

### Common Commands

```bash
# Full database dump — custom format, compressed
pg_dump -Fc -d "postgres://user:pass@host:5432/mydb" -f mydb_$(date +%Y%m%d_%H%M%S).dump

# Schema only (no data) — useful before migrations
pg_dump -Fc --schema-only -d "postgres://user:pass@host:5432/mydb" -f schema.dump

# Data only (no DDL)
pg_dump -Fc --data-only -d "postgres://user:pass@host:5432/mydb" -f data.dump

# Single table
pg_dump -Fc -t public.orders -d "postgres://user:pass@host:5432/mydb" -f orders.dump

# Multiple tables
pg_dump -Fc -t public.orders -t public.order_items -d "postgres://..." -f orders_tables.dump

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

# Restore schema only
pg_restore --schema-only -d "postgres://user:pass@host:5432/mydb" mydb.dump

# Plain SQL restore (for -Fp dumps)
psql -d "postgres://user:pass@host:5432/mydb" -f mydb.sql
```

### Automated Daily Dump Script

Save as `/usr/local/bin/pg-backup.sh` and run via cron or systemd timer.

```bash
#!/usr/bin/env bash
set -euo pipefail

# Configuration — override via environment
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

# Prune old backups
find "$BACKUP_DIR" -name "*.dump" -mtime "+${KEEP_DAYS}" -delete
echo "[$(date -u +%Y-%m-%dT%H:%M:%SZ)] Pruned backups older than ${KEEP_DAYS} days"
```

Add to root crontab (`crontab -e`):

```cron
# Daily at 02:00 UTC
0 2 * * * DATABASE_URL="postgres://..." BACKUP_DIR=/var/backups/postgres /usr/local/bin/pg-backup.sh >> /var/log/pg-backup.log 2>&1
```

### Go Helper: Shell Out to pg_dump with Progress Logging

```go
package dbbackup

import (
    "context"
    "fmt"
    "log/slog"
    "os/exec"
    "time"
)

type DumpConfig struct {
    DatabaseURL string
    OutputPath  string
    Jobs        int // parallel workers; 0 = single-threaded
}

// RunDump shells out to pg_dump and logs progress. Blocks until complete.
func RunDump(ctx context.Context, cfg DumpConfig) error {
    if cfg.Jobs <= 0 {
        cfg.Jobs = 4
    }

    args := []string{
        "-Fc",
        "-j", fmt.Sprintf("%d", cfg.Jobs),
        "-d", cfg.DatabaseURL,
        "-f", cfg.OutputPath,
    }

    slog.InfoContext(ctx, "pg_dump starting",
        "output", cfg.OutputPath,
        "jobs", cfg.Jobs,
    )

    start := time.Now()
    cmd := exec.CommandContext(ctx, "pg_dump", args...)

    // Capture stderr for progress/error output
    out, err := cmd.CombinedOutput()
    elapsed := time.Since(start)

    if err != nil {
        slog.ErrorContext(ctx, "pg_dump failed",
            "error", err,
            "stderr", string(out),
            "elapsed", elapsed,
        )
        return fmt.Errorf("pg_dump: %w — %s", err, string(out))
    }

    slog.InfoContext(ctx, "pg_dump complete",
        "output", cfg.OutputPath,
        "elapsed", elapsed,
    )
    return nil
}
```

---

## pg_basebackup

Physical backup of the entire PostgreSQL cluster. Produces a binary copy that can be restored directly as a data directory — no SQL parsing, faster than pg_dump for full-cluster restores. Required as the base for WAL archiving / PITR.

### Key Flags

| Flag | Purpose |
|---|---|
| `-D /path/to/backup` | Output directory |
| `-Ft` | Tar format (single file per tablespace) |
| `-z` | gzip compress (combine with `-Ft`) |
| `-Xs` (default) or `-Xstream` | Include WAL via streaming (avoids needing `wal_keep_size`) |
| `-P` | Show progress |
| `-c fast` | Force immediate checkpoint (faster start, brief I/O spike) |
| `-c spread` | Spread checkpoint over time (gentler on production) |
| `-R` | Write `standby.signal` + connection info for replica setup |

### Example Commands

```bash
# Backup to local directory — streaming WAL, progress, fast checkpoint
pg_basebackup \
  -h localhost -p 5432 -U replicator \
  -D /var/backups/postgres/basebackup_$(date +%Y%m%d) \
  -Xs -P -c fast

# Tar format + gzip (single archive, better for remote storage)
pg_basebackup \
  -h localhost -p 5432 -U replicator \
  -D /var/backups/postgres/basebackup_$(date +%Y%m%d) \
  -Ft -z -Xs -P -c spread

# Seed a read replica (writes recovery config automatically)
pg_basebackup \
  -h primary-host -p 5432 -U replicator \
  -D /var/lib/postgresql/16/main \
  -Xs -P -R -c fast
```

**Replication user requirement.** The connecting user needs the `REPLICATION` role:

```sql
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'strongpassword';
```

---

## WAL Archiving and PITR

Write-Ahead Log (WAL) archiving captures every transaction as it happens. Combined with a base backup, it enables Point-in-Time Recovery (PITR) — restoring the database to any moment in history, including 5 minutes before an accidental `DELETE`.

### postgresql.conf Settings

```ini
# Enable WAL archiving
wal_level = replica          # minimum for archiving; use 'logical' for logical replication
archive_mode = on
archive_command = 'cp %p /var/lib/postgresql/wal_archive/%f'
archive_timeout = 60         # seconds; force WAL segment switch if idle (limits RPO)

# For pgBackRest or WAL-G, replace archive_command:
# archive_command = 'pgbackrest --stanza=mydb archive-push %p'
# archive_command = 'wal-g wal-push %p'
```

Requires a server restart after changes.

### Continuous Archiving Setup

1. Create the archive directory and set permissions:

```bash
mkdir -p /var/lib/postgresql/wal_archive
chown postgres:postgres /var/lib/postgresql/wal_archive
chmod 700 /var/lib/postgresql/wal_archive
```

2. Take a base backup to mark the starting point:

```bash
pg_basebackup -D /var/backups/base -Xs -P -c fast
```

3. Verify archiving is working:

```sql
SELECT pg_switch_wal();  -- Forces a WAL segment to archive immediately
SELECT * FROM pg_stat_archiver;
-- Check: last_archived_wal is recent, failed_count = 0
```

### Point-in-Time Recovery (PITR) — PostgreSQL 12+

PostgreSQL 12 removed `recovery.conf`. Recovery settings go directly in `postgresql.conf` and recovery is triggered by a `recovery.signal` file.

**recovery.conf was removed in PG 12.** All recovery parameters now live in `postgresql.conf`.

```ini
# postgresql.conf (on the recovery target server)
restore_command = 'cp /var/lib/postgresql/wal_archive/%f %p'
recovery_target_time = '2026-03-04 14:30:00 UTC'   # stop replaying here
recovery_target_action = 'promote'                  # promote to primary when target reached
```

```bash
# recovery.signal triggers recovery mode (empty file)
touch /var/lib/postgresql/16/main/recovery.signal
```

### Step-by-Step PITR Procedure

**Scenario:** Accidental `DELETE FROM orders WHERE TRUE` at 2026-03-04 14:35 UTC. Recover to 14:30 UTC.

```bash
# 1. Stop PostgreSQL on recovery server (or use separate server)
systemctl stop postgresql

# 2. Clear the data directory
rm -rf /var/lib/postgresql/16/main/*

# 3. Restore the most recent base backup
tar -xzf /var/backups/base/base.tar.gz -C /var/lib/postgresql/16/main/

# 4. Ensure WAL archive is accessible (copy to local path or set restore_command)
# restore_command copies archived WAL segments into pg_wal as PostgreSQL requests them

# 5. Configure recovery in postgresql.conf
cat >> /var/lib/postgresql/16/main/postgresql.conf <<EOF
restore_command = 'cp /var/lib/postgresql/wal_archive/%f %p'
recovery_target_time = '2026-03-04 14:30:00 UTC'
recovery_target_action = 'promote'
EOF

# 6. Create recovery.signal
touch /var/lib/postgresql/16/main/recovery.signal

# 7. Fix permissions
chown -R postgres:postgres /var/lib/postgresql/16/main

# 8. Start PostgreSQL — it will replay WAL up to the target time and promote
systemctl start postgresql

# 9. Watch logs to confirm recovery
journalctl -u postgresql -f
# Look for: "recovery stopping before commit..." and "database system is ready"

# 10. Verify data
psql -c "SELECT COUNT(*) FROM orders;"  -- should show pre-DELETE count

# 11. Remove recovery_target_time from postgresql.conf (to prevent re-entry on next restart)
# Then restart: systemctl restart postgresql
```

---

## Backup Validation

**A backup you have never restored is not a backup.** Test restores regularly — weekly for production, on every schema change for staging.

### Automated Restore Test Script

Restores a dump to a temporary database, runs sanity queries, and exits non-zero on failure.

```bash
#!/usr/bin/env bash
set -euo pipefail

DUMP_FILE="${1:?Usage: $0 <dump_file>}"
TEST_DB="restore_test_$(date +%s)"
DB_HOST="${DB_HOST:-localhost}"
DB_USER="${DB_USER:-postgres}"
ADMIN_URL="postgres://${DB_USER}@${DB_HOST}/postgres"

echo "[validate] Creating temporary database: $TEST_DB"
createdb -h "$DB_HOST" -U "$DB_USER" "$TEST_DB"

cleanup() {
    echo "[validate] Dropping $TEST_DB"
    dropdb -h "$DB_HOST" -U "$DB_USER" "$TEST_DB" 2>/dev/null || true
}
trap cleanup EXIT

echo "[validate] Restoring dump..."
pg_restore -j 4 -d "postgres://${DB_USER}@${DB_HOST}/${TEST_DB}" "$DUMP_FILE"

echo "[validate] Running sanity queries..."
psql -h "$DB_HOST" -U "$DB_USER" -d "$TEST_DB" <<'SQL'
-- Check at least one row exists in key tables
DO $$
DECLARE
    cnt INTEGER;
BEGIN
    SELECT COUNT(*) INTO cnt FROM users;
    IF cnt = 0 THEN RAISE EXCEPTION 'users table is empty'; END IF;

    SELECT COUNT(*) INTO cnt FROM orders;
    IF cnt = 0 THEN RAISE EXCEPTION 'orders table is empty'; END IF;

    RAISE NOTICE 'Sanity checks passed: users=%, orders=%', cnt, cnt;
END;
$$;
SQL

echo "[validate] Backup validation PASSED: $DUMP_FILE"
```

### Go Function: Validate Backup by Row Counts

```go
package dbbackup

import (
    "context"
    "fmt"
    "log/slog"
    "os/exec"
    "time"

    "github.com/jmoiron/sqlx"
    _ "github.com/lib/pq"
)

type TableCheck struct {
    Table    string
    MinRows  int64
}

// ValidateRestore restores a dump to a temp DB and verifies minimum row counts.
// Returns an error if restore fails or any table falls below the minimum.
func ValidateRestore(ctx context.Context, adminURL, dumpPath string, checks []TableCheck) error {
    testDB := fmt.Sprintf("restore_test_%d", time.Now().UnixNano())

    slog.InfoContext(ctx, "backup validation: creating temp db", "name", testDB)

    // Create temp database
    if out, err := exec.CommandContext(ctx, "createdb", "-d", adminURL, testDB).CombinedOutput(); err != nil {
        return fmt.Errorf("createdb: %w — %s", err, string(out))
    }
    defer func() {
        dropCtx := context.Background()
        if out, err := exec.CommandContext(dropCtx, "dropdb", "-d", adminURL, testDB).CombinedOutput(); err != nil {
            slog.WarnContext(dropCtx, "backup validation: failed to drop temp db",
                "name", testDB, "error", err, "output", string(out))
        }
    }()

    // Restore dump
    slog.InfoContext(ctx, "backup validation: restoring dump", "dump", dumpPath)
    restoreURL := fmt.Sprintf("%s/%s", adminURL, testDB)
    if out, err := exec.CommandContext(ctx, "pg_restore", "-j", "4", "-d", restoreURL, dumpPath).CombinedOutput(); err != nil {
        return fmt.Errorf("pg_restore: %w — %s", err, string(out))
    }

    // Connect and run checks
    db, err := sqlx.ConnectContext(ctx, "postgres", restoreURL)
    if err != nil {
        return fmt.Errorf("connect to restored db: %w", err)
    }
    defer db.Close()

    for _, chk := range checks {
        var count int64
        row := db.QueryRowContext(ctx, fmt.Sprintf("SELECT COUNT(*) FROM %s", chk.Table))
        if err := row.Scan(&count); err != nil {
            return fmt.Errorf("count %s: %w", chk.Table, err)
        }
        if count < chk.MinRows {
            return fmt.Errorf("table %s has %d rows, expected >= %d", chk.Table, count, chk.MinRows)
        }
        slog.InfoContext(ctx, "backup validation: table OK",
            "table", chk.Table, "rows", count, "min", chk.MinRows)
    }

    slog.InfoContext(ctx, "backup validation: PASSED", "dump", dumpPath)
    return nil
}
```

Usage:

```go
err := dbbackup.ValidateRestore(ctx, "postgres://postgres@localhost/postgres",
    "/var/backups/mydb_20260304.dump",
    []dbbackup.TableCheck{
        {Table: "users",  MinRows: 1},
        {Table: "orders", MinRows: 1},
    },
)
if err != nil {
    slog.ErrorContext(ctx, "backup validation failed", "error", err)
}
```

### pg_verifybackup (PostgreSQL 13+)

`pg_verifybackup` checks that a base backup is internally consistent before you need it.

```bash
# After pg_basebackup with --manifest-checksums
pg_basebackup -D /var/backups/base -Xs -P --manifest-checksums=sha256

# Verify integrity of the backup
pg_verifybackup /var/backups/base
# Output: "backup successfully verified"
```

**Automate after every base backup** — add `pg_verifybackup` as the next step in your backup script.

---

## Managed Database Backups

### AWS RDS / Aurora

| Feature | Detail |
|---|---|
| Automated snapshots | Daily during backup window; retain 1–35 days |
| Manual snapshots | On-demand; retained until deleted; charged per GB/month above free tier |
| PITR | Enabled by default; restore to any second within retention window |
| Cross-region | Manual snapshot copy to another region (additional storage cost) |
| Export to S3 | Export snapshot data to S3 as Parquet/CSV (useful for analytics) |

```bash
# AWS CLI: create manual snapshot
aws rds create-db-snapshot \
  --db-instance-identifier myapp-prod \
  --db-snapshot-identifier myapp-prod-$(date +%Y%m%d)

# Restore to a new instance from snapshot
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier myapp-prod-restored \
  --db-snapshot-identifier myapp-prod-20260304
```

### Google Cloud SQL

| Feature | Detail |
|---|---|
| Automated backups | Daily; retain 7 backups by default (up to 365) |
| On-demand backups | Manual trigger; counted against retention limit |
| PITR | Enabled per-instance; uses write-ahead logs; restore to any second |
| Cross-region | Point-in-time clones to another region (Enterprise Plus tier) |

```bash
# gcloud CLI: create on-demand backup
gcloud sql backups create --instance=myapp-prod

# List backups
gcloud sql backups list --instance=myapp-prod

# Restore to a new instance
gcloud sql instances clone myapp-prod myapp-prod-restored \
  --point-in-time='2026-03-04T14:30:00.000Z'
```

### Supabase

| Feature | Detail |
|---|---|
| Free plan | Daily backups, 7-day retention, no PITR |
| Pro plan | Daily backups, 30-day retention, PITR (to any second within window) |
| Storage | Included in plan; no additional backup charges |
| Restore UI | One-click restore from dashboard to a new project |
| Download | Download SQL dump from dashboard (logical backup) |

**Note:** Supabase PITR uses WAL archiving internally. On Pro, enable PITR in Settings > Database > Point in Time Recovery.

### Comparison Summary

| Provider | Automated | PITR | Retention | Cross-Region | Extra Cost |
|---|---|---|---|---|---|
| AWS RDS | Yes (daily) | Yes (to second) | 1–35 days | Manual copy | Snapshot storage + copy |
| Cloud SQL | Yes (daily) | Yes (to second) | 7–365 backups | Enterprise Plus | Minimal |
| Supabase | Yes (daily) | Pro+ only | 7–30 days | No | Pro plan required |

---

## Docker Development Backups

### Volume Backup

```bash
# Identify the volume name
docker volume ls | grep postgres

# Backup volume to a tar archive
docker run --rm \
  -v my_project_postgres_data:/source:ro \
  -v $(pwd):/backup \
  alpine tar czf /backup/pg_volume_$(date +%Y%m%d).tar.gz -C /source .

# Restore volume from archive
docker run --rm \
  -v my_project_postgres_data:/target \
  -v $(pwd):/backup \
  alpine tar xzf /backup/pg_volume_20260304.tar.gz -C /target
```

### docker exec pg_dump Pattern

Most convenient for development — no volume mounts required.

```bash
# Dump from running container
docker exec myapp_postgres pg_dump \
  -U postgres mydb \
  -Fc > mydb_dev_$(date +%Y%m%d).dump

# Restore to running container
docker exec -i myapp_postgres pg_restore \
  -U postgres -d mydb < mydb_dev_20260304.dump

# Quick plain SQL dump
docker exec myapp_postgres pg_dump -U postgres mydb > mydb_dev.sql
```

### docker-compose with Backup Sidecar

```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: devpassword
    volumes:
      - postgres_data:/var/lib/postgresql/data

  pg-backup:
    image: postgres:16-alpine
    depends_on:
      - postgres
    environment:
      PGPASSWORD: devpassword
    volumes:
      - ./backups:/backups
    # Run dump every 6 hours via simple shell loop
    command: >
      sh -c "while true; do
        pg_dump -h postgres -U postgres mydb -Fc
          > /backups/mydb_$$(date +%Y%m%d_%H%M%S).dump
          && echo backup done;
        find /backups -name '*.dump' -mtime +7 -delete;
        sleep 21600;
      done"

volumes:
  postgres_data:
```

Cross-reference: for PostgreSQL Docker setup with proper health checks and persistent volumes, see [gin-deploy/references/docker-compose.md](../../gin-deploy/references/docker-compose.md).

---

## Backup Schedule Recommendations

### Schedule by Environment

| Environment | Strategy | Frequency | Retention | Notes |
|---|---|---|---|---|
| **Development** | pg_dump (plain or custom) | Daily | 7 days | Manual or docker sidecar |
| **Staging** | pg_dump (custom, parallel) | Daily | 30 days | Mirrors prod dump process |
| **Staging** | WAL archiving | Continuous | 7 days | Optional but validates PITR setup |
| **Production** | pg_basebackup + WAL archiving | Daily base + continuous WAL | 90 days | Full PITR window |
| **Production** | Cross-region copy | Weekly | 90 days | Offsite disaster recovery |

### pg_cron Integration

Run pg_dump from inside PostgreSQL using `pg_cron` (requires the extension):

```sql
-- Enable pg_cron (postgresql.conf: shared_preload_libraries = 'pg_cron')
CREATE EXTENSION IF NOT EXISTS pg_cron;

-- Schedule a shell backup script at 02:00 UTC every day
-- NOTE: pg_cron runs SQL, not shell. Use it to call a stored procedure that calls pg_notify
-- or use a dedicated cron job on the host for pg_dump.

-- pg_cron is better suited for SQL maintenance tasks:
-- Vacuum, analyze, archival inserts, partition creation
SELECT cron.schedule('nightly-analyze', '0 3 * * *', 'ANALYZE;');
SELECT cron.schedule('monthly-partition', '0 0 1 * *',
    $$CALL create_monthly_partition('events', now() + interval '1 month')$$);
```

**For pg_dump itself, use host cron or a Kubernetes CronJob.** pg_cron does not execute shell commands — it runs SQL only.

### Kubernetes CronJob for pg_dump

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
spec:
  schedule: "0 2 * * *"   # 02:00 UTC daily
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: pg-backup
            image: postgres:16-alpine
            env:
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
            command:
            - /bin/sh
            - -c
            - |
              pg_dump -h postgres-service -U myuser mydb -Fc \
                -f /backups/mydb_$(date +%Y%m%d_%H%M%S).dump
            volumeMounts:
            - name: backup-storage
              mountPath: /backups
          restartPolicy: OnFailure
          volumes:
          - name: backup-storage
            persistentVolumeClaim:
              claimName: backup-pvc
```

---

## Disaster Recovery Checklist

Run this checklist quarterly and after every major infrastructure change.

### Documentation

- [ ] Backup runbook is written and stored in the team wiki
- [ ] Restore procedure is documented step-by-step (not just "restore from backup")
- [ ] `recovery_target_time` syntax and PITR steps are documented for the current PostgreSQL version
- [ ] Runbook identifies who is responsible for executing recovery during an incident
- [ ] Contact list for DBA / cloud provider support is current

### Tested Restores

- [ ] Full restore drill completed in the last 30 days (for production)
- [ ] PITR drill completed in the last 90 days (verify WAL replay works end-to-end)
- [ ] Selective table restore tested (pg_dump `-t` flag for logical backups)
- [ ] Restore time measured and documented (do you meet your RTO?)
- [ ] Restored database passed all sanity checks (row counts, FK integrity, application smoke test)

### Monitoring and Alerting

- [ ] Alert fires if `pg_stat_archiver.failed_count` increases (WAL archive failures)
- [ ] Alert fires if last successful backup is older than 25 hours
- [ ] Backup file sizes are monitored — a sudden drop in size indicates a problem
- [ ] `pg_verifybackup` runs after every base backup and failures are alerted

### Offsite / Cross-Region

- [ ] At least one backup copy exists in a different region or cloud account
- [ ] Offsite copy restore has been tested at least once
- [ ] Access credentials for offsite storage are stored separately from primary systems

### Team Readiness

- [ ] At least two team members can execute the restore procedure independently
- [ ] On-call engineer has access to backup storage and recovery documentation
- [ ] Recovery procedure has been rehearsed (tabletop or live drill) in the last 6 months

### Security

- [ ] Backup storage access is restricted to the minimum required principals
- [ ] Backups containing PII are encrypted at rest (AES-256 or provider default)
- [ ] Backup credentials are rotated and not committed to source control
- [ ] Backup access logs are retained for audit purposes
