# Complexity Assessment — Pattern Selection Matrix and Right-Size Thinking

Pattern selection with complexity costs, gates, and the right-size matrix by team/stage.

---

## Right-Size Thinking Matrix

Find the row that best matches your situation. Use "Max Pattern Score" as a ceiling.

| Team Size | Stage | Traffic (RPM) | Data Complexity | Recommended Architecture | Max Score |
|---|---|---|---|---|---|
| 1–3 | MVP | < 1K | CRUD | Flat layout, direct SQL with sqlx | 5–8 |
| 1–3 | MVP | < 1K | Relational | handler → service → repository, PostgreSQL | 8–10 |
| 1–3 | Growth | 1K–10K | Relational | Same + Redis for caching hot reads | 10–13 |
| 3–8 | MVP | < 10K | Relational | Feature modules, shared repository interfaces | 12–15 |
| 3–8 | Growth | 10K–100K | Relational + events | Feature modules + async worker, Redis queue | 15–20 |
| 3–8 | Growth | 10K–100K | Complex rules | Domain services per bounded area | 16–20 |
| 8–15 | Growth | 10K–100K | Relational | Modular monolith, consider 1–2 service extractions | 18–22 |
| 8–15 | Mature | 100K+ | Mixed | Limited microservices (proven pain only), CQRS for read-heavy | 22–28 |
| 15+ | Mature | 100K+ | Event-driven | Multiple services, event sourcing for audit-critical domains | 28+ |

---

## Pattern Selection Matrix

### Repository Pattern

| | |
|---|---|
| Complexity cost | 1 |
| What it solves | Decouples business logic from data access; enables mocking in tests |
| You need it when | You want to mock the DB in unit tests, or have more than one data source |
| You DON'T need it if | Building a prototype where tests aren't required |

```go
type Repository interface {
    GetByID(ctx context.Context, id string) (*User, error)
    Create(ctx context.Context, u *User) error
}
```

### Service Layer

| | |
|---|---|
| Complexity cost | 1 |
| What it solves | Business logic outside handlers; testable without HTTP |
| You need it when | You have business logic beyond "validate + persist" |
| You DON'T need it if | Handler literally only validates, calls one repo method, returns result |

### CQRS

| | |
|---|---|
| Complexity cost | 4 |
| What it solves | Separate read and write models; optimize each independently |
| Simpler alternative | Single model with `ListXxx` method returning a DTO |
| You need it when | Read queries require joining 5+ tables and are 10x the volume of writes |

### Event Sourcing

| | |
|---|---|
| Complexity cost | 5 |
| What it solves | Complete audit trail; rebuild any past state; temporal queries |
| Simpler alternative | Append-only audit log table in PostgreSQL |
| You need it when | Need to reconstruct state at any point in time AND complex state transitions AND team has operational experience |

### Saga / Choreography

| | |
|---|---|
| Complexity cost | 4 |
| What it solves | Distributed transactions across multiple services without 2PC |
| Simpler alternative | Database transactions (if same DB) or a single service owning the workflow |
| You need it when | Multi-step workflow spanning 3+ services with partial failure compensation |

### Circuit Breaker

| | |
|---|---|
| Complexity cost | 2 |
| What it solves | Prevents cascade failure when a downstream dependency is unhealthy |
| Simpler alternative | Timeout + retry with backoff |
| You need it when | Dependency fails in a way that ties up goroutines (slow failures) |

```go
cb := gobreaker.NewCircuitBreaker(gobreaker.Settings{
    Name:        "payment-service",
    MaxRequests: 5,
    Interval:    10 * time.Second,
    Timeout:     30 * time.Second,
    ReadyToTrip: func(counts gobreaker.Counts) bool {
        failRatio := float64(counts.TotalFailures) / float64(counts.Requests)
        return counts.Requests >= 10 && failRatio >= 0.6
    },
})
```

### Feature Flags

| | |
|---|---|
| Complexity cost | 2 |
| What it solves | Deploy code without activating it; gradual rollouts; A/B testing |
| Simpler alternative | Environment variables (for simple on/off by deployment) |
| You need it when | Per-user or per-percentage rollouts, or to roll back without redeploying |

```go
func (s *FeatureService) IsEnabled(ctx context.Context, flag, userID string) bool {
    f, err := s.repo.GetFlag(ctx, flag)
    if err != nil || !f.Enabled { return false }
    if slices.Contains(f.UserIDs, userID) { return true }
    h := fnv.New32a()
    h.Write([]byte(userID + flag))
    return int(h.Sum32()%100) < f.RolloutPct
}
```

### API Gateway

| | |
|---|---|
| Complexity cost | 3 |
| What it solves | Single entry point for routing, auth, rate limiting, SSL termination |
| Simpler alternative | Nginx or Caddy reverse proxy |
| You need it when | 3+ services with different auth models, request transformation or fan-out needed |

### Service Mesh

| | |
|---|---|
| Complexity cost | 5 |
| What it solves | mTLS between services, observability at network layer, traffic management |
| Simpler alternative | Application-level TLS + OpenTelemetry + circuit breaker |
| You need it when | 10+ services, strict mTLS compliance, dedicated platform team |
