# Complexity Assessment — You Don't Need This Yet (Gates)

For each high-complexity pattern, ALL listed conditions must be true before adoption is justified. If any are false, use the simpler alternative.

---

## Microservices

All 5 must be true:

- [ ] You have 3+ independent teams that need to deploy on different schedules and are currently blocked
- [ ] You have measured (not estimated) that a specific module has an independent scaling bottleneck
- [ ] You have working distributed tracing across all services (not just logging)
- [ ] You have a CI/CD pipeline that can build, test, and deploy each service independently in under 10 minutes
- [ ] You have on-call runbooks for each service and a tested rollback procedure

**If any is false:** Build a modular monolith. Clean internal package boundaries give you 80% of the benefit at 10% of the cost.

---

## CQRS

All 4 must be true:

- [ ] Read volume is measurably 10x+ write volume (check your query logs)
- [ ] The read model requires data that is inconvenient to compute at query time (multiple joins, aggregations)
- [ ] You have profiled PostgreSQL and confirmed it is the bottleneck — not your Go code or network
- [ ] Your team has built CQRS before and understands eventual consistency implications

**If any is false:** Add a read-optimized query method to your repository. A well-indexed PostgreSQL view handles most "CQRS-shaped problems" without the overhead.

---

## Event Sourcing

All 4 must be true:

- [ ] You have a legal or compliance requirement to reproduce system state at any past point in time
- [ ] Your domain has complex state machines where the transition history matters, not just current state
- [ ] Your team has operational experience running an event store (Kafka, EventStoreDB) in production
- [ ] You have designed your aggregates and projections and the team agrees on the boundaries

**If any is false:** Use a PostgreSQL append-only audit log + standard CRUD domain model.

```sql
-- Handles 90% of audit requirements with 10% of event sourcing complexity:
CREATE TABLE audit_log (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    table_name TEXT NOT NULL,
    record_id  TEXT NOT NULL,
    action     TEXT NOT NULL,  -- INSERT, UPDATE, DELETE
    old_values JSONB,
    new_values JSONB,
    actor_id   TEXT,
    created_at TIMESTAMPTZ DEFAULT now()
);
```

---

## Service Mesh (Istio, Linkerd)

All 3 must be true:

- [ ] You have 10+ services in production with active traffic
- [ ] You have a dedicated platform engineer who owns the mesh configuration
- [ ] You have a compliance requirement for mTLS between services that cannot be satisfied by application-level TLS

**If any is false:** Handle TLS at the load balancer, use OpenTelemetry for observability, implement circuit breakers at the application layer.

---

## Distributed Tracing (OpenTelemetry)

You need this when:

- [ ] You have 2+ services communicating with each other
- [ ] You are debugging latency issues that cross service boundaries
- [ ] Structured logging alone cannot explain where time is being spent

**If any is false:** Structured logging with `log/slog` + request IDs is sufficient for a monolith.

---

## Saga / Orchestration

All 4 must be true:

- [ ] A business transaction spans 3+ services or databases
- [ ] Each step can fail independently after already partially committing
- [ ] You need compensating actions that can't be expressed as a single DB rollback
- [ ] Calling services sequentially with manual compensation is insufficient for your retry/durability requirements

**If any is false:** Use a database transaction (single DB) or sequential calls with explicit compensation (2 services).

---

## Message Queue (RabbitMQ, SQS, Kafka)

You need this when:

- [ ] The caller does not need the result immediately
- [ ] The work can fail and be retried later (retry-later semantics)
- [ ] Volume exceeds 1K/min OR durability across process restarts is required

**If any is false:** Use a goroutine + buffered channel for fire-and-forget work.

---

## Final Rule

> Architecture is not a competition. The system that ships, that the team can maintain, and that can be extended by a new hire in their first week — that is the right architecture. Complexity is a debt. Spend it wisely.
