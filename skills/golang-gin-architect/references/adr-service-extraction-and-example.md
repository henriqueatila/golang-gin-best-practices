# Architecture Decision Records — Service Extraction Template and Worked Example

Companion to `adr-format-and-templates.md` (ADR format, when to write, database/auth templates).

---

## Template: Service Extraction

```markdown
# ADR-NNN: Extract [Feature] into Separate Service

## Context
[Why now? What pain in the monolith? Be specific: deploy coupling, scaling bottleneck, team contention]

## Decision
Extract [feature] into a standalone service communicating via [HTTP / gRPC / events].

## Extraction Checklist
- [ ] Feature has clear bounded context (few shared data dependencies)
- [ ] Team can independently deploy and operate the service
- [ ] CI/CD pipeline exists for the new service
- [ ] Monitoring and alerting configured
- [ ] Rollback plan defined
- [ ] Data migration plan (if splitting database)

## Boundary Definition
| Data/Entity | Stays in Monolith | Moves to New Service |
|---|---|---|
| Users | X | |
| Orders | | X |
| Payments | | X |
| Products | X | |

## Communication Pattern
| Interaction | Pattern | Why |
|---|---|---|
| Create order | Sync HTTP | Caller needs confirmation |
| Order shipped | Async event | Notification, non-blocking |
| Get user for order | Sync HTTP | Need data to process |

## Alternatives Considered
### Keep in Monolith + Modularize
- Pros: No distributed system complexity, shared transactions
- Cons: Doesn't solve [specific pain point]

### Extract as Library
- Pros: Code reuse without network boundary
- Cons: Still coupled deployment
```

---

## Worked Example: ADR-001 Use PostgreSQL as Primary Database

```markdown
# ADR-001: Use PostgreSQL as Primary Database

**Status:** Accepted
**Date:** 2026-01-15
**Deciders:** Backend team (3 engineers)

## Context

Building an e-commerce API for a B2B marketplace. Expected load: ~5K RPM initially,
growing to ~50K RPM in 12 months. Data includes products, orders, users, and inventory.
Team has strong SQL experience. Need full-text search for products and transactional
consistency for orders.

## Decision

Use PostgreSQL 16 as the sole database. Use built-in tsvector for full-text search.
Add Redis later only if measured latency exceeds targets.

## Alternatives Considered

### MongoDB
- Pros: Flexible product schemas, horizontal scaling
- Cons: Weaker transactions, no MongoDB experience, needs separate search solution
- Why rejected: Transaction safety for orders is critical. Product schema is actually
  structured. Team learning curve adds timeline risk.

### PostgreSQL + Elasticsearch
- Pros: Best-in-class search
- Cons: Two systems to maintain, data sync complexity
- Why rejected: PostgreSQL tsvector + GIN index handles requirements. ParadeDB is a
  drop-in upgrade without a separate system if needed.

## Consequences

### Positive
- Single database to operate, monitor, and backup
- ACID transactions for order processing
- Team productive immediately (familiar technology)
- Extension path: pgvector for recommendations, ParadeDB for advanced search

### Negative
- Horizontal scaling harder than MongoDB (not needed at 50K RPM)
- Schema changes require migrations (mitigated by golang-gin-psql-dba migration safety guide)

### Risks
- Unpredictable product schemas → mitigated by JSONB columns
- Full-text search outgrows tsvector → mitigated by ParadeDB upgrade path

## Follow-up Actions
- [x] Set up PostgreSQL 16 with Docker
- [x] Define initial schema
- [ ] Configure connection pooling
- [ ] Set up backup strategy
```

---

## ADR Index Template

`docs/adr/README.md`:

```markdown
# Architecture Decision Records

| # | Title | Status | Date |
|---|---|---|---|
| 001 | [Use PostgreSQL as Primary Database](001-use-postgresql.md) | Accepted | 2026-01-15 |
| 002 | [JWT with Refresh Tokens for Auth](002-jwt-auth.md) | Accepted | 2026-01-20 |
| 003 | [URL Path API Versioning](003-api-versioning.md) | Accepted | 2026-02-01 |
```

**Naming convention:** `NNN-short-slug.md` (e.g., `001-use-postgresql.md`)

Caching and versioning ADR templates: see **[adr-format-and-templates.md](adr-format-and-templates.md)**.
