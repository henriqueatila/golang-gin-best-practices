# Skill Orchestration — Workflows, Composition, and Boundaries

Companion to `skill-orchestration-overview.md` (skill overview and decision matrix).

---

## Common Workflows

### New Feature (End-to-End)

```
1. golang-gin-architect  → Assess complexity, choose patterns
2. golang-gin-psql-dba   → Design schema, plan migration
3. golang-gin-database   → Implement repository + migration files
4. golang-gin-api        → Implement handler + service + routes
5. golang-gin-auth       → Add auth middleware (if protected)
6. golang-gin-testing    → Write unit + integration tests
7. golang-gin-deploy     → Update Docker/K8s configs (if needed)
```

**For simple CRUD (80% of cases), skip step 1** — go directly to schema → repository → handler → tests.

### Add Authentication to Existing App

```
1. golang-gin-auth       → JWT middleware, login/register handlers, RBAC
2. golang-gin-database   → User model, token store (if refresh tokens in DB)
3. golang-gin-psql-dba   → User table schema, indexes on email
4. golang-gin-api        → Wire auth middleware to route groups
5. golang-gin-testing    → Auth test helpers, protected route tests
```

### Performance Investigation

```
1. golang-gin-psql-dba   → EXPLAIN ANALYZE, index analysis, query optimization
2. golang-gin-architect  → Caching strategy, read replicas decision
3. golang-gin-database   → Optimize repository queries, connection pool tuning
4. golang-gin-testing    → Benchmark tests
5. golang-gin-deploy     → Horizontal scaling, resource limits
```

### Database Migration (Schema Change)

```
1. golang-gin-psql-dba   → Migration safety analysis (lock levels, zero-downtime)
2. golang-gin-database   → Write migration files (golang-migrate)
3. golang-gin-testing    → Test migration up/down
4. golang-gin-deploy     → Update deployment to run migration before app starts
```

### Greenfield Project Setup

```
1. golang-gin-architect  → Project structure, technology choices, ADRs
2. golang-gin-api        → Server setup, project layout, base middleware
3. golang-gin-database   → Database connection, initial schema
4. golang-gin-psql-dba   → Schema design, initial indexes
5. golang-gin-auth       → Auth setup (if needed from start)
6. golang-gin-testing    → Test infrastructure, CI integration
7. golang-gin-deploy     → Docker, docker-compose for local dev
```

---

## Skill Composition Patterns

### Pattern 1: Schema-First (Recommended)

```
golang-gin-psql-dba (schema) → golang-gin-database (repository) → golang-gin-api (handler) → golang-gin-testing
```

**Why:** Schema changes are the hardest to modify later. Get the data model right first.

### Pattern 2: API-First

```
golang-gin-api (contract/types) → golang-gin-database (repository) → golang-gin-psql-dba (schema)
```

**When:** External consumers need to see the contract before implementation. Use OpenAPI spec as starting point.

### Pattern 3: Test-First (TDD)

```
golang-gin-testing (test) → golang-gin-api (handler) → golang-gin-database (repository) → golang-gin-psql-dba (schema)
```

**When:** Requirements are clear and well-defined. Each test drives the next implementation layer.

---

## Skill Boundary Rules

### What Each Skill Should NOT Do

| Skill | Should NOT |
|---|---|
| golang-gin-api | Write SQL queries, design schemas, make auth decisions |
| golang-gin-auth | Design database schemas, set up deployment |
| golang-gin-database | Decide index types, analyze EXPLAIN plans, choose extensions |
| golang-gin-psql-dba | Write Go repository code, implement handlers |
| golang-gin-testing | Implement features, change production code |
| golang-gin-deploy | Make architecture decisions, change business logic |
| golang-gin-architect | Write implementation code (routes to specific skills instead) |

### Overlap Resolution

| Overlap Area | Owner | Other Skill's Role |
|---|---|---|
| Database connection setup | golang-gin-database | golang-gin-psql-dba advises pool sizing |
| Migration files | golang-gin-database | golang-gin-psql-dba reviews safety |
| Auth middleware | golang-gin-auth | golang-gin-api wires it to routes |
| Error handling | golang-gin-api | golang-gin-architect defines strategy |
| Docker health checks | golang-gin-deploy | golang-gin-api implements `/health` endpoint |
| Test infrastructure | golang-gin-testing | golang-gin-deploy sets up test DB in CI |

---

## Troubleshooting: Which Skill?

1. **Code or decisions?** — Code → specific skill; Decisions → golang-gin-architect
2. **HTTP or data?** — HTTP (routes, handlers) → golang-gin-api; Data → golang-gin-database or golang-gin-psql-dba
3. **Go code or PostgreSQL?** — Go code → golang-gin-database; PostgreSQL DDL/indexes → golang-gin-psql-dba
4. **Building or running?** — Building → api/database/auth; Running → golang-gin-deploy
5. **Verifying correctness?** — golang-gin-testing

### Common Misroutes

| User Says | Seems Like | Actually |
|---|---|---|
| "Add a database" | golang-gin-database | golang-gin-psql-dba (schema) THEN golang-gin-database (code) |
| "Make it faster" | golang-gin-architect | golang-gin-psql-dba first (90% of perf issues are DB) |
| "Add middleware" | golang-gin-api | golang-gin-auth if it's auth middleware |
| "Set up migrations" | golang-gin-psql-dba | golang-gin-database (tooling) + golang-gin-psql-dba (safety review) |
| "Scale the app" | golang-gin-deploy | golang-gin-architect first (is scaling the right answer?) |
