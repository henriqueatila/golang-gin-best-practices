# Messaging — When to Use RabbitMQ and Connection Factory

When to use async messaging in Go Gin APIs and how to establish the RabbitMQ connection.

**Package:** `github.com/rabbitmq/amqp091-go`

---

## When to Use Messaging

**Gate:** Add async messaging only when synchronous HTTP creates a user-facing bottleneck or reliability risk.

```
START: Does the caller need the result immediately?
  ├── Yes → Synchronous HTTP. Done.
  └── No → Can the work fail and be retried later?
      ├── No → Synchronous with timeout. Done.
      └── Yes → Message queue.
          └── Need exactly-once delivery?
              ├── No → Producer + consumer with manual ack.
              └── Yes → Transactional outbox + idempotent consumer.
```

**Good candidates for async:** Email/SMS, image processing, PDF generation, webhook delivery, audit log writes, cache invalidation, batch data exports.

**Bad candidates (keep sync):** Auth checks, payment confirmation (user waits), read queries, anything requiring immediate response data.

---

## RabbitMQ Connection Factory

One connection per process. One channel per goroutine. Never share channels across goroutines.

```go
// pkg/messaging/connection.go
type Config struct {
    URL            string
    ReconnectDelay time.Duration
    MaxRetries     int
}

type Connection struct {
    cfg    Config
    conn   *amqp.Connection
    logger *slog.Logger
}

func NewConnection(cfg Config, logger *slog.Logger) (*Connection, error) {
    if cfg.ReconnectDelay == 0 {
        cfg.ReconnectDelay = 5 * time.Second
    }
    if cfg.MaxRetries == 0 {
        cfg.MaxRetries = 10
    }
    c := &Connection{cfg: cfg, logger: logger}
    if err := c.dial(); err != nil {
        return nil, fmt.Errorf("messaging.NewConnection: %w", err)
    }
    return c, nil
}

func (c *Connection) dial() error {
    var err error
    for i := range c.cfg.MaxRetries {
        c.conn, err = amqp.Dial(c.cfg.URL)
        if err == nil {
            c.logger.Info("rabbitmq connected", "attempt", i+1)
            return nil
        }
        c.logger.Warn("rabbitmq dial failed, retrying",
            "attempt", i+1, "max", c.cfg.MaxRetries,
            "error", err, "delay", c.cfg.ReconnectDelay,
        )
        time.Sleep(c.cfg.ReconnectDelay)
    }
    return fmt.Errorf("rabbitmq dial: exhausted %d retries: %w", c.cfg.MaxRetries, err)
}

func (c *Connection) Channel() (*amqp.Channel, error) {
    ch, err := c.conn.Channel()
    if err != nil {
        return nil, fmt.Errorf("messaging.Channel: %w", err)
    }
    return ch, nil
}

func (c *Connection) Reconnect(ctx context.Context) error {
    notifyClose := c.conn.NotifyClose(make(chan *amqp.Error, 1))
    select {
    case <-ctx.Done():
        return ctx.Err()
    case err := <-notifyClose:
        c.logger.Warn("rabbitmq connection closed, reconnecting", "error", err)
        return c.dial()
    }
}

func (c *Connection) Close() error { return c.conn.Close() }
```

---

## See Also

- `messaging-rabbitmq-producer.md` — queue declaration helper and producer pattern
- `messaging-consumer-workqueues.md` — consumer with manual ack, work queues, Docker Compose
- `messaging-dlq-idempotency.md` — dead letter queues, deduplication
