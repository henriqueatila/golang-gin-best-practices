# Architecture Decision Records — Format and Templates

Lightweight templates for documenting architecture decisions. Store in `docs/adr/` in your project.

Companion to `adr-service-extraction-and-example.md` (service extraction template, worked example).

---

## ADR Format

```markdown
# ADR-NNN: [Short Decision Title]

**Status:** Proposed | Accepted | Deprecated | Superseded by ADR-XXX
**Date:** YYYY-MM-DD
**Deciders:** [who was involved]

## Context
What is the problem? Why are we making this decision now?
What forces are at play (technical, business, team)?

## Decision
What did we choose? State it clearly in one sentence, then elaborate.

## Alternatives Considered

### Option A: [Name]
- Pros: ...
- Cons: ...
- Why rejected: ...

## Consequences
- Positive: What gets better?
- Negative: What gets worse or harder?
- Risks: What could go wrong?

## Follow-up Actions
- [ ] Action items resulting from this decision
```

Rules: number sequentially (ADR-001, ADR-002, ...); never delete — mark as `Deprecated` or `Superseded by ADR-XXX`; keep short (1-2 pages); write ADRs for decisions, not implementations.

**When to write:** choosing between 2+ technologies; changing an established pattern; adding infrastructure; making an irreversible or expensive-to-reverse decision; decision affects multiple teams.

**Don't write for:** obvious choices, minor refactoring, bug fixes, style preferences.

ADR lifecycle: `Proposed → Accepted → [lives forever] → Deprecated | Superseded by ADR-XXX`

---

## Template: Database Choice

```markdown
# ADR-NNN: Database Selection for [Feature/Service]

## Context
[Describe data requirements: volume, access patterns, consistency needs, team familiarity]

## Decision
Use [PostgreSQL / MongoDB / Redis / etc.] as the primary datastore for [scope].

## Evaluation Criteria
| Criteria | Weight | Option A | Option B |
|---|---|---|---|
| Team familiarity | High | ... | ... |
| Query flexibility | Medium | ... | ... |
| Operational cost | High | ... | ... |

## Alternatives Considered
### PostgreSQL
- Pros: ACID, mature ecosystem, pgvector/PostGIS, team knows SQL
### MongoDB
- Pros: Flexible schema, easy horizontal scaling
- Cons: Eventual consistency by default, weaker joins

## Follow-up Actions
- [ ] Set up database infrastructure; define schema; configure connection pooling
```

---

## Template: Auth Strategy

```markdown
# ADR-NNN: Authentication and Authorization Strategy

## Context
[Who are the users? Internal/external? Mobile/web/API? Security requirements?]

## Decision
Use [JWT with refresh tokens / session cookies / OAuth2 + OIDC / API keys]
for authentication. Use [RBAC / ABAC / simple role check] for authorization.

## Alternatives Considered
### JWT with Refresh Tokens
- Pros: Stateless verification, works across services, mobile-friendly
- Cons: Can't revoke individual tokens without a blocklist
### Session Cookies
- Pros: Simple, revocable, well-understood
- Cons: Requires session store (Redis), not ideal for mobile
### OAuth2 + External Provider
- Pros: Offload auth complexity, social login support
- Cons: Vendor dependency, more complex integration
```

---

## Template: Caching Layer

```markdown
# ADR-NNN: Caching Strategy for [Feature/System]

## Context
[What's slow? Read/write ratio? Current latency? Target latency? Freshness requirements?]

## Decision
Use [HTTP cache headers / in-memory cache / Redis] for [scope].
Cache TTL: [duration]. Invalidation strategy: [TTL / write-through / event-driven].

## Alternatives Considered
### No Cache (Optimize Queries) — Try this first
### In-Memory (ristretto) — < 100MB, single-instance
### Redis — > 100MB, multi-instance, need consistent cache
```

---

## Template: API Versioning

```markdown
# ADR-NNN: API Versioning Strategy

## Decision
Use URL path versioning (`/api/v1/`, `/api/v2/`) with minimum 3-month
deprecation notice for external consumers.

## Version Bumping Rules
| Change Type | Version Bump? |
|---|---|
| Add response field | No |
| Remove response field | Yes |
| Change field type | Yes |
| Change pagination format | Yes |
```

---

## See Also

- `adr-service-extraction-and-example.md` — service extraction template and worked PostgreSQL example
