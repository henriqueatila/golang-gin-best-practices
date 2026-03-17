# Backup and Recovery — Managed Services, Docker, Schedules, and DR Checklist

## Managed Database Backups

### AWS RDS / Aurora

| Feature | Detail |
|---|---|
| Automated snapshots | Daily during backup window; retain 1–35 days |
| Manual snapshots | On-demand; retained until deleted |
| PITR | Enabled by default; restore to any second within retention window |
| Cross-region | Manual snapshot copy to another region |
| Export to S3 | Export snapshot data to S3 as Parquet/CSV |

```bash
aws rds create-db-snapshot --db-instance-identifier myapp-prod --db-snapshot-identifier myapp-prod-$(date +%Y%m%d)
aws rds restore-db-instance-from-db-snapshot --db-instance-identifier myapp-prod-restored --db-snapshot-identifier myapp-prod-20260304
```

### Google Cloud SQL

| Feature | Detail |
|---|---|
| Automated backups | Daily; retain 7 backups by default (up to 365) |
| PITR | Enabled per-instance; restore to any second |
| Cross-region | Point-in-time clones (Enterprise Plus tier) |

```bash
gcloud sql backups create --instance=myapp-prod
gcloud sql instances clone myapp-prod myapp-prod-restored \
  --point-in-time='2026-03-04T14:30:00.000Z'
```

### Supabase

| Feature | Detail |
|---|---|
| Free plan | Daily backups, 7-day retention, no PITR |
| Pro plan | Daily backups, 30-day retention, PITR (to any second) |
| Restore UI | One-click restore from dashboard |

### Comparison Summary

| Provider | Automated | PITR | Retention | Cross-Region |
|---|---|---|---|---|
| AWS RDS | Yes (daily) | Yes (to second) | 1–35 days | Manual copy |
| Cloud SQL | Yes (daily) | Yes (to second) | 7–365 backups | Enterprise Plus |
| Supabase | Yes (daily) | Pro+ only | 7–30 days | No |

---

## Docker Development Backups

### docker exec pg_dump Pattern

```bash
# Dump from running container
docker exec myapp_postgres pg_dump \
  -U postgres mydb \
  -Fc > mydb_dev_$(date +%Y%m%d).dump

# Restore to running container
docker exec -i myapp_postgres pg_restore \
  -U postgres -d mydb < mydb_dev_20260304.dump
```

### docker-compose with Backup Sidecar

Add a `pg-backup` sidecar service (`image: postgres:16-alpine`, `depends_on: postgres`, volume `./backups:/backups`). The command runs a `while true` loop: `pg_dump -h postgres -Fc > /backups/mydb_$(date).dump`, deletes dumps older than 7 days, then `sleep 21600` (6 hours).

---

## Backup Schedule Recommendations

| Environment | Strategy | Frequency | Retention |
|---|---|---|---|
| **Development** | pg_dump | Daily | 7 days |
| **Staging** | pg_dump (custom, parallel) | Daily | 30 days |
| **Production** | pg_basebackup + WAL archiving | Daily base + continuous WAL | 90 days |
| **Production** | Cross-region copy | Weekly | 90 days |

### pg_cron for SQL Maintenance

```sql
-- pg_cron runs SQL only — use host cron for pg_dump
SELECT cron.schedule('nightly-analyze', '0 3 * * *', 'ANALYZE;');
SELECT cron.schedule('monthly-partition', '0 0 1 * *',
    $$CALL create_monthly_partition('events', now() + interval '1 month')$$);
```

**For pg_dump itself, use host cron or a Kubernetes CronJob.**

For a complete Kubernetes CronJob spec running pg_dump, see the **golang-gin-deploy** skill.

---

## Disaster Recovery Checklist

### Documentation
- [ ] Backup runbook written and stored in team wiki
- [ ] Restore procedure documented step-by-step
- [ ] `recovery_target_time` syntax documented for current PostgreSQL version
- [ ] Contact list for DBA / cloud provider support is current

### Tested Restores
- [ ] Full restore drill completed in last 30 days (production)
- [ ] PITR drill completed in last 90 days
- [ ] Restore time measured and documented (meets RTO?)
- [ ] Restored database passed all sanity checks

### Monitoring and Alerting
- [ ] Alert fires if `pg_stat_archiver.failed_count` increases
- [ ] Alert fires if last successful backup is older than 25 hours
- [ ] `pg_verifybackup` runs after every base backup

### Offsite / Cross-Region
- [ ] At least one backup copy exists in a different region or cloud account
- [ ] Offsite copy restore has been tested at least once
- [ ] Access credentials for offsite storage stored separately from primary systems

### Security
- [ ] Backup storage access restricted to minimum required principals
- [ ] Backups containing PII encrypted at rest (AES-256 or provider default)
- [ ] Backup credentials rotated and not committed to source control
