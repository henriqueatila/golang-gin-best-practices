# Messaging — Consumer Pattern and Work Queues

How to run background consumers in Go Gin APIs using RabbitMQ.

Companion to `messaging-consumer-docker.md` (Docker Compose setup for RabbitMQ).

---

## Consumer Pattern

**Gate:** Run consumers as goroutines in a dedicated worker binary or in `main.go`. Never block the HTTP server. **Cost: MEDIUM** — each goroutine holds a channel; size your pool accordingly.

```go
// internal/worker/email_consumer.go
package worker

import (
	"context"
	"encoding/json"
	"fmt"
	"log/slog"

	amqp "github.com/rabbitmq/amqp091-go"
	"yourmodule/pkg/messaging"
)

type EmailConsumer struct {
	conn   *messaging.Connection
	mailer Mailer
	logger *slog.Logger
}

type EmailPayload struct {
	UserID string `json:"user_id"`
	Email  string `json:"email"`
	Type   string `json:"type"`
}

func (c *EmailConsumer) Run(ctx context.Context) error {
	ch, err := c.conn.Channel()
	if err != nil {
		return fmt.Errorf("EmailConsumer.Run channel: %w", err)
	}
	defer ch.Close()

	if err := ch.Qos(1, 0, false); err != nil { // one unacked message at a time
		return fmt.Errorf("EmailConsumer.Run qos: %w", err)
	}

	q, err := messaging.DeclareQueue(ch, messaging.QueueOpts{
		Name:    "email.send",
		Durable: true,
		DLX:     "dlx.email",
	})
	if err != nil {
		return fmt.Errorf("EmailConsumer.Run declare: %w", err)
	}

	deliveries, err := ch.Consume(q.Name, "", false, false, false, false, nil)
	if err != nil {
		return fmt.Errorf("EmailConsumer.Run consume: %w", err)
	}

	c.logger.Info("email consumer started", "queue", q.Name)
	for {
		select {
		case <-ctx.Done():
			c.logger.Info("email consumer shutting down")
			return nil
		case d, ok := <-deliveries:
			if !ok {
				return fmt.Errorf("EmailConsumer: delivery channel closed")
			}
			c.handle(ctx, d)
		}
	}
}

func (c *EmailConsumer) handle(ctx context.Context, d amqp.Delivery) {
	var payload EmailPayload
	if err := json.Unmarshal(d.Body, &payload); err != nil {
		c.logger.ErrorContext(ctx, "bad payload, dead-lettering",
			"message_id", d.MessageId, "error", err)
		_ = d.Nack(false, false) // permanent failure → DLQ
		return
	}
	if err := c.mailer.Send(ctx, payload.Email, payload.Type); err != nil {
		c.logger.WarnContext(ctx, "transient error, requeueing",
			"message_id", d.MessageId, "error", err)
		_ = d.Nack(false, true) // requeue for retry
		return
	}
	c.logger.InfoContext(ctx, "email sent", "message_id", d.MessageId, "to", payload.Email)
	_ = d.Ack(false)
}
```

---

## Work Queues — Competing Consumers

**Gate:** Use when a single consumer cannot keep up. **Cost: LOW** — same queue, multiple consumers; RabbitMQ round-robins by default.

```go
// cmd/worker/main.go
package main

import (
	"context"
	"log/slog"
	"os"
	"os/signal"
	"syscall"

	"yourmodule/internal/worker"
	"yourmodule/pkg/messaging"
)

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

	conn, err := messaging.NewConnection(messaging.Config{
		URL: os.Getenv("RABBITMQ_URL"),
	}, logger)
	if err != nil {
		logger.Error("failed to connect to rabbitmq", "error", err)
		os.Exit(1)
	}
	defer conn.Close()

	ctx, cancel := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
	defer cancel()

	// Run N=3 competing consumers on the same queue.
	for range 3 {
		consumer := worker.NewEmailConsumer(conn, logger)
		go func() {
			if err := consumer.Run(ctx); err != nil {
				logger.Error("email consumer error", "error", err)
			}
		}()
	}

	<-ctx.Done()
	logger.Info("worker shutting down")
}
```

**Key Rule:** Set `ch.Qos(1, 0, false)` so each consumer only prefetches one message. Without this, a busy consumer gets starved while another is overloaded.
