# Messaging — RabbitMQ Docker Compose Setup

Local development setup for RabbitMQ. Companion to `messaging-consumer-workqueues.md` (consumer pattern, competing consumers).

---

## Docker Compose — RabbitMQ for Dev

**Cost: LOW** — single-node is fine for local dev; never run single-node in production without persistence config.

```yaml
# docker-compose.yml (RabbitMQ section)
services:
  rabbitmq:
    image: rabbitmq:4-management-alpine
    container_name: rabbitmq
    ports:
      - "5672:5672"    # AMQP protocol
      - "15672:15672"  # Management UI (http://localhost:15672)
    environment:
      RABBITMQ_DEFAULT_USER: guest
      RABBITMQ_DEFAULT_PASS: guest
    volumes:
      - rabbitmq_data:/data
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

volumes:
  rabbitmq_data:
```

**Environment variable for app:**

```bash
RABBITMQ_URL=amqp://guest:guest@localhost:5672/
```

**Management UI credentials:** `guest / guest` at `http://localhost:15672`

---

## Useful CLI Tasks Inside Container

```bash
docker exec rabbitmq rabbitmqctl list_queues name messages consumers
docker exec rabbitmq rabbitmqctl list_exchanges
docker exec rabbitmq rabbitmqctl purge_queue email.send  # dev only
```

---

## Cross-Skill References

| Topic | Reference |
|---|---|
| Consumer pattern + competing consumers | `messaging-consumer-workqueues.md` |
| Connection factory + reconnect | `messaging-rabbitmq-connection.md` |
| Dead letter queues | `messaging-dlq-setup.md` |
| Idempotent consumers | `messaging-dlq-idempotency.md` |
