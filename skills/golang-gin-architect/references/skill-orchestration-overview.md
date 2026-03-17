# Skill Orchestration — Overview and Decision Matrix

How and when to activate each skill in the gingo ecosystem.

Companion to `skill-orchestration-workflows.md` (common workflows, composition patterns, boundary rules).

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
