# Messaging — Queue Declaration Helper and Producer Pattern

Companion to `messaging-rabbitmq-connection.md` (connection factory, when to use messaging).

---

## Queue Declaration Helper

```go
// pkg/messaging/topology.go
type QueueOpts struct {
    Name       string
    Durable    bool
    AutoDelete bool
    DLX        string // dead letter exchange name (empty = no DLX)
}

func DeclareQueue(ch *amqp.Channel, opts QueueOpts) (amqp.Queue, error) {
    args := amqp.Table{}
    if opts.DLX != "" {
        args["x-dead-letter-exchange"] = opts.DLX
    }
    q, err := ch.QueueDeclare(opts.Name, opts.Durable, opts.AutoDelete, false, false, args)
    if err != nil {
        return amqp.Queue{}, fmt.Errorf("DeclareQueue %q: %w", opts.Name, err)
    }
    return q, nil
}
```

Always declare queues/exchanges idempotently (same arguments on every startup). RabbitMQ errors on redeclare with different settings.

---

## Producer Pattern

Publish from the service layer, never directly from Gin handlers.

```go
// pkg/messaging/producer.go
type Producer struct {
    conn   *Connection
    logger *slog.Logger
}

func NewProducer(conn *Connection, logger *slog.Logger) *Producer {
    return &Producer{conn: conn, logger: logger}
}

func (p *Producer) Publish(ctx context.Context, exchange, routingKey string, payload any) error {
    body, err := json.Marshal(payload)
    if err != nil {
        return fmt.Errorf("producer.Publish marshal: %w", err)
    }
    ch, err := p.conn.Channel()
    if err != nil {
        return fmt.Errorf("producer.Publish channel: %w", err)
    }
    defer ch.Close()

    msgID := uuid.NewString()
    msg := amqp.Publishing{
        ContentType:  "application/json",
        DeliveryMode: amqp.Persistent,
        MessageId:    msgID,
        Timestamp:    time.Now().UTC(),
        Body:         body,
    }
    if err := ch.PublishWithContext(ctx, exchange, routingKey, false, false, msg); err != nil {
        return fmt.Errorf("producer.Publish: %w", err)
    }
    p.logger.InfoContext(ctx, "message published",
        "exchange", exchange, "routing_key", routingKey, "message_id", msgID)
    return nil
}
```

Calling from a service:

```go
func (s *Service) SendWelcomeEmail(ctx context.Context, userID, email string) error {
    payload := map[string]string{"user_id": userID, "email": email, "type": "welcome"}
    return s.producer.Publish(ctx, "", "email.send", payload)
}
```
