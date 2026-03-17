# Messaging — Idempotent Consumers

How to prevent duplicate message processing. Companion to `messaging-dlq-setup.md` (dead letter queues).

---

## Gate

Add idempotency when duplicate message delivery would cause real harm (double charge, duplicate record). RabbitMQ guarantees at-least-once delivery on requeue.

**Cost: MEDIUM** — requires Redis or PostgreSQL for dedup state.

---

## Redis-Based Dedup

```go
// internal/worker/idempotency.go
package worker

import (
	"context"
	"fmt"
	"time"

	"github.com/redis/go-redis/v9"
)

const dedupTTL = 24 * time.Hour

type Deduplicator struct {
	rdb *redis.Client
}

// IsDuplicate returns true if messageID was already processed.
// Marks messageID as processed atomically on first call.
func (d *Deduplicator) IsDuplicate(ctx context.Context, messageID string) (bool, error) {
	key := "msg:processed:" + messageID
	// SET NX: only set if not exists. Returns true if set (new), false if existed.
	ok, err := d.rdb.SetNX(ctx, key, "1", dedupTTL).Result()
	if err != nil {
		return false, fmt.Errorf("dedup check %q: %w", messageID, err)
	}
	return !ok, nil // !ok means key already existed → duplicate
}
```

---

## PostgreSQL-Based Dedup

```sql
-- migrations: create processed_messages table
CREATE TABLE processed_messages (
    message_id TEXT PRIMARY KEY,
    processed_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

```go
func (d *PgDeduplicator) IsDuplicate(ctx context.Context, db *sql.DB, messageID string) (bool, error) {
	result, err := db.ExecContext(ctx,
		`INSERT INTO processed_messages (message_id) VALUES ($1) ON CONFLICT DO NOTHING`,
		messageID,
	)
	if err != nil {
		return false, fmt.Errorf("pg dedup %q: %w", messageID, err)
	}
	n, _ := result.RowsAffected()
	return n == 0, nil // 0 rows affected = already existed → duplicate
}
```

---

## Usage in Consumer

```go
func (c *EmailConsumer) handle(ctx context.Context, d amqp.Delivery) {
	dup, err := c.dedup.IsDuplicate(ctx, d.MessageId)
	if err != nil {
		c.logger.WarnContext(ctx, "dedup check failed, processing anyway", "error", err)
	}
	if dup {
		c.logger.InfoContext(ctx, "duplicate message skipped", "message_id", d.MessageId)
		_ = d.Ack(false)
		return
	}
	// ... proceed with normal processing
}
```

---

## Quick Reference

| Pattern | Use When | Client Call | Cost |
|---|---|---|---|
| Work queue | Task distribution across N workers | `ch.Consume` + `Qos(1)` | LOW |
| Fanout exchange | One event, many consumers | `ExchangeDeclare("fanout")` | MEDIUM |
| Topic exchange | Selective routing by key pattern | `ExchangeDeclare("topic")` | MEDIUM |
| DLQ | Message failure must not cause data loss | `x-dead-letter-exchange` arg | MEDIUM |
| Idempotent consumer | At-least-once delivery + side effects | Redis `SETNX` or PG `ON CONFLICT` | MEDIUM |
| Transactional outbox | Exactly-once across DB + broker | DB write + poller publishes | HIGH |

## Cross-Skill References

| Topic | Reference |
|---|---|
| Database transactions for outbox pattern | `data-patterns-outbox.md` |
| Redis client setup for dedup | `redis-caching-patterns.md` |
| Dead letter queue setup | `messaging-dlq-setup.md` |
