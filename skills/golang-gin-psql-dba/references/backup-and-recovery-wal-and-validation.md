# Backup and Recovery — WAL Archiving, PITR, and Validation

## WAL Archiving and PITR

WAL archiving captures every transaction as it happens. Combined with a base backup, it enables Point-in-Time Recovery (PITR).

### postgresql.conf Settings

```ini
wal_level = replica
archive_mode = on
archive_command = 'cp %p /var/lib/postgresql/wal_archive/%f'
archive_timeout = 60   # seconds; limits RPO when idle

# For pgBackRest or WAL-G:
# archive_command = 'pgbackrest --stanza=mydb archive-push %p'
# archive_command = 'wal-g wal-push %p'
```

Requires a server restart after changes.

### Continuous Archiving Setup

```bash
# 1. Create archive directory
mkdir -p /var/lib/postgresql/wal_archive
chown postgres:postgres /var/lib/postgresql/wal_archive
chmod 700 /var/lib/postgresql/wal_archive

# 2. Take a base backup
pg_basebackup -D /var/backups/base -Xs -P -c fast
```

```sql
-- 3. Verify archiving is working
SELECT pg_switch_wal();
SELECT * FROM pg_stat_archiver;
-- Check: last_archived_wal is recent, failed_count = 0
```

### Point-in-Time Recovery (PITR) — PostgreSQL 12+

`recovery.conf` was removed in PG 12. All recovery parameters now live in `postgresql.conf`.

```ini
# postgresql.conf (on the recovery target server)
restore_command = 'cp /var/lib/postgresql/wal_archive/%f %p'
recovery_target_time = '2026-03-04 14:30:00 UTC'
recovery_target_action = 'promote'
```

```bash
# recovery.signal triggers recovery mode (empty file)
touch /var/lib/postgresql/16/main/recovery.signal
```

### Step-by-Step PITR Procedure

**Scenario:** Accidental `DELETE FROM orders WHERE TRUE` at 14:35 UTC. Recover to 14:30 UTC.

```bash
# 1. Stop PostgreSQL on recovery server
systemctl stop postgresql

# 2. Clear the data directory
rm -rf /var/lib/postgresql/16/main/*

# 3. Restore the most recent base backup
tar -xzf /var/backups/base/base.tar.gz -C /var/lib/postgresql/16/main/

# 4. Configure recovery in postgresql.conf
cat >> /var/lib/postgresql/16/main/postgresql.conf <<EOF
restore_command = 'cp /var/lib/postgresql/wal_archive/%f %p'
recovery_target_time = '2026-03-04 14:30:00 UTC'
recovery_target_action = 'promote'
EOF

# 5. Create recovery.signal
touch /var/lib/postgresql/16/main/recovery.signal

# 6. Fix permissions
chown -R postgres:postgres /var/lib/postgresql/16/main

# 7. Start PostgreSQL — replays WAL up to target time, then promotes
systemctl start postgresql

# 8. Verify data
psql -c "SELECT COUNT(*) FROM orders;"

# 9. Remove recovery_target_time from postgresql.conf, then restart
```


> For backup validation (automated restore test script, Go ValidateRestore, pg_verifybackup), see [backup-and-recovery-managed-and-schedules.md](backup-and-recovery-managed-and-schedules.md).
