# Extensions Toolkit — Setup, pg_cron

## Extension Management in Migrations

Always use `IF NOT EXISTS` — migrations must be idempotent:

```sql
-- db/migrations/000001_setup_extensions.up.sql
CREATE EXTENSION IF NOT EXISTS "pgcrypto";
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";
```

```sql
-- db/migrations/000001_setup_extensions.down.sql
DROP EXTENSION IF EXISTS "pg_trgm";
DROP EXTENSION IF EXISTS "uuid-ossp";
DROP EXTENSION IF EXISTS "pgcrypto";
```

Rules:
- Install extensions in the earliest migration possible — other migrations may depend on them.
- Extensions owned by `superuser` — in managed DBs (RDS, Cloud SQL), use the platform's pre-installed list.

### Docker: Installing Extensions

```dockerfile
FROM postgres:16

RUN apt-get update && apt-get install -y \
    postgresql-16-cron \
    postgresql-16-partman \
    postgresql-16-audit \
    && rm -rf /var/lib/apt/lists/*

# pg_cron requires shared_preload_libraries
CMD ["postgres", "-c", "shared_preload_libraries=pg_cron,pg_stat_statements"]
```

### golang-migrate Example

```sql
-- db/migrations/000001_extensions.up.sql
CREATE EXTENSION IF NOT EXISTS "pgcrypto";
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";
CREATE EXTENSION IF NOT EXISTS "pg_stat_statements";
CREATE EXTENSION IF NOT EXISTS "pg_partman" SCHEMA partman;
-- CREATE EXTENSION IF NOT EXISTS "pg_cron";  -- only after shared_preload_libraries is set
```

---

## pg_cron — Scheduled Jobs

### Setup

```ini
# postgresql.conf
shared_preload_libraries = 'pg_cron'
cron.database_name = 'myapp'   # database where cron.job table lives
```

After restart:

```sql
CREATE EXTENSION IF NOT EXISTS pg_cron;
GRANT USAGE ON SCHEMA cron TO myapp_user;
```

### Schedule Syntax

| Expression | Meaning |
|---|---|
| `0 3 * * *` | Every day at 03:00 |
| `*/15 * * * *` | Every 15 minutes |
| `0 0 * * 0` | Every Sunday at midnight |
| `@daily` | Alias for `0 0 * * *` |

### Examples

```sql
-- Nightly cleanup of soft-deleted records
SELECT cron.schedule('nightly-cleanup', '0 3 * * *', $$
    DELETE FROM users WHERE deleted_at < NOW() - INTERVAL '90 days';
$$);

-- Hourly aggregation into a stats table
SELECT cron.schedule('hourly-stats', '0 * * * *', $$
    INSERT INTO hourly_event_counts (hour, event_type, count)
    SELECT date_trunc('hour', created_at), event_type, COUNT(*)
    FROM events
    WHERE created_at >= NOW() - INTERVAL '2 hours'
      AND created_at < date_trunc('hour', NOW())
    GROUP BY 1, 2
    ON CONFLICT (hour, event_type) DO UPDATE SET count = EXCLUDED.count;
$$);

-- Partition maintenance
SELECT cron.schedule('partman-maintenance', '*/30 * * * *', 'CALL partman.run_maintenance_proc()');

-- Manage jobs
SELECT jobid, jobname, schedule, command, active FROM cron.job;
UPDATE cron.job SET active = false WHERE jobname = 'hourly-stats';
SELECT cron.unschedule('nightly-cleanup');
```

### Monitoring Job Runs

```sql
-- Failed jobs in the last 24 hours
SELECT jobname, start_time, return_message
FROM cron.job_run_details r
JOIN cron.job j ON j.jobid = r.jobid
WHERE r.status = 'failed'
  AND r.start_time > NOW() - INTERVAL '24 hours';
```

### Go Helper — Manage Cron Jobs via SQL

```go
// UpsertCronJob ensures a named cron job exists with the given schedule and command.
func UpsertCronJob(ctx context.Context, db *sqlx.DB, name, schedule, command string, logger *slog.Logger) error {
    _, err := db.ExecContext(ctx, `SELECT cron.unschedule(jobname) FROM cron.job WHERE jobname = $1`, name)
    if err != nil {
        return fmt.Errorf("unschedule %q: %w", name, err)
    }
    _, err = db.ExecContext(ctx, `SELECT cron.schedule($1, $2, $3)`, name, schedule, command)
    if err != nil {
        return fmt.Errorf("schedule %q: %w", name, err)
    }
    logger.Info("cron job registered", "name", name, "schedule", schedule)
    return nil
}
```
