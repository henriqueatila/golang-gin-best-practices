# Skill Orchestration Guide

How and when to activate each skill in the gingo ecosystem. Decision matrix, common workflows, and skill composition patterns.

## Table of Contents

- [Skill Overview](#skill-overview)
- [Decision Matrix](#decision-matrix)
- [Common Workflows](#common-workflows)
- [Skill Composition Patterns](#skill-composition-patterns)
- [Skill Boundary Rules](#skill-boundary-rules)
- [Troubleshooting: Which Skill?](#troubleshooting-which-skill)

---

## Skill Overview

| Skill | Domain | One-line Purpose |
|---|---|---|
| **golang-gin-architect** | Architecture | System design, complexity assessment, pattern selection, skill routing |
| **golang-gin-api** | HTTP Layer | Routing, handlers, binding, middleware, error handling |
| **golang-gin-auth** | Security | JWT, RBAC, login/register, token lifecycle, protected routes |
| **golang-gin-database** | Data Access | GORM/sqlx wiring, repository pattern, migrations tooling, transactions |
| **golang-gin-psql-dba** | Database | Schema design, index strategy, query optimization, extensions, migration safety |
| **golang-gin-testing** | Quality | Unit/integration/e2e tests, httptest, testcontainers, table-driven tests |
| **golang-gin-deploy** | Operations | Docker, docker-compose, Kubernetes, CI/CD, 12-factor config |

**Key distinction:**
- `golang-gin-database` = **how to write Go code** that talks to PostgreSQL (GORM, sqlx, repository pattern)
- `golang-gin-psql-dba` = **how to design and optimize PostgreSQL** (schemas, indexes, migrations, extensions)
- `golang-gin-architect` = **when to use what** and how skills compose together

---

## Decision Matrix

Use this to determine which skill(s) to activate for a given task.

### By Task Type

| Task | Primary Skill | May Also Need |
|---|---|---|
| "Create a new endpoint" | golang-gin-api | golang-gin-database, golang-gin-testing |
| "Add login/signup" | golang-gin-auth | golang-gin-api, golang-gin-database |
| "Design the schema" | golang-gin-psql-dba | golang-gin-database (migration tooling) |
| "Write repository code" | golang-gin-database | golang-gin-psql-dba (schema decisions) |
| "Query is slow" | golang-gin-psql-dba | golang-gin-database (query code) |
| "Add tests" | golang-gin-testing | (reads other skills for patterns) |
| "Dockerize the app" | golang-gin-deploy | — |
| "Set up CI/CD" | golang-gin-deploy | golang-gin-testing (test step) |
| "Should I use microservices?" | golang-gin-architect | — |
| "How to structure this feature?" | golang-gin-architect | golang-gin-api, golang-gin-database |
| "Add caching" | golang-gin-architect | golang-gin-api (middleware) |
| "Add full-text search" | golang-gin-psql-dba | golang-gin-database (Go code) |
| "Add WebSocket support" | golang-gin-api | — |
| "Rate limit endpoints" | golang-gin-api | — |
| "Set up monitoring" | golang-gin-deploy | golang-gin-architect (observability design) |

### By Keyword

| If the user mentions... | Activate |
|---|---|
| route, handler, middleware, binding, JSON response | golang-gin-api |
| JWT, token, login, signup, RBAC, permission, role | golang-gin-auth |
| GORM, sqlx, repository, migration tool, transaction | golang-gin-database |
| schema, index, EXPLAIN, ALTER TABLE, extension, pgvector, PostGIS | golang-gin-psql-dba |
| test, httptest, testcontainers, mock, coverage | golang-gin-testing |
| Docker, Kubernetes, CI/CD, deploy, helm, health check | golang-gin-deploy |
| architecture, microservice, monolith, CQRS, pattern, scale, design | golang-gin-architect |

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

**For simple CRUD features (80% of cases), skip step 1** — go directly to schema → repository → handler → tests.

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
3. golang-gin-database   → Optimize repository queries, add connection pool tuning
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
1. golang-gin-architect  → Project structure decision, technology choices, ADRs
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

Design the database first, then work upward.

```
golang-gin-psql-dba (schema) → golang-gin-database (repository) → golang-gin-api (handler) → golang-gin-testing
```

**Why:** Schema changes are the hardest to modify later. Get the data model right first.

### Pattern 2: API-First

Design the API contract first, then work downward.

```
golang-gin-api (contract/types) → golang-gin-database (repository to match) → golang-gin-psql-dba (schema to match)
```

**When:** External API consumers need to see the contract before implementation. Use OpenAPI spec as the starting point.

### Pattern 3: Test-First (TDD)

Write tests first, then implement.

```
golang-gin-testing (test) → golang-gin-api (handler) → golang-gin-database (repository) → golang-gin-psql-dba (schema)
```

**When:** Requirements are very clear and well-defined. Each test drives the next implementation layer.

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

### "I'm not sure which skill to use"

Ask these questions:

1. **Is this about code or decisions?**
   - Code → specific skill (golang-gin-api, golang-gin-database, golang-gin-auth, etc.)
   - Decisions → golang-gin-architect

2. **Is this about HTTP or data?**
   - HTTP (routes, middleware, handlers) → golang-gin-api
   - Data (queries, schema, migrations) → golang-gin-database or golang-gin-psql-dba

3. **Is this about Go code or PostgreSQL?**
   - Go code that talks to DB → golang-gin-database
   - PostgreSQL itself (DDL, indexes, EXPLAIN) → golang-gin-psql-dba

4. **Is this about running the app or building it?**
   - Building → golang-gin-api, golang-gin-database, golang-gin-auth
   - Running → golang-gin-deploy

5. **Is this about verifying correctness?**
   - Yes → golang-gin-testing

### Common Misroutes

| User Says | Seems Like | Actually |
|---|---|---|
| "Add a database" | golang-gin-database | golang-gin-psql-dba (schema) THEN golang-gin-database (code) |
| "Make it faster" | golang-gin-architect | golang-gin-psql-dba first (90% of perf issues are DB) |
| "Add middleware" | golang-gin-api | golang-gin-auth if it's auth middleware |
| "Set up migrations" | golang-gin-psql-dba | golang-gin-database (tooling) with golang-gin-psql-dba (safety review) |
| "Scale the app" | golang-gin-deploy | golang-gin-architect first (is scaling the right answer?) |
