# PostgreSQL Extensions Toolkit

A practical reference for the PostgreSQL extension ecosystem in Go/Gin APIs. Covers installation, migration management, and Go integration patterns for the seven most operationally important extensions: pg_cron (scheduled jobs), pg_partman (partition management), pg_trgm (fuzzy search), pg_stat_statements (query statistics), pgcrypto (encryption), uuid-ossp (UUID generation), and pgAudit (audit logging). All Go examples use `sqlx` with raw SQL, `log/slog`, and `context.Context`. No GORM.

## Table of Contents

1. [Extension Management in Migrations](#extension-management-in-migrations)
2. [pg_cron — Scheduled Jobs](#pg_cron--scheduled-jobs)
3. [pg_partman — Partition Management](#pg_partman--partition-management)
4. [pg_trgm — Trigram Matching](#pg_trgm--trigram-matching)
5. [pg_stat_statements — Query Statistics](#pg_stat_statements--query-statistics)
6. [pgcrypto — Encryption and Hashing](#pgcrypto--encryption-and-hashing)
7. [uuid-ossp — UUID Generation](#uuid-ossp--uuid-generation)
8. [pgAudit — Audit Logging](#pgaudit--audit-logging)
9. [Extension Compatibility Matrix](#extension-compatibility-matrix)

---

## Extension Management in Migrations

### CREATE EXTENSION in Migration Files

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

**Rules:**
- Install extensions in the earliest migration possible — other migrations may depend on them.
- Extensions owned by `superuser` — in managed DBs (RDS, Cloud SQL), use the platform's pre-installed list.
- Extensions live in a specific schema (default: `public`). If using a custom schema, add `WITH SCHEMA myschema`.

### Versioning Extensions

```sql
-- Check current version
SELECT extname, extversion FROM pg_extension WHERE extname = 'pg_trgm';

-- Update extension to a specific version
ALTER EXTENSION pg_trgm UPDATE TO '1.6';

-- Update to latest available
ALTER EXTENSION pg_trgm UPDATE;
```

### Extension Dependencies

Some extensions depend on others. PostgreSQL handles this automatically, but document it:

```sql
-- pg_partman depends on pg_jobmon (optional) and pgcrypto
CREATE EXTENSION IF NOT EXISTS "pgcrypto";
CREATE EXTENSION IF NOT EXISTS "pg_partman" SCHEMA partman;
```

### Docker: Installing Extensions

Custom extensions (pg_cron, pg_partman, pgAudit) require OS packages in the Dockerfile:

```dockerfile
FROM postgres:16

# Install contrib extensions (pg_trgm, pgcrypto, uuid-ossp included in contrib)
# Install pg_cron and pg_partman from pgdg
RUN apt-get update && apt-get install -y \
    postgresql-16-cron \
    postgresql-16-partman \
    postgresql-16-audit \
    && rm -rf /var/lib/apt/lists/*
```

Or using the official `postgres` image with `pgdg` repo:

```dockerfile
FROM postgres:16-bullseye

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        postgresql-16-cron \
        postgresql-16-partman \
    && rm -rf /var/lib/apt/lists/*

# pg_cron requires shared_preload_libraries
CMD ["postgres", "-c", "shared_preload_libraries=pg_cron,pg_stat_statements"]
```

### golang-migrate Example with Extension Setup

```sql
-- db/migrations/000001_extensions.up.sql
-- Must run before any migration that uses these extensions.
-- pg_cron requires shared_preload_libraries (needs server restart — handle at infra level).
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

pg_cron requires a server-level configuration change — it cannot be enabled at runtime:

```
# postgresql.conf
shared_preload_libraries = 'pg_cron'
cron.database_name = 'myapp'   # database where cron.job table lives
```

After adding `shared_preload_libraries`, restart PostgreSQL, then:

```sql
CREATE EXTENSION IF NOT EXISTS pg_cron;
GRANT USAGE ON SCHEMA cron TO myapp_user;
```

### Schedule Syntax (Cron Format)

```
┌─── minute (0–59)
│  ┌─── hour (0–23)
│  │  ┌─── day of month (1–31)
│  │  │  ┌─── month (1–12)
│  │  │  │  ┌─── day of week (0–7, 0 and 7 = Sunday)
│  │  │  │  │
*  *  *  *  *
```

| Expression | Meaning |
|---|---|
| `0 3 * * *` | Every day at 03:00 |
| `*/15 * * * *` | Every 15 minutes |
| `0 0 * * 0` | Every Sunday at midnight |
| `0 * * * *` | Every hour on the hour |
| `@daily` | Alias for `0 0 * * *` |

### Examples

**Nightly cleanup of soft-deleted records:**

```sql
SELECT cron.schedule(
    'nightly-cleanup',
    '0 3 * * *',
    $$
        DELETE FROM users
        WHERE deleted_at < NOW() - INTERVAL '90 days';
    $$
);
```

**Hourly aggregation into a stats table:**

```sql
SELECT cron.schedule(
    'hourly-stats',
    '0 * * * *',
    $$
        INSERT INTO hourly_event_counts (hour, event_type, count)
        SELECT
            date_trunc('hour', created_at) AS hour,
            event_type,
            COUNT(*) AS count
        FROM events
        WHERE created_at >= NOW() - INTERVAL '2 hours'
          AND created_at < date_trunc('hour', NOW())
        GROUP BY 1, 2
        ON CONFLICT (hour, event_type) DO UPDATE
            SET count = EXCLUDED.count;
    $$
);
```

**Partition maintenance (delegate to pg_partman):**

```sql
SELECT cron.schedule(
    'partman-maintenance',
    '*/30 * * * *',
    'CALL partman.run_maintenance_proc()'
);
```

**Manage jobs:**

```sql
-- List all scheduled jobs
SELECT jobid, jobname, schedule, command, active FROM cron.job;

-- Disable a job
UPDATE cron.job SET active = false WHERE jobname = 'hourly-stats';

-- Delete a job
SELECT cron.unschedule('nightly-cleanup');
-- Or by id:
SELECT cron.unschedule(jobid) FROM cron.job WHERE jobname = 'nightly-cleanup';
```

### Monitoring Job Runs

```sql
-- Last 20 job runs with status
SELECT
    j.jobname,
    r.start_time,
    r.end_time,
    r.status,
    r.return_message
FROM cron.job_run_details r
JOIN cron.job j ON j.jobid = r.jobid
ORDER BY r.start_time DESC
LIMIT 20;

-- Failed jobs in the last 24 hours
SELECT jobname, start_time, return_message
FROM cron.job_run_details r
JOIN cron.job j ON j.jobid = r.jobid
WHERE r.status = 'failed'
  AND r.start_time > NOW() - INTERVAL '24 hours';
```

### Go Helper — Manage Cron Jobs via SQL

```go
// internal/repository/cron.go
package repository

import (
    "context"
    "fmt"
    "log/slog"

    "github.com/jmoiron/sqlx"
)

type CronJob struct {
    JobID    int64  `db:"jobid"`
    JobName  string `db:"jobname"`
    Schedule string `db:"schedule"`
    Command  string `db:"command"`
    Active   bool   `db:"active"`
}

// UpsertCronJob ensures a named cron job exists with the given schedule and command.
// Idempotent: updates schedule/command if the job already exists.
func UpsertCronJob(ctx context.Context, db *sqlx.DB, name, schedule, command string, logger *slog.Logger) error {
    // Unschedule old version if it exists (pg_cron has no upsert)
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

// ListFailedJobs returns cron jobs that failed in the last N hours.
func ListFailedJobs(ctx context.Context, db *sqlx.DB, hours int) ([]CronJob, error) {
    const q = `
        SELECT j.jobid, j.jobname, j.schedule, j.command, j.active
        FROM cron.job j
        WHERE j.jobid IN (
            SELECT DISTINCT jobid FROM cron.job_run_details
            WHERE status = 'failed'
              AND start_time > NOW() - ($1 || ' hours')::INTERVAL
        )
    `
    var jobs []CronJob
    if err := db.SelectContext(ctx, &jobs, q, hours); err != nil {
        return nil, fmt.Errorf("list failed jobs: %w", err)
    }
    return jobs, nil
}
```

---

## pg_partman — Partition Management

### Setup

```sql
CREATE SCHEMA IF NOT EXISTS partman;
CREATE EXTENSION IF NOT EXISTS pg_partman SCHEMA partman;
```

Grant access for maintenance:

```sql
GRANT ALL ON SCHEMA partman TO myapp_user;
GRANT ALL ON ALL TABLES IN SCHEMA partman TO myapp_user;
```

### create_parent() Function

```
partman.create_parent(
    p_parent_table   TEXT,     -- fully qualified: 'public.events'
    p_control        TEXT,     -- partition key column: 'created_at'
    p_type           TEXT,     -- 'native' (PG 10+) or 'partman' (trigger-based, legacy)
    p_interval       TEXT,     -- 'monthly', 'weekly', 'daily', or integer for serial
    p_premake        INT,      -- how many future partitions to pre-create (default 4)
    p_start_partition TEXT     -- optional: first partition start date
)
```

### Time-Based Partitioning

```sql
-- Create the parent table (no PARTITION BY yet — partman handles it)
CREATE TABLE events (
    id         BIGINT GENERATED ALWAYS AS IDENTITY,
    tenant_id  UUID          NOT NULL,
    event_type TEXT          NOT NULL,
    payload    JSONB,
    created_at TIMESTAMPTZ   NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

-- Hand off to partman
SELECT partman.create_parent(
    p_parent_table  => 'public.events',
    p_control       => 'created_at',
    p_type          => 'native',
    p_interval      => 'monthly',
    p_premake       => 3   -- create 3 months ahead
);
```

### Serial-Based Partitioning

```sql
CREATE TABLE orders (
    id         BIGINT GENERATED ALWAYS AS IDENTITY,
    user_id    UUID   NOT NULL,
    total      NUMERIC(19,4),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (id);

SELECT partman.create_parent(
    p_parent_table => 'public.orders',
    p_control      => 'id',
    p_type         => 'native',
    p_interval     => '1000000'  -- partition every 1 million rows
);
```

### Retention Policies

Configure in `partman.part_config`:

```sql
-- Keep 12 months of data, drop older partitions automatically
UPDATE partman.part_config
SET
    retention             = '12 months',
    retention_keep_table  = false,   -- false = DROP, true = detach and keep
    retention_keep_index  = false,   -- drop indexes along with table
    infinite_time_partitions = true  -- continue creating future partitions indefinitely
WHERE parent_table = 'public.events';
```

| Setting | Meaning |
|---|---|
| `retention` | Interval after which partitions are eligible for removal |
| `retention_keep_table` | `false` = DROP TABLE; `true` = DETACH PARTITION (data preserved) |
| `retention_keep_index` | Keep indexes when detaching (only applies when `retention_keep_table = true`) |
| `infinite_time_partitions` | Keep pre-creating partitions past `premake` window |

### Maintenance: run_maintenance_proc()

pg_partman needs periodic maintenance to create future partitions and enforce retention:

```sql
-- Run manually (or via pg_cron)
CALL partman.run_maintenance_proc();

-- Check partition configuration
SELECT parent_table, control, partition_interval, premake, retention
FROM partman.part_config;

-- List all child partitions
SELECT
    child.relname         AS partition_name,
    pg_size_pretty(pg_total_relation_size(child.oid)) AS size
FROM pg_inherits
JOIN pg_class parent ON pg_inherits.inhparent = parent.oid
JOIN pg_class child  ON pg_inherits.inhrelid  = child.oid
WHERE parent.relname = 'events'
ORDER BY child.relname;
```

### Complete Example: Events Table with pg_cron Integration

```sql
-- db/migrations/000010_events_partitioned.up.sql

-- 1. Enable required extensions
CREATE EXTENSION IF NOT EXISTS pg_partman SCHEMA partman;
-- pg_cron must be in shared_preload_libraries (done at infra level)

-- 2. Create partitioned parent table
CREATE TABLE events (
    id         BIGINT GENERATED ALWAYS AS IDENTITY,
    tenant_id  UUID         NOT NULL,
    event_type TEXT         NOT NULL,
    payload    JSONB,
    created_at TIMESTAMPTZ  NOT NULL DEFAULT now(),
    PRIMARY KEY (id, created_at)   -- partition key must be in PK for native partitioning
) PARTITION BY RANGE (created_at);

-- 3. Index on each partition (partman propagates to new partitions)
CREATE INDEX ON events (tenant_id, created_at);
CREATE INDEX ON events USING gin (payload jsonb_path_ops);

-- 4. Hand off to partman
SELECT partman.create_parent(
    p_parent_table => 'public.events',
    p_control      => 'created_at',
    p_type         => 'native',
    p_interval     => 'monthly',
    p_premake      => 3
);

-- 5. Set 12-month retention
UPDATE partman.part_config
SET retention = '12 months', retention_keep_table = false
WHERE parent_table = 'public.events';

-- 6. Schedule maintenance via pg_cron (every 30 minutes)
SELECT cron.schedule('partman-maintenance', '*/30 * * * *', 'CALL partman.run_maintenance_proc()');
```

```sql
-- db/migrations/000010_events_partitioned.down.sql
SELECT cron.unschedule('partman-maintenance');
UPDATE partman.part_config SET retention = NULL WHERE parent_table = 'public.events';
SELECT partman.undo_partition('public.events', p_keep_table => false);
DROP TABLE IF EXISTS events;
```

---

## pg_trgm — Trigram Matching

### Setup

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;
```

Contrib module — available on all standard PostgreSQL installations and managed DBs.

### Key Functions

| Function | Returns | Use Case |
|---|---|---|
| `similarity(a, b)` | float (0–1) | Overall string similarity |
| `word_similarity(a, b)` | float (0–1) | Best matching substring |
| `strict_word_similarity(a, b)` | float (0–1) | Whole-word matching |
| `show_trgm(text)` | text[] | Debug: show trigrams |

```sql
SELECT similarity('hello world', 'hello wrold');   -- ~0.5
SELECT word_similarity('world', 'hello world');    -- 1.0
SELECT show_trgm('gin');                           -- {"  g"," gi","gin","in "}
```

**Similarity threshold** (controls `%` operator):

```sql
SET pg_trgm.similarity_threshold = 0.3;  -- default 0.3; lower = more results
SELECT 'hello' % 'helo';                 -- true if similarity >= threshold
```

### GIN Index for LIKE '%pattern%'

```sql
-- Accelerates LIKE '%pattern%' and ILIKE '%pattern%' queries
CREATE INDEX idx_products_name_trgm
    ON products USING gin (name gin_trgm_ops);

-- Query using the index
SELECT id, name
FROM products
WHERE name ILIKE '%wireless headphones%';
```

### GiST Index for Similarity Search

```sql
-- Accelerates % (similarity) operator and ORDER BY similarity()
CREATE INDEX idx_users_username_trgm
    ON users USING gist (username gist_trgm_ops);

-- Fuzzy search: users with username similar to 'johndoe'
SELECT username, similarity(username, 'johndoe') AS score
FROM users
WHERE username % 'johndoe'
ORDER BY score DESC
LIMIT 10;
```

**GIN vs GiST for pg_trgm:**
- **GIN** — faster for `LIKE`/`ILIKE` queries; larger index size; slower to build
- **GiST** — smaller index; faster builds; better for `ORDER BY similarity()` and KNN-style queries

### Fuzzy Search in Go

```go
// internal/repository/product_repo.go
package repository

import (
    "context"
    "fmt"

    "github.com/jmoiron/sqlx"
)

type Product struct {
    ID         string  `db:"id"`
    Name       string  `db:"name"`
    Similarity float64 `db:"similarity"`
}

// SearchProducts returns products matching query using trigram similarity.
// minSimilarity: 0.1 (broad) to 0.9 (strict). 0.3 is a good default.
func SearchProducts(ctx context.Context, db *sqlx.DB, query string, minSimilarity float64, limit int) ([]Product, error) {
    const q = `
        SELECT
            id,
            name,
            similarity(name, $1) AS similarity
        FROM products
        WHERE name % $1
           OR name ILIKE '%' || $1 || '%'
        ORDER BY similarity DESC, name
        LIMIT $3
    `
    // Set threshold for this session
    if _, err := db.ExecContext(ctx, `SET pg_trgm.similarity_threshold = $1`, minSimilarity); err != nil {
        return nil, fmt.Errorf("set trgm threshold: %w", err)
    }

    var products []Product
    if err := db.SelectContext(ctx, &products, q, query, minSimilarity, limit); err != nil {
        return nil, fmt.Errorf("search products: %w", err)
    }
    return products, nil
}
```

---

## pg_stat_statements — Query Statistics

### Setup

Requires `shared_preload_libraries` (server restart needed):

```
# postgresql.conf
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.track = all         # all | top | none
pg_stat_statements.max = 10000         # max tracked statements
pg_stat_statements.track_utility = off # skip VACUUM, COPY, etc.
```

Then in the database:

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

Available on all managed DBs (RDS, Cloud SQL, Supabase) — often pre-enabled.

### Key Columns

| Column | Type | Meaning |
|---|---|---|
| `query` | text | Normalized query text (params replaced with `$1`, `$2`) |
| `calls` | bigint | Total executions |
| `total_exec_time` | float | Total time (ms) across all calls |
| `mean_exec_time` | float | Average time per call (ms) |
| `stddev_exec_time` | float | Variance — high = inconsistent performance |
| `rows` | bigint | Total rows returned/affected |
| `shared_blks_hit` | bigint | Buffer cache hits |
| `shared_blks_read` | bigint | Disk reads (high = cache miss) |
| `temp_blks_written` | bigint | Temp disk usage (high = `work_mem` too low) |

### Top 10 Slowest Queries (by total time)

```sql
SELECT
    LEFT(query, 120)            AS query,
    calls,
    ROUND(total_exec_time::numeric, 2)  AS total_ms,
    ROUND(mean_exec_time::numeric, 2)   AS mean_ms,
    ROUND(stddev_exec_time::numeric, 2) AS stddev_ms,
    rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

### Top 10 Most Called Queries

```sql
SELECT
    LEFT(query, 120)                    AS query,
    calls,
    ROUND(mean_exec_time::numeric, 2)   AS mean_ms,
    ROUND(total_exec_time::numeric, 2)  AS total_ms
FROM pg_stat_statements
ORDER BY calls DESC
LIMIT 10;
```

### High Cache Miss Queries

```sql
SELECT
    LEFT(query, 120)                              AS query,
    shared_blks_read,
    shared_blks_hit,
    ROUND(100.0 * shared_blks_read /
        NULLIF(shared_blks_read + shared_blks_hit, 0), 1) AS cache_miss_pct
FROM pg_stat_statements
WHERE shared_blks_read + shared_blks_hit > 1000
ORDER BY cache_miss_pct DESC
LIMIT 10;
```

### Reset Stats

```sql
-- Reset all statistics (use in dev/staging, not production casually)
SELECT pg_stat_statements_reset();

-- Reset for a specific query by queryid
SELECT pg_stat_statements_reset(0, 0, queryid)
FROM pg_stat_statements
WHERE query LIKE '%your_table%';
```

### Go Monitoring Function

```go
// internal/repository/pg_stats.go
package repository

import (
    "context"
    "fmt"

    "github.com/jmoiron/sqlx"
)

type SlowQuery struct {
    Query       string  `db:"query"`
    Calls       int64   `db:"calls"`
    TotalMS     float64 `db:"total_ms"`
    MeanMS      float64 `db:"mean_ms"`
    StddevMS    float64 `db:"stddev_ms"`
}

// TopSlowQueries returns the N slowest queries by total execution time.
func TopSlowQueries(ctx context.Context, db *sqlx.DB, n int) ([]SlowQuery, error) {
    const q = `
        SELECT
            LEFT(query, 200) AS query,
            calls,
            ROUND(total_exec_time::numeric, 2)  AS total_ms,
            ROUND(mean_exec_time::numeric, 2)   AS mean_ms,
            ROUND(stddev_exec_time::numeric, 2) AS stddev_ms
        FROM pg_stat_statements
        WHERE calls > 10
        ORDER BY total_exec_time DESC
        LIMIT $1
    `
    var rows []SlowQuery
    if err := db.SelectContext(ctx, &rows, q, n); err != nil {
        return nil, fmt.Errorf("top slow queries: %w", err)
    }
    return rows, nil
}
```

---

## pgcrypto — Encryption and Hashing

### Setup

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;
```

Contrib module — available everywhere.

### gen_random_uuid()

Built-in since PostgreSQL 13 (no pgcrypto needed for PG 13+):

```sql
SELECT gen_random_uuid();  -- 'a3f4c7d2-...'
```

### Password Hashing: crypt() and gen_salt()

pgcrypto can hash passwords in the database, but **prefer Go-side bcrypt** — keeps secrets out of query logs.

```sql
-- Hash a password (DATABASE SIDE — avoid in production)
SELECT crypt('user_password', gen_salt('bf', 12));

-- Verify
SELECT (stored_hash = crypt('user_password', stored_hash)) FROM users WHERE email = 'user@example.com';
```

**Recommendation:** Use Go `golang.org/x/crypto/bcrypt` instead. Database-side hashing exposes plaintext passwords in logs and `pg_stat_statements`.

### PGP Symmetric Encryption — Column Encryption

For encrypting PII at the column level while keeping data queryable:

```sql
-- Encrypt on insert (key from environment — never hardcode)
INSERT INTO patients (id, name, ssn_encrypted)
VALUES (
    gen_random_uuid(),
    'John Doe',
    pgp_sym_encrypt('123-45-6789', current_setting('app.encryption_key'))
);

-- Decrypt on read
SELECT
    name,
    pgp_sym_decrypt(ssn_encrypted, current_setting('app.encryption_key')) AS ssn
FROM patients
WHERE id = $1;
```

Set the key as a database parameter (not hardcoded in SQL):

```sql
-- Set per connection in application code before queries
SET app.encryption_key = 'your-secret-key-from-env';
```

### Raw AES Encryption

```sql
-- Encrypt bytes using AES-128-CBC
SELECT encrypt('sensitive data'::bytea, 'aes-key-16bytes!'::bytea, 'aes');

-- Decrypt
SELECT convert_from(
    decrypt(encrypted_col, 'aes-key-16bytes!'::bytea, 'aes'),
    'UTF8'
) FROM secrets WHERE id = $1;
```

### When to Encrypt in Go vs PostgreSQL

| Factor | Encrypt in Go | Encrypt in PostgreSQL |
|---|---|---|
| **Key management** | Easier — key never enters DB | Key passed via SQL (risk of query log exposure) |
| **Performance** | App scales horizontally | DB CPU used; serialized on single node |
| **Queryability** | Encrypted in transit, search requires decryption | `pgp_sym_encrypt` columns are searchable by exact encrypted value only |
| **Audit trail** | Decryption logged in app | Decryption logged in pg_stat_statements |
| **Use when** | PII, secrets, bulk data | Legacy schemas, multi-app access to same DB |

**Recommendation:** Encrypt sensitive fields in Go before storing. Use `pgp_sym_encrypt` only when multiple services share the same database and you want encryption enforcement at the storage layer.

### Example: PII Column Encryption in Go

```go
// internal/service/patient_service.go
package service

import (
    "context"
    "crypto/aes"
    "crypto/cipher"
    "crypto/rand"
    "encoding/base64"
    "fmt"
    "io"
)

// EncryptField encrypts a plaintext string using AES-GCM.
// key must be 16, 24, or 32 bytes.
func EncryptField(plaintext string, key []byte) (string, error) {
    block, err := aes.NewCipher(key)
    if err != nil {
        return "", fmt.Errorf("new cipher: %w", err)
    }

    gcm, err := cipher.NewGCM(block)
    if err != nil {
        return "", fmt.Errorf("new gcm: %w", err)
    }

    nonce := make([]byte, gcm.NonceSize())
    if _, err := io.ReadFull(rand.Reader, nonce); err != nil {
        return "", fmt.Errorf("generate nonce: %w", err)
    }

    ciphertext := gcm.Seal(nonce, nonce, []byte(plaintext), nil)
    return base64.StdEncoding.EncodeToString(ciphertext), nil
}

// DecryptField decrypts a base64-encoded AES-GCM ciphertext.
func DecryptField(encoded string, key []byte) (string, error) {
    data, err := base64.StdEncoding.DecodeString(encoded)
    if err != nil {
        return "", fmt.Errorf("decode base64: %w", err)
    }

    block, err := aes.NewCipher(key)
    if err != nil {
        return "", fmt.Errorf("new cipher: %w", err)
    }

    gcm, err := cipher.NewGCM(block)
    if err != nil {
        return "", fmt.Errorf("new gcm: %w", err)
    }

    nonceSize := gcm.NonceSize()
    if len(data) < nonceSize {
        return "", fmt.Errorf("ciphertext too short")
    }

    plaintext, err := gcm.Open(nil, data[:nonceSize], data[nonceSize:], nil)
    if err != nil {
        return "", fmt.Errorf("decrypt: %w", err)
    }
    return string(plaintext), nil
}
```

---

## uuid-ossp — UUID Generation

### Setup

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
```

### uuid_generate_v4() vs gen_random_uuid()

| Function | Source | Available |
|---|---|---|
| `uuid_generate_v4()` | uuid-ossp extension | All PostgreSQL versions |
| `gen_random_uuid()` | Built-in | PostgreSQL 13+ |

**Recommendation:** Use `gen_random_uuid()` for PostgreSQL 13+. No extension required.

```sql
-- PG 13+ — no extension needed
SELECT gen_random_uuid();

-- Older PostgreSQL — requires uuid-ossp
SELECT uuid_generate_v4();
```

### Other uuid-ossp Functions

```sql
-- UUID v1: time-based (includes MAC address — privacy concern)
SELECT uuid_generate_v1();

-- UUID v3: name-based, MD5 hash (deterministic)
SELECT uuid_generate_v3(uuid_ns_dns(), 'example.com');

-- UUID v5: name-based, SHA-1 hash (deterministic, preferred over v3)
SELECT uuid_generate_v5(uuid_ns_url(), 'https://example.com/users/42');
```

UUID v5 is useful for generating deterministic IDs from known inputs (e.g., idempotency keys, external ID mapping).

### UUID as Primary Key — Performance Considerations

UUIDv4 primary keys cause **index fragmentation** because values are random — new rows insert into arbitrary leaf pages of the B-tree index, causing frequent page splits.

| Key Type | Insert Pattern | Index Fragmentation | Notes |
|---|---|---|---|
| `BIGINT IDENTITY` | Sequential | Minimal | Best write performance |
| `UUIDv4` | Random | High | Good for distribution; poor for sequential inserts |
| `UUIDv7` | Time-ordered random | Minimal | Best of both worlds — sequential + globally unique |
| `ULID` | Time-ordered | Minimal | Lexicographically sortable, stored as UUID |

**For high-insert tables, prefer `BIGINT GENERATED ALWAYS AS IDENTITY` or UUIDv7** over random UUIDv4. If you need UUID-shaped IDs externally but sequential writes internally, generate UUIDv7 in Go:

```go
// Use github.com/google/uuid v1.6+ for UUIDv7
import "github.com/google/uuid"

id, err := uuid.NewV7()
// id is time-ordered, globally unique, stores as standard UUID column
```

---

## pgAudit — Audit Logging

### Setup

Requires `shared_preload_libraries` (server restart):

```
# postgresql.conf
shared_preload_libraries = 'pgaudit'
pgaudit.log = 'write, ddl'      # see log classes below
pgaudit.log_catalog = off       # reduce noise from system catalog queries
pgaudit.log_relation = on       # log table name in each statement
pgaudit.log_parameter = off     # set 'on' only if needed — logs query params
```

Then per-database:

```sql
CREATE EXTENSION IF NOT EXISTS pgaudit;
```

### Log Classes

| Class | Logs |
|---|---|
| `READ` | `SELECT`, `COPY FROM` |
| `WRITE` | `INSERT`, `UPDATE`, `DELETE`, `TRUNCATE`, `COPY TO` |
| `FUNCTION` | Function and procedure calls |
| `ROLE` | `GRANT`, `REVOKE`, `CREATE/ALTER/DROP ROLE` |
| `DDL` | All DDL except `ROLE` class |
| `MISC` | `CHECKPOINT`, `VACUUM`, `SET` |
| `ALL` | Everything |

**Recommended minimum for production:** `pgaudit.log = 'write, ddl, role'`

### Per-Role Auditing

```sql
-- Audit only specific roles (reduces log volume vs global setting)
ALTER ROLE admin_user SET pgaudit.log = 'all';
ALTER ROLE api_user SET pgaudit.log = 'write';
ALTER ROLE readonly_user SET pgaudit.log = 'read';
```

### Log Format

pgAudit writes to the PostgreSQL log in this format:

```
AUDIT: SESSION,1,1,WRITE,INSERT,TABLE,public.users,"INSERT INTO users ...",<none>
       ^         ^ ^     ^      ^     ^     ^         ^                     ^
       type   stmt sub  class  cmd   type  name      query                params
```

Fields: `AUDIT_TYPE, STATEMENT_ID, SUBSTATEMENT_ID, CLASS, COMMAND, OBJECT_TYPE, OBJECT_NAME, STATEMENT, PARAMETER`

### Integration with Log Aggregators

pgAudit logs go to the PostgreSQL server log (stdout in Docker, or a log file). Forward to your aggregator:

```yaml
# docker-compose.yml — stream PostgreSQL logs to stdout for collection
services:
  postgres:
    image: postgres:16
    command: >
      postgres
        -c shared_preload_libraries=pgaudit
        -c pgaudit.log=write,ddl
        -c log_destination=stderr
        -c logging_collector=off
```

For structured logging, combine with `log_line_prefix` and a log shipper (Fluentd, Vector, Datadog Agent).

### pgAudit vs Application-Level Audit Table

| Factor | pgAudit | Application-Level Audit Table |
|---|---|---|
| **Coverage** | All SQL, including direct DB access | Only operations through your app |
| **Performance** | Low overhead (async log write) | Adds write to audit table on every change |
| **Queryability** | Log files or external tool (Splunk, etc.) | SQL queryable directly |
| **Bypass risk** | Cannot be bypassed by app code | Can be bypassed (skip audit call) |
| **Compliance** | SOC 2, PCI-DSS require DB-level audit | Insufficient alone for compliance |
| **Use when** | Regulatory compliance, security monitoring | Product-level audit (user-visible history) |

**Pattern:** Use both. pgAudit for compliance; application audit table for product features ("who changed this record?").

**Simple application-level audit table:**

```sql
CREATE TABLE audit_log (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    table_name  TEXT        NOT NULL,
    record_id   TEXT        NOT NULL,
    operation   TEXT        NOT NULL,  -- INSERT, UPDATE, DELETE
    actor_id    UUID,                  -- authenticated user, nullable for system ops
    old_data    JSONB,
    new_data    JSONB,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_log_record ON audit_log (table_name, record_id);
CREATE INDEX idx_audit_log_actor  ON audit_log (actor_id) WHERE actor_id IS NOT NULL;
CREATE INDEX idx_audit_log_time   ON audit_log USING brin (created_at);
```

---

## Extension Compatibility Matrix

| Extension | Min PG Version | AWS RDS | Cloud SQL | Supabase | Needs `shared_preload_libraries` |
|---|---|---|---|---|---|
| **pgcrypto** | 8.3 | Yes | Yes | Yes | No |
| **uuid-ossp** | 8.3 | Yes | Yes | Yes | No |
| **pg_trgm** | 8.3 | Yes | Yes | Yes | No |
| **pg_stat_statements** | 9.4 | Yes (pre-enabled) | Yes | Yes (pre-enabled) | **Yes** |
| **pg_cron** | 9.5 | Yes (v10.4+) | No | No | **Yes** |
| **pg_partman** | 10 | No (self-managed only) | No | No | No (for native) |
| **pgAudit** | 9.5 | Yes | Yes | No | **Yes** |
| **pgvector** | 11 | Yes (v15.2+) | Yes | Yes | No |
| **PostGIS** | 9.4 | Yes | Yes | Yes | No |
| **TimescaleDB** | 12 | No (self-managed) | No | No | **Yes** |
| **ParadeDB (pg_search)** | 14 | No | No | No | No |

**Notes:**

- `shared_preload_libraries` extensions require a **server restart** to enable. On managed DBs, this means a maintenance window or rebooting the instance.
- **pg_partman** on managed DBs: the extension itself can be installed, but `run_maintenance_proc()` requires a superuser or specific privileges. Verify with your provider.
- **pg_cron on Cloud SQL / Supabase**: not supported as of 2025. Use external schedulers (Cloud Scheduler, cron jobs in your app, or Kubernetes CronJob) instead.
- **Supabase** pre-installs many extensions — check the dashboard under Database > Extensions before adding to migrations.
- **RDS pg_cron**: available on RDS PostgreSQL 12.5+ and Aurora PostgreSQL 12.6+. The `cron.database_name` must match your RDS database name.

### Extensions Requiring Restart vs Hot-Enable

| Can enable with `CREATE EXTENSION` only | Requires `shared_preload_libraries` + restart first |
|---|---|
| pgcrypto, uuid-ossp, pg_trgm, pgvector, PostGIS, pg_partman, ParadeDB | pg_cron, pg_stat_statements, pgAudit, TimescaleDB |
