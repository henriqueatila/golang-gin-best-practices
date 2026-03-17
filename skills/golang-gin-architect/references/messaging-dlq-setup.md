# Messaging — Dead Letter Queues

How to handle message failures safely with DLQ. Companion to `messaging-dlq-idempotency.md` (idempotent consumers).

---

## Gate

Add DLQ when message loss on failure is unacceptable. Without DLQ, rejected messages vanish.

**Cost: MEDIUM** — requires separate DLX, DLQ, and a monitoring consumer.

---

## Declare DLX and DLQ

```go
// pkg/messaging/dlq.go
package messaging

import (
	"fmt"
	amqp "github.com/rabbitmq/amqp091-go"
)

// SetupDLQ declares dead letter exchange + queue and binds them.
// Call this before declaring the primary queue that uses DLX.
func SetupDLQ(ch *amqp.Channel, dlxName, dlqName string) error {
	if err := ch.ExchangeDeclare(dlxName, "direct", true, false, false, false, nil); err != nil {
		return fmt.Errorf("SetupDLQ exchange %q: %w", dlxName, err)
	}
	dlq, err := ch.QueueDeclare(dlqName, true, false, false, false, nil)
	if err != nil {
		return fmt.Errorf("SetupDLQ queue %q: %w", dlqName, err)
	}
	if err := ch.QueueBind(dlq.Name, "", dlxName, false, nil); err != nil {
		return fmt.Errorf("SetupDLQ bind: %w", err)
	}
	return nil
}
```

---

## Primary Queue Using DLX

```go
// In consumer setup:
if err := messaging.SetupDLQ(ch, "dlx.email", "dlq.email"); err != nil {
    return fmt.Errorf("setup dlq: %w", err)
}

q, err := messaging.DeclareQueue(ch, messaging.QueueOpts{
    Name:    "email.send",
    Durable: true,
    DLX:     "dlx.email", // failed messages go here
})
```

---

## DLQ Monitor Consumer

```go
// internal/worker/dlq_monitor.go
package worker

import (
	"context"
	"log/slog"

	amqp "github.com/rabbitmq/amqp091-go"
	"yourmodule/pkg/messaging"
)

type DLQMonitor struct {
	conn    *messaging.Connection
	alerter Alerter // PagerDuty, Slack, etc.
	logger  *slog.Logger
}

func (m *DLQMonitor) Run(ctx context.Context, dlqName string) error {
	ch, _ := m.conn.Channel()
	defer ch.Close()

	q, _ := messaging.DeclareQueue(ch, messaging.QueueOpts{Name: dlqName, Durable: true})
	deliveries, _ := ch.Consume(q.Name, "", false, false, false, false, nil)

	for {
		select {
		case <-ctx.Done():
			return nil
		case d, ok := <-deliveries:
			if !ok {
				return nil
			}
			m.logger.ErrorContext(ctx, "dead letter received",
				"message_id", d.MessageId,
				"routing_key", d.RoutingKey,
				"body", string(d.Body),
			)
			m.alerter.Alert(ctx, "dead letter: "+d.RoutingKey, d.Body)
			_ = d.Ack(false) // ack to remove from DLQ after alerting
		}
	}
}
```
