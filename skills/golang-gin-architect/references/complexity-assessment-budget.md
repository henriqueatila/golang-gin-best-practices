# Complexity Assessment — Budget Framework and Decision Trees

The complexity budget framework and full decision trees for Go Gin API architecture choices.

---

## Complexity Budget Framework

Every architecture decision has a cost: cognitive load, infrastructure to maintain, onboarding friction, and surface area for bugs. That cost is paid continuously.

### Scoring Scale

| Score | Meaning | Examples |
|---|---|---|
| 1 | Trivially simple | Flat package layout, basic CRUD handlers |
| 2 | Low overhead | Repository pattern, service layer, Redis caching |
| 3 | Moderate overhead | API Gateway, feature flags, modular monolith |
| 4 | High overhead | CQRS, Saga/choreography, event-driven async |
| 5 | Expert territory | Event sourcing, service mesh, distributed tracing across 10+ services |

### Budget by Team and Stage

| Team Size | Product Stage | Max Score | Guidance |
|---|---|---|---|
| 1–3 devs | MVP / early | 10–12 | Flat layout + service + repository. Done. |
| 1–3 devs | Growth | 12–15 | Add feature modules and Redis if justified |
| 3–8 devs | MVP | 12–15 | Modular monolith. No microservices. |
| 3–8 devs | Growth | 15–20 | Extract 1–2 services max, only at proven pain points |
| 8–15 devs | Growth | 20–25 | Bounded modules, limited service extraction |
| 8–15 devs | Mature | 25–30 | Multiple services viable if infra maturity is there |
| 15+ devs | Mature | 30+ | Full distributed systems justified — but still measure ROI |

### Budget Example: 3-Dev Startup

Over-engineered proposal:

| Choice | Cost |
|---|---|
| Modular monolith | 3 |
| CQRS for orders domain | 4 |
| Event sourcing for audit trail | 5 |
| Kafka for async notifications | 4 |
| Service mesh (Istio) | 5 |
| **Total** | **21** |

Budget for a 3-dev MVP is 10–12. What they should build:

| Choice | Cost |
|---|---|
| Flat handler/service/repository | 1 |
| PostgreSQL (transactions handle audit) | 1 |
| Goroutine + channel for email notifications | 1 |
| **Total** | **3** |

---

## Decision Trees — Full Detail

### Monolith vs Microservices

```
START: Are you starting a new project?
  └── Yes → MONOLITH. Come back when you feel real pain.

START: Existing system. Do you have measurable deploy coupling pain?
  ├── No → MODULAR MONOLITH. Clean package boundaries, shared DB, single deploy.
  └── Yes → Do you have 3+ teams that need to deploy independently?
      ├── No → MODULAR MONOLITH. Reorganize package ownership.
      └── Yes → Does the module have a clearly different scaling profile?
          ├── No → Still MODULAR MONOLITH.
          └── Yes → Do you have infra maturity to run a service independently?
              ├── No → BUILD INFRA MATURITY FIRST. Extract after.
              └── Yes → Extract THAT ONE MODULE. Re-evaluate in 6 months.
```

**Anti-patterns:** Resume-Driven Development, Netflix Envy, preemptive extraction, micro-monolith (services sharing a DB).

**The modular monolith sweet spot (90% of projects):**

```go
// internal/order/   ← team A owns this
// internal/inventory/ ← team B owns this
// Cross-module calls via interface. Same binary. One deploy. Zero distributed complexity.
```

### Sync vs Async

```
START: Does the caller need the result to proceed?
  └── Yes → SYNCHRONOUS HTTP. Done.

START: Caller doesn't need the result immediately.
  └── Is the operation fast (< 200ms, no external I/O)?
      ├── Yes → Still SYNCHRONOUS.
      └── No → Is failure acceptable (retry-later semantics OK)?
          ├── No → SYNCHRONOUS with timeout + retry + circuit breaker.
          └── Yes → Is the work CPU-bound or I/O-bound?
              ├── CPU-bound → Goroutine pool + work channel (bounded concurrency).
              └── I/O-bound →
                  ├── Volume < 1K/min → Goroutine + channel. Simple.
                  └── Volume > 1K/min or needs persistence →
                      Message queue (Redis Streams, SQS, RabbitMQ).
```

### SQL vs NoSQL

```
START: What is the shape of your data?
  ├── Relational → POSTGRESQL.
  ├── Document (each record has different schema) →
  │   ├── Can you normalize it? → POSTGRESQL with JSONB.
  │   └── Truly heterogeneous → MONGODB.
  ├── Key-value (sessions, rate limits, counters) → REDIS.
  ├── Time-series (IoT readings, metrics) → TIMESCALEDB or CLICKHOUSE.
  └── Graph (social networks) → NEO4J or PostgreSQL recursive CTEs.
```

**PostgreSQL + Redis is enough for 99% of Gin APIs.**

### DDD vs Simple CRUD

```
START: What does your domain model look like?
  ├── Entities with direct DB mapping, validation, computed fields →
  │   REPOSITORY PATTERN. handler → service → repository. Done.
  ├── Business rules spanning multiple entities + invariants →
  │   ├── 1–3 bounded areas → DOMAIN SERVICES.
  │   └── 4+ bounded areas → CONSIDER BOUNDED CONTEXTS.
  └── Rich domain with complex state machines, multiple teams →
      DDD (aggregates, domain events, bounded contexts).
```

### Layered vs Clean vs Hexagonal Architecture

```
START: Do your tests need to swap out the web framework or database?
  ├── No → LAYERED ARCHITECTURE. Interfaces at repository layer only.
  └── Yes → Why?
      ├── Testing business logic without a DB → Mock the repository interface.
      │   Layered architecture with interfaces is enough.
      └── Planning to swap Gin for another framework →
          This almost never happens. Don't pay the complexity tax in advance.
```
