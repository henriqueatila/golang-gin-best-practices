# Replication and HA — Streaming Replication Setup

## Primary: postgresql.conf

```ini
# postgresql.conf on primary
wal_level            = replica        # minimum for streaming; use 'logical' if logical replication also needed
max_wal_senders      = 5              # max concurrent standby connections (include headroom for pg_basebackup)
wal_keep_size        = 512MB          # WAL retained on primary; protects against slow standbys
hot_standby          = on             # allows standbys to serve read-only queries (default on since PG 14)

# For synchronous replication (optional)
# synchronous_standby_names = 'standby1'
# synchronous_commit = on
```

Reload with `SELECT pg_reload_conf();` — no restart needed for most of these except `wal_level` and `max_wal_senders`.

## Replication User

```sql
-- Run on primary
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'strong_password_here';
```

In `pg_hba.conf` on primary, allow the standby to connect:

```
# pg_hba.conf — allow standby IP to connect for replication
host  replication  replicator  10.0.0.11/32  scram-sha-256
```

## Standby: Initial Base Backup

```bash
# Run on the standby host — copies primary data directory
pg_basebackup \
  --host=10.0.0.10 \
  --username=replicator \
  --pgdata=/var/lib/postgresql/data \
  --wal-method=stream \
  --checkpoint=fast \
  --progress \
  --verbose
```

## Standby: postgresql.conf + standby.signal (PG 12+)

```ini
# postgresql.conf on standby
primary_conninfo = 'host=10.0.0.10 port=5432 user=replicator password=strong_password_here application_name=standby1'
hot_standby      = on
```

```bash
# Create empty file to signal standby mode (replaces recovery.conf from PG 11 and earlier)
touch /var/lib/postgresql/data/standby.signal
```

Start the standby. Monitor replication on the primary:

```sql
SELECT application_name, state, sync_state, replay_lag
FROM pg_stat_replication;
```

## Docker Compose: Primary + Replica

```yaml
# docker-compose.yml
version: "3.9"

services:
  primary:
    image: postgres:16
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: mydb
    volumes:
      - primary_data:/var/lib/postgresql/data
      - ./pg-init/primary.sh:/docker-entrypoint-initdb.d/01-replication.sh
    ports:
      - "5432:5432"

  replica:
    image: postgres:16
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      PGPASSWORD: strong_password_here
    volumes:
      - replica_data:/var/lib/postgresql/data
      - ./pg-init/replica-entrypoint.sh:/entrypoint.sh
    entrypoint: ["/entrypoint.sh"]
    depends_on:
      - primary
    ports:
      - "5433:5432"

volumes:
  primary_data:
  replica_data:
```

```bash
# pg-init/primary.sh — runs inside primary container on first start
psql -U app -c "CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'strong_password_here';"
echo "host replication replicator all scram-sha-256" >> /var/lib/postgresql/data/pg_hba.conf
psql -U app -c "SELECT pg_reload_conf();"
```

```bash
# pg-init/replica-entrypoint.sh
#!/bin/bash
set -e

# Wait for primary to be ready
until pg_isready -h primary -p 5432 -U app; do
  echo "waiting for primary..." && sleep 2
done

# Base backup if data directory is empty
if [ ! -f /var/lib/postgresql/data/PG_VERSION ]; then
  pg_basebackup -h primary -U replicator -D /var/lib/postgresql/data \
    --wal-method=stream --checkpoint=fast
  cat >> /var/lib/postgresql/data/postgresql.conf <<EOF
primary_conninfo = 'host=primary port=5432 user=replicator password=strong_password_here'
hot_standby = on
EOF
  touch /var/lib/postgresql/data/standby.signal
fi

exec docker-entrypoint.sh postgres
```
