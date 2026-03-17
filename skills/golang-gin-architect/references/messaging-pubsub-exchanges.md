# Messaging — Pub/Sub with Exchanges

Fanout and topic exchange patterns for fan-out event delivery in Go Gin APIs.

---

## Pub/Sub with Exchanges

**Gate:** Use exchanges when one event must fan out to multiple independent consumers (e.g., order.placed triggers inventory, email, and analytics).

**Cost: MEDIUM** — topology complexity grows; document your exchange/binding map.

### Fanout (Broadcast to All Queues)

```go
// pkg/messaging/pubsub.go
package messaging

import (
	"fmt"
	amqp "github.com/rabbitmq/amqp091-go"
)

// DeclareFanout declares a fanout exchange and binds a queue to it.
func DeclareFanout(ch *amqp.Channel, exchange, queueName string) error {
	if err := ch.ExchangeDeclare(exchange, "fanout", true, false, false, false, nil); err != nil {
		return fmt.Errorf("DeclareFanout exchange %q: %w", exchange, err)
	}
	q, err := DeclareQueue(ch, QueueOpts{Name: queueName, Durable: true})
	if err != nil {
		return err
	}
	if err := ch.QueueBind(q.Name, "", exchange, false, nil); err != nil {
		return fmt.Errorf("DeclareFanout bind %q→%q: %w", queueName, exchange, err)
	}
	return nil
}
```

### Topic Exchange (Pattern Routing)

```go
// Routing key examples: "order.placed", "order.cancelled", "user.created"
// Binding pattern:      "order.*"  matches order.placed and order.cancelled
//                       "#"        matches everything

func DeclareTopicBinding(ch *amqp.Channel, exchange, queueName, pattern string) error {
	if err := ch.ExchangeDeclare(exchange, "topic", true, false, false, false, nil); err != nil {
		return fmt.Errorf("DeclareTopicBinding exchange %q: %w", exchange, err)
	}
	q, err := DeclareQueue(ch, QueueOpts{Name: queueName, Durable: true})
	if err != nil {
		return err
	}
	if err := ch.QueueBind(q.Name, pattern, exchange, false, nil); err != nil {
		return fmt.Errorf("DeclareTopicBinding bind %q pattern=%q: %w", queueName, pattern, err)
	}
	return nil
}
```

### Example: Order Event Fan-out

```
Exchange: events (topic)
  ├── routing: "order.*"  → queue: inventory.orders  → InventoryConsumer
  ├── routing: "order.*"  → queue: email.orders      → EmailConsumer
  └── routing: "order.*"  → queue: analytics.orders  → AnalyticsConsumer
```

**Publisher:**

```go
// Publish order.placed event to topic exchange
err := producer.Publish(ctx, "events", "order.placed", orderPayload)
```

**Consumer binding (each service sets up its own queue):**

```go
// InventoryConsumer setup
if err := messaging.DeclareTopicBinding(ch, "events", "inventory.orders", "order.*"); err != nil {
    return fmt.Errorf("setup binding: %w", err)
}
```

---

## Exchange Type Summary

| Exchange Type | Routing Logic | Use Case |
|---|---|---|
| **direct** | Exact routing key match | Point-to-point, DLQ |
| **fanout** | Broadcast to all bound queues | Notifications to all consumers |
| **topic** | Pattern matching (`*`, `#`) | Selective routing by event type |
| **headers** | Message header attributes | Rarely needed — use topic instead |

**Rules:**
- Always declare exchanges as durable (`true`) — survives broker restart
- Declare topology (exchanges + queues + bindings) at consumer startup, idempotently
- One exchange can have many bindings; one queue can be bound to many exchanges

---

## Cross-Skill References

| Topic | Reference |
|---|---|
| Connection and queue setup | `messaging-rabbitmq-setup.md` |
| Consumer and work queue pattern | `messaging-consumer-workqueues.md` |
| DLQ and idempotency | `messaging-dlq-idempotency.md` |
| Outbox pattern for guaranteed delivery | `data-patterns-outbox.md` |
| Worker deployment | `golang-gin-deploy` skill |
