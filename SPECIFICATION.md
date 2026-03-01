# Specification: Build the `gin-skills` Repository

## Mission

Build a public repository of **Agent Skills** for Go REST API development using the **Gin web framework**, published on [skills.sh](https://skills.sh). As of March 2026, **no Gin/Echo/Fiber skill exists on skills.sh**, making this the first Go web framework skill collection in the ecosystem.

The repository must follow the [Agent Skills open standard](https://agentskills.io/) and the [Claude Code skill authoring best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices).

---

## CRITICAL: Official API Reference

**Before writing ANY code**, read `GIN-API-REFERENCE.md` in this repository. It contains the verified Gin API surface extracted from the official documentation at [gin-gonic.com/en/docs](https://gin-gonic.com/en/docs/) and the [gin-gonic/gin GitHub repo](https://github.com/gin-gonic/gin/blob/master/docs/doc.md).

### Anti-Hallucination Rules

1. **NEVER invent methods, struct tags, or parameters** that are not in `GIN-API-REFERENCE.md`.
2. **Binding methods come in two sets**: `Bind*` (auto-aborts with 400) and `ShouldBind*` (returns error). Always use `ShouldBind*` in examples.
3. **Validation uses `go-playground/validator/v10`**. Custom validators are registered via `binding.Validator.Engine().(*validator.Validate)`.
4. **`gin.H`** is `map[string]any`. Do NOT write `map[string]interface{}` in new code.
5. **`c.Copy()` is required before using `*gin.Context` in goroutines**.
6. **`gin.Default()` = Logger + Recovery**. For production patterns, always show `gin.New()` + explicit `r.Use(...)`.
7. **Graceful shutdown uses `http.Server.Shutdown(ctx)`** — use EXACTLY the pattern from GIN-API-REFERENCE.md.
8. **Testing uses `httptest.NewRecorder()` + `router.ServeHTTP(w, req)`**.
9. **Go 1.24+** is the minimum version.
10. When referencing gin-contrib middleware, the import path is `"github.com/gin-contrib/<name>"`.

### If the Reference Doesn't Cover It

For patterns that go BEYOND the Gin framework itself (GORM, sqlx, JWT libraries, Docker, Kubernetes), use mainstream Go community patterns but clearly mark them as architectural recommendations, not Gin API.

---

## Repository Structure

Create exactly this structure:

```
gin-skills/
├── README.md
├── LICENSE                          # MIT License
├── GIN-API-REFERENCE.md             # Official API reference — source of truth
├── gin-api/
│   ├── SKILL.md
│   └── references/
│       ├── routing.md
│       ├── middleware.md
│       └── error-handling.md
├── gin-auth/
│   ├── SKILL.md
│   └── references/
│       ├── jwt-patterns.md
│       └── rbac.md
├── gin-database/
│   ├── SKILL.md
│   └── references/
│       ├── gorm-patterns.md
│       ├── sqlx-patterns.md
│       └── migrations.md
├── gin-testing/
│   ├── SKILL.md
│   └── references/
│       ├── unit-tests.md
│       ├── integration-tests.md
│       └── e2e.md
└── gin-deploy/
    ├── SKILL.md
    └── references/
        ├── dockerfile.md
        ├── docker-compose.md
        └── kubernetes.md
```

Total: 5 SKILL.md files + 14 reference files + README.md + LICENSE + GIN-API-REFERENCE.md = 22 files.

---

## SKILL.md Authoring Rules

### Frontmatter (YAML between `---` markers)

```yaml
---
name: skill-name-here
description: "Concise description of WHAT + WHEN. Be slightly pushy on triggers to combat undertriggering. Max 1024 chars."
---
```

- `name`: lowercase, hyphens only, max 64 chars
- `description`: PRIMARY triggering mechanism. Include what it does, key topics as keywords, explicit trigger phrases. Be "pushy" — Claude undertriggers by default.

### Body Content

1. **Under 500 lines total** (including frontmatter). Hard ceiling.
2. Start with `# Skill Title` followed by a 1-line description
3. Include a `## When to Use` section with bullet points
4. Show the MOST IMPORTANT patterns inline (the 20% that covers 80% of use cases)
5. Reference files for everything else with guidance on when to read them
6. All code examples MUST: compile, follow `gofmt`, handle errors, use `context.Context`, use `log/slog`

### Writing Style

- Imperative form: "Use X" not "You should use X"
- Explain WHY, not just WHAT
- Show the pattern first, then explain it
- Mark critical concerns with `**Critical:**` or `**Warning:**`

---

## Skill-by-Skill Specifications

### 1. `gin-api` — Core REST API Development

**Scope:** Server setup, routing, handlers, request binding/validation, response formatting, error handling, project structure, middleware basics.

**SKILL.md must contain (inline):**
- Recommended project structure (cmd/internal/pkg layout with Clean Architecture)
- Server setup with graceful shutdown (complete, runnable `main.go` skeleton)
- Handler pattern: thin handlers that call services (with complete example)
- Request binding & validation (JSON, query, URI — show struct tags)
- Route registration with groups (public vs protected, versioned API)
- Centralized error handling (domain errors → HTTP status codes mapping)
- Health check endpoint

**References:**

`references/routing.md` — Route groups and nesting, API versioning (URL path, header-based), path parameter patterns, query parameter binding with pagination, wildcard routes and NoRoute handler, custom validators, request size limits, multipart file upload handling.

`references/middleware.md` — Middleware signature and chain execution order, CORS configuration (dev vs prod), request logging with log/slog (request ID, duration, status), rate limiting with golang.org/x/time/rate, request ID middleware, recovery middleware with custom error response, timeout middleware, custom middleware template.

`references/error-handling.md` — Domain error types (AppError struct), sentinel errors (NotFound, Unauthorized, Forbidden, Conflict, ValidationFailed), handleServiceError function, error wrapping/unwrapping, validation error formatting, panic recovery, consistent JSON error response format.

---

### 2. `gin-auth` — Authentication & Authorization

**Scope:** JWT authentication, middleware-based auth, RBAC, OAuth2 basics, session patterns.

**SKILL.md must contain (inline):**
- JWT middleware pattern (extract token, validate, inject claims into context)
- Login handler (validate credentials, generate access + refresh tokens)
- Claims struct with standard and custom fields
- Token generation and validation functions
- Protected route example (middleware applied to route group)
- Getting current user from context in any handler

**References:**

`references/jwt-patterns.md` — Access + refresh token architecture, token refresh endpoint, token blacklisting (Redis-based), storage recommendations (httpOnly cookies vs localStorage), custom claims (user ID, roles, permissions, tenant ID), RS256 vs HS256, complete auth flow example.

`references/rbac.md` — Role-based middleware (RequireRole, RequireAnyRole), permission-based middleware, role hierarchy, multi-tenant authorization, resource-level authorization, admin impersonation, complete RBAC example.

---

### 3. `gin-database` — Database Integration

**Scope:** Repository pattern, connection management, GORM patterns, sqlx patterns, migrations, transactions.

**SKILL.md must contain (inline):**
- Repository interface pattern (define interface at consumer, implement with GORM or sqlx)
- Database connection setup with pool configuration
- Repository interface example for User entity
- GORM implementation (brief, pointing to reference)
- sqlx implementation (brief, pointing to reference)
- Transaction pattern (context-based)
- Dependency injection: handlers → services → repositories wiring in main.go

**References:**

`references/gorm-patterns.md` — Model definition with GORM tags, soft deletes, CRUD operations, scopes, preloading, raw SQL, batch operations, hooks (with trade-offs), connection pooling, error handling (gorm.ErrRecordNotFound → domain error), PostgreSQL-specific features, complete repository implementation.

`references/sqlx-patterns.md` — Connection setup, struct scanning with db tags, Get/Select/NamedExec, safe dynamic queries, transactions with sqlx.Tx, sqlx.In for IN clauses, null handling, query builder patterns, connection pooling, complete repository implementation (same interface as GORM version).

`references/migrations.md` — golang-migrate/migrate (CLI and library), file naming convention, safe migrations (zero-downtime), running on startup vs CI/CD, seeding data, rollback strategies, example migrations for users + roles.

---

### 4. `gin-testing` — Testing REST APIs

**Scope:** Unit tests, integration tests with real DB, e2e tests, test helpers, mocking.

**SKILL.md must contain (inline):**
- Testing philosophy: unit for business logic, integration for DB, e2e for critical flows
- Handler test with httptest (complete example)
- Table-driven test pattern for handlers
- Service test with mocked repository
- Test helper functions (setupTestRouter, performRequest, assertJSON)
- Running tests: `go test -v -race -cover ./...`

**References:**

`references/unit-tests.md` — Handler testing with httptest, testing JSON and error responses, testing authenticated routes, service testing with mocks, mock generation patterns, table-driven tests with subtests, test fixtures and factories, t.Helper/t.Cleanup/t.Parallel, testing middleware in isolation.

`references/integration-tests.md` — Test database setup (testcontainers-go), cleanup between tests, repository integration tests, API integration tests, build tags (//go:build integration), TestMain for setup/teardown, testing migrations, fixtures loading.

`references/e2e.md` — E2e test structure, critical flow scenarios (register → login → CRUD), testing with docker-compose, CI/CD integration (GitHub Actions), test environment configuration, cleanup and idempotency.

---

### 5. `gin-deploy` — Containerization & Deployment

**Scope:** Docker, docker-compose, CI/CD, Kubernetes basics, health checks, configuration management.

**SKILL.md must contain (inline):**
- Multi-stage Dockerfile (builder + distroless runtime)
- Health check endpoint pattern (/health with DB ping)
- Readiness vs liveness probes explanation
- Configuration via environment variables (12-factor)
- .dockerignore essentials
- docker-compose for local development (app + postgres + redis)

**References:**

`references/dockerfile.md` — Multi-stage build explained, distroless vs Alpine vs scratch, build arguments and secrets, .dockerignore, layer caching optimization, non-root user, health check instruction, image size optimization, complete production Dockerfile.

`references/docker-compose.md` — Development compose (app with Air hot reload, postgres, redis, pgadmin), service dependencies and health checks, volume mounts, environment variables, networking, integration test compose, production-like compose, complete docker-compose.yml.

`references/kubernetes.md` — Deployment manifest, Service manifest, ConfigMap and Secret, liveness/readiness probes, HPA, Ingress, PVC for database, complete manifests, brief Helm chart structure, GitHub Actions workflow for build → push → deploy.

---

## Domain Model for Examples

Use this consistent domain model across ALL skills:

```go
// User is the core domain entity used in examples
type User struct {
    ID        string    `json:"id"`
    Name      string    `json:"name"`
    Email     string    `json:"email"`
    Role      string    `json:"role"`
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at"`
}

// CreateUserRequest is the DTO for creating users
type CreateUserRequest struct {
    Name     string `json:"name" binding:"required,min=2,max=100"`
    Email    string `json:"email" binding:"required,email"`
    Password string `json:"password" binding:"required,min=8"`
    Role     string `json:"role" binding:"omitempty,oneof=admin user"`
}

// UserRepository defines the data access contract
type UserRepository interface {
    Create(ctx context.Context, user *User) error
    GetByID(ctx context.Context, id string) (*User, error)
    GetByEmail(ctx context.Context, email string) (*User, error)
    List(ctx context.Context, opts ListOptions) ([]User, int64, error)
    Update(ctx context.Context, user *User) error
    Delete(ctx context.Context, id string) error
}
```

---

## Cross-Skill References

Use this pattern: `For authentication middleware to protect these routes, see the **gin-auth** skill.`

Map:
- `gin-api` → references gin-auth (protected routes), gin-database (handler→service→repo wiring)
- `gin-auth` → references gin-api (handler patterns), gin-database (user repository)
- `gin-database` → references gin-api (dependency injection in main.go), gin-testing (repo tests)
- `gin-testing` → references gin-api (handler test examples), gin-database (integration test setup), gin-auth (testing auth middleware)
- `gin-deploy` → references gin-api (project structure and health checks), gin-database (migration running in Docker)

---

## Execution Plan

Build files in this order (dependencies flow top-down):

0. **Read `GIN-API-REFERENCE.md` first**
1. `LICENSE` (MIT)
2. `gin-api/SKILL.md`
3. `gin-api/references/error-handling.md`
4. `gin-api/references/routing.md`
5. `gin-api/references/middleware.md`
6. `gin-database/SKILL.md`
7. `gin-database/references/gorm-patterns.md`
8. `gin-database/references/sqlx-patterns.md`
9. `gin-database/references/migrations.md`
10. `gin-auth/SKILL.md`
11. `gin-auth/references/jwt-patterns.md`
12. `gin-auth/references/rbac.md`
13. `gin-testing/SKILL.md`
14. `gin-testing/references/unit-tests.md`
15. `gin-testing/references/integration-tests.md`
16. `gin-testing/references/e2e.md`
17. `gin-deploy/SKILL.md`
18. `gin-deploy/references/dockerfile.md`
19. `gin-deploy/references/docker-compose.md`
20. `gin-deploy/references/kubernetes.md`
21. `README.md` (last — references all skills)

---

## Quality Checklist

### Per SKILL.md:
- [ ] Frontmatter has `name` (lowercase, hyphens) and `description` (with trigger phrases)
- [ ] Total line count is under 500
- [ ] Starts with `# Title` and includes `## When to Use`
- [ ] All code examples compile and handle errors
- [ ] Uses `context.Context` in all blocking calls
- [ ] Uses `log/slog` (not `log` or `fmt.Println`)
- [ ] References point to correct file paths with guidance on when to load them
- [ ] No business logic in handlers (thin handler pattern)
- [ ] All Gin API calls match `GIN-API-REFERENCE.md` exactly
- [ ] Uses `ShouldBind*` methods (not `Bind*`)
- [ ] Uses `gin.New()` + explicit middleware (not `gin.Default()`) in production patterns
- [ ] Any `c.Copy()` is called before goroutine usage of `*gin.Context`

### Per reference file:
- [ ] Contains a brief header explaining what this file covers
- [ ] Code examples are complete and compilable
- [ ] Includes both the pattern and a practical example
- [ ] If over 300 lines, includes a table of contents at the top
- [ ] Explains WHY, not just HOW

### Cross-skill consistency:
- [ ] Same project structure referenced across all skills
- [ ] Same error handling pattern (`AppError`) used in all skills
- [ ] Same User/repository examples used where possible
- [ ] gin-auth JWT middleware is referenced by gin-api route examples
- [ ] gin-testing examples test the handlers/services shown in gin-api
- [ ] gin-deploy Dockerfile builds the same project structure from gin-api
- [ ] gin-database repository pattern is consumed by gin-api handlers

### Repository-level:
- [ ] README.md is complete with all sections
- [ ] LICENSE file exists (MIT)
- [ ] All 22 files exist in the correct paths
- [ ] No file exceeds 800 lines

---

## README.md Specifications

Must contain:
1. Repository title and one-line description
2. Table of all 5 skills with description and install command
3. Quick Start section (all-at-once and individual install commands)
4. Architecture section explaining progressive disclosure and composability
5. Directory tree showing full structure
6. Design principles (opinionated patterns, PostgreSQL default, production-first, Go 1.24+)
7. Contributing guidelines (brief)
8. MIT License badge/link

Install commands use placeholder: `npx skills add <your-username>/gin-skills --skill gin-api`

---

## Anti-Patterns to Avoid

1. **Monolithic SKILL.md** — violates progressive disclosure, wastes context tokens
2. **No reference files** — everything inline loads unnecessary content
3. **Generic Go patterns without framework specificity** — focus on GIN-SPECIFIC patterns
4. **Pseudo-code that doesn't compile** — every code block must be valid Go
5. **Missing error handling** — Go devs ignore examples that use `_` for errors
6. **Using `gin.Default()` in production** — always show `gin.New()` + explicit middleware
7. **Using `log.Println`** — always use `log/slog`
8. **Handlers with business logic** — always show thin handler → service → repository pattern
9. **Missing context propagation** — always pass `c.Request.Context()` to downstream calls
10. **Hardcoded configuration** — always show environment variable-based config
11. **Hallucinated API methods** — NEVER use methods not in `GIN-API-REFERENCE.md`
12. **Using `Bind*` instead of `ShouldBind*`** — Bind* prevents custom error responses
13. **Forgetting `c.Copy()` in goroutines** — causes data races
14. **Using `make(chan os.Signal)` without buffer** — must be `make(chan os.Signal, 1)`
