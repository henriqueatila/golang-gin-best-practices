# Replication and HA — Lag Monitoring

## Query on Primary: pg_stat_replication

```sql
-- Shows lag per connected standby
SELECT
    application_name,
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    write_lag,
    flush_lag,
    replay_lag,
    pg_size_pretty(pg_wal_lsn_diff(sent_lsn, replay_lsn)) AS lag_bytes
FROM pg_stat_replication
ORDER BY replay_lag DESC NULLS LAST;
```

`replay_lag` is wall-clock time since the primary last confirmed the standby replayed that WAL segment. Alert when it exceeds 30 seconds.

## Query on Replica

```sql
-- Transaction replay lag (most human-readable)
SELECT now() - pg_last_xact_replay_timestamp() AS replication_lag;

-- LSN-based lag in bytes (run on replica, compare to primary sent_lsn)
SELECT
    pg_last_wal_receive_lsn()  AS received_lsn,
    pg_last_wal_replay_lsn()   AS replayed_lsn,
    pg_wal_lsn_diff(
        pg_last_wal_receive_lsn(),
        pg_last_wal_replay_lsn()
    )                           AS receive_vs_replay_bytes,
    pg_is_in_recovery()        AS is_replica;
```

## Go Lag Monitor

```go
// internal/db/replication_monitor.go
package db

import (
    "context"
    "fmt"
    "log/slog"
    "time"

    "github.com/jmoiron/sqlx"
)

const lagWarningThreshold = 30 * time.Second

// CheckReplicationLag queries pg_stat_replication on the primary and logs
// a warning for any standby that exceeds the threshold.
func CheckReplicationLag(ctx context.Context, primary *sqlx.DB, logger *slog.Logger) error {
    const q = `
        SELECT
            application_name,
            state,
            EXTRACT(EPOCH FROM replay_lag)::bigint  AS replay_lag_seconds,
            pg_wal_lsn_diff(sent_lsn, replay_lsn)  AS lag_bytes
        FROM pg_stat_replication`

    type row struct {
        ApplicationName  string `db:"application_name"`
        State            string `db:"state"`
        ReplayLagSeconds int64  `db:"replay_lag_seconds"`
        LagBytes         int64  `db:"lag_bytes"`
    }

    var rows []row
    if err := primary.SelectContext(ctx, &rows, q); err != nil {
        return fmt.Errorf("pg_stat_replication query: %w", err)
    }

    if len(rows) == 0 {
        logger.WarnContext(ctx, "no standbys connected to primary")
        return nil
    }

    for _, r := range rows {
        lag := time.Duration(r.ReplayLagSeconds) * time.Second
        if lag > lagWarningThreshold {
            logger.WarnContext(ctx, "replication lag exceeds threshold",
                "standby", r.ApplicationName,
                "state", r.State,
                "lag", lag.String(),
                "lag_bytes", r.LagBytes,
            )
        } else {
            logger.InfoContext(ctx, "replication healthy",
                "standby", r.ApplicationName,
                "lag", lag.String(),
            )
        }
    }
    return nil
}
```

When `ReplayLagSeconds` exceeds the threshold, route reads to primary until lag recovers:

```go
// ShouldUsePrimary returns true when replica lag is too high.
func ShouldUsePrimary(ctx context.Context, replica *sqlx.DB) bool {
    var lagSeconds float64
    err := replica.QueryRowContext(ctx,
        `SELECT EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp()))`,
    ).Scan(&lagSeconds)
    if err != nil || lagSeconds > 30 {
        return true
    }
    return false
}
```
