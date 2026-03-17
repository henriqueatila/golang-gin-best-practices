# golang-gin-best-practices

Agent Skills for building production-grade REST APIs with Go and the Gin framework.

[![Release](https://img.shields.io/github/v/release/henriqueatila/golang-gin-best-practices?label=release)](https://github.com/henriqueatila/golang-gin-best-practices/releases/latest)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Skills.sh](https://img.shields.io/badge/skills.sh-golang--gin--best--practices-blue?logo=data:image/svg+xml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHZpZXdCb3g9IjAgMCAyNCAyNCIgZmlsbD0id2hpdGUiPjxwYXRoIGQ9Ik0xMiAyTDIgNy41djlMMTIgMjJsMTAtNS41di05TDEyIDJ6Ii8+PC9zdmc+)](https://skills.sh/henriqueatila/golang-gin-best-practices)

---

## Skills

| Skill | Description | Install |
|---|---|---|
| **golang-gin-architect** | Software architect — system design, complexity assessment, API design, skill orchestration | `npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-architect` |
| **golang-gin-api** | Core REST API — routing, handlers, binding, error handling, project structure | `npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-api` |
| **golang-gin-auth** | JWT authentication, login handler, RBAC middleware, token lifecycle | `npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-auth` |
| **golang-gin-database** | PostgreSQL with GORM or sqlx, repository pattern, migrations, connection pooling | `npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-database` |
| **golang-gin-psql-dba** | PostgreSQL DBA — schema design, index strategy, migration safety, extensions (ParadeDB, pgvector, PostGIS, TimescaleDB) | `npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-psql-dba` |
| **golang-gin-swagger** | Swagger/OpenAPI docs with swaggo/swag, annotations, Swagger UI, doc generation | `npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-swagger` |
| **golang-gin-testing** | Unit tests with httptest, integration tests with testcontainers, e2e flows, load testing | `npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-testing` |
| **golang-gin-deploy** | Multi-stage Dockerfile, docker-compose, Kubernetes manifests, CI/CD pipelines | `npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-deploy` |

---

## Quick Start

### Install all skills

```bash
npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-architect
npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-api
npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-auth
npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-database
npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-psql-dba
npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-swagger
npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-testing
npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-deploy
```

### Install individual skills

```bash
# Architecture brain — system design, complexity assessment, skill orchestration
npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-architect

# Core API only
npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-api

# Add authentication to an existing project
npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-auth

# Add database layer
npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-database

# Add DBA/architect guidance (schema design, indexes, extensions)
npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-psql-dba

# Add Swagger/OpenAPI documentation
npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-swagger

# Add testing infrastructure
npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-testing

# Add deployment configuration
npx skills add henriqueatila/golang-gin-best-practices --skill golang-gin-deploy
```

### Manual installation

Download the zip package for any skill from the `skills/` directory and extract it into your project's `.claude/skills/` folder:

```bash
# Example: download and extract golang-gin-api
curl -L https://github.com/henriqueatila/golang-gin-best-practices/raw/main/skills/golang-gin-api.zip -o golang-gin-api.zip
unzip golang-gin-api.zip -d .claude/skills/golang-gin-api/
```

---

## Architecture

### Progressive Disclosure

Each skill follows a two-level structure:

1. **SKILL.md** — Quick-reference guide loaded automatically. Covers key rules, conventions, and patterns in under 150 lines. Includes a Quality Mindset section inspired by [NoPUA](https://github.com/wuji-labs/nopua) for trust-based problem-solving.
2. **references/*.md** — Deep-dive files loaded on demand. Each covers one topic with complete, compilable code examples.

This means the agent loads minimal context by default and fetches reference files only when needed — keeping token usage low while providing comprehensive coverage.

### Composability

Skills are designed to work together with shared conventions:

- **golang-gin-architect** is the "brain" — assesses complexity, selects patterns, routes to specific skills, and orchestrates workflows
- **golang-gin-api** defines the project structure, `AppError` type, and handler patterns that all other skills follow
- **golang-gin-auth** provides JWT middleware consumed by **golang-gin-api** route groups
- **golang-gin-database** provides the `UserRepository` interface and implementations used by **golang-gin-auth** and **golang-gin-api** handlers
- **golang-gin-psql-dba** provides PostgreSQL architecture decisions (schema design, indexes, migrations, extensions) that inform **golang-gin-database** implementations
- **golang-gin-testing** tests the handlers, services, and middleware defined across all skills
- **golang-gin-swagger** documents the endpoints from **golang-gin-api** and **golang-gin-auth** with OpenAPI annotations
- **golang-gin-deploy** builds the project structure from **golang-gin-api** and wires health checks from **golang-gin-api**

Start with `golang-gin-architect` for architecture decisions, `golang-gin-api` for implementation, and add skills incrementally as your project grows.

---

## Security Coverage

Skills collectively prevent common vulnerabilities:

| Threat | Skill | Technique |
|--------|-------|-----------|
| SQL Injection | database | Parameterized queries (`?` / `$N`), never string concatenation |
| XSS | api | `html.EscapeString` + `strings.TrimSpace` after binding |
| Brute Force | auth | `IPRateLimiter` middleware on auth endpoints |
| Token Theft | auth | JWT blacklisting (jti), short TTL, refresh rotation |
| Slowloris DoS | api | `ReadHeaderTimeout: 10s` on `http.Server` |
| IP Spoofing | api | `r.SetTrustedProxies()` for `c.ClientIP()` |
| Credential Leak | api, deploy | `json:"-"` on PasswordHash, env vars, `.dockerignore` |
| CAPTCHA Bypass | auth | Server-side reCAPTCHA/hCaptcha verification middleware |
| Path Traversal | api | `filepath.Base(file.Filename)` for uploads |
| Data Leaks | api | Generic error messages, `err.Error()` never exposed to clients |

---

## Directory Structure

```
golang-gin-best-practices/
├── README.md
├── CLAUDE.md                        # Agent guidance for this repo
├── AGENTS.md                        # Multi-agent guidance
├── LICENSE
│
├── skills/
│   ├── golang-gin-architect/        # System design, complexity, skill orchestration (53 refs)
│   │   ├── SKILL.md
│   │   ├── metadata.json
│   │   └── references/              # complexity-*, system-design-*, data-patterns-*,
│   │                                # resilience-*, api-design-*, cross-cutting-*,
│   │                                # redis-*, messaging-*, object-storage-*,
│   │                                # error-flow-*, golden-main-*, grpc-interop-*,
│   │                                # data-ownership-*, adr-*, clean-architecture-*,
│   │                                # tech-debt-*, skill-orchestration-*
│   │
│   ├── golang-gin-api/              # Server setup, routing, handlers, binding, errors (29 refs)
│   │   ├── SKILL.md
│   │   ├── metadata.json
│   │   └── references/              # server-*, routing-*, middleware-*, error-handling-*,
│   │                                # websocket-*, rate-limiting-*, file-uploads-*,
│   │                                # background-jobs-*
│   │
│   ├── golang-gin-auth/             # JWT middleware, RBAC, OAuth2, CAPTCHA (16 refs)
│   │   ├── SKILL.md
│   │   ├── metadata.json
│   │   └── references/              # auth-*, jwt-patterns-*, rbac-*, oauth2-*, captcha-*
│   │
│   ├── golang-gin-database/         # Repository pattern, GORM/sqlx, Redis (19 refs)
│   │   ├── SKILL.md
│   │   ├── metadata.json
│   │   └── references/              # setup-*, gorm-patterns-*, sqlx-patterns-*,
│   │                                # migrations-*, redis-patterns-*
│   │
│   ├── golang-gin-psql-dba/         # Schema design, indexes, migrations, extensions (39 refs)
│   │   ├── SKILL.md
│   │   ├── metadata.json
│   │   └── references/              # schema-design-*, migration-impact-*, index-strategy-*,
│   │                                # query-performance-*, extensions-toolkit-*,
│   │                                # paradedb-*, pgvector-*, postgis-*, timescaledb-*,
│   │                                # row-level-security-*, backup-and-recovery-*,
│   │                                # replication-and-ha-*
│   │
│   ├── golang-gin-swagger/          # Swagger/OpenAPI annotations, UI, CI/CD (9 refs)
│   │   ├── SKILL.md
│   │   ├── metadata.json
│   │   └── references/              # annotations-*, setup-*, ci-cd-*
│   │
│   ├── golang-gin-testing/          # Unit, integration, e2e, load testing (13 refs)
│   │   ├── SKILL.md
│   │   ├── metadata.json
│   │   └── references/              # test-patterns-*, unit-tests-*, integration-tests-*,
│   │                                # e2e-*, load-testing-*
│   │
│   ├── golang-gin-deploy/           # Docker, docker-compose, K8s, observability (13 refs)
│   │   ├── SKILL.md
│   │   ├── metadata.json
│   │   └── references/              # configuration-*, dockerfile-*, docker-compose-*,
│   │                                # kubernetes-*, observability-*
│   │
│   ├── golang-gin-architect.zip     # Packaged skills (SKILL.md + metadata.json + references/)
│   ├── golang-gin-api.zip
│   ├── golang-gin-auth.zip
│   ├── golang-gin-database.zip
│   ├── golang-gin-psql-dba.zip
│   ├── golang-gin-swagger.zip
│   ├── golang-gin-testing.zip
│   └── golang-gin-deploy.zip
```

---

## Design Principles

**Opinionated patterns** — Each skill teaches one right way, not five options. The patterns are chosen for production correctness, not tutorial simplicity.

**PostgreSQL default** — All database examples target PostgreSQL. GORM and sqlx are both covered; the repository interface allows swapping implementations without touching business logic.

**Production-first** — Examples use `gin.New()` + explicit middleware instead of `gin.Default()`, `log/slog` instead of `fmt.Println`, `ShouldBind*` instead of `Bind*`, and environment variables instead of hardcoded configuration.

**Go 1.24+** — All code targets Go 1.24 or later. Uses standard library features (`log/slog`, `context`, `errors.As`) and avoids deprecated patterns.

**Thin handlers** — Handlers bind input, call a service, and format the response. Business logic lives in services. Data access lives in repositories. This is enforced consistently across all skills.

**Context propagation** — All blocking calls receive `c.Request.Context()` (handlers) or a propagated `context.Context` (services and repositories). This enables proper cancellation and tracing.

---

## Contributing

1. Follow the design principles above — new patterns must be production-ready
2. Code examples must compile and handle errors (no `_` for errors, no `fmt.Println`)
3. SKILL.md files must stay under 150 lines — move detail to reference files
4. Reference files must stay under 150 lines — split by logical boundaries if needed
5. Each SKILL.md must include Quality Mindset, Scope, and Security sections
6. Use `gin.New()`, `log/slog`, `ShouldBind*`, and `context.Context` consistently
7. Verify all Gin API calls match official documentation before submitting a PR

---

## License

MIT License — see [LICENSE](LICENSE) for details.
