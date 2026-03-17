# Clean Architecture — Feature Module Example and Decision Guide

Companion to `clean-architecture-layers-di.md` (layer mapping, dependency rule, ports & adapters, DI wiring).

---

## Complete Feature Module Layout

For medium+ projects using feature modules (recommended default):

```
internal/
├── user/
│   ├── domain/
│   │   ├── user.go           # Entity: User struct, validation
│   │   └── errors.go         # Domain errors: ErrUserNotFound, ErrDuplicateEmail
│   ├── usecase/
│   │   ├── ports.go          # Interface: UserRepository
│   │   └── user_service.go   # Business logic
│   ├── handler/
│   │   └── user_handler.go   # Gin HTTP handlers
│   └── repository/
│       └── postgres/
│           └── user_repository.go
├── order/
│   ├── domain/
│   ├── usecase/
│   ├── handler/
│   └── repository/
└── platform/                  # Shared infrastructure (not a domain)
    ├── database/
    ├── middleware/
    └── config/
```

---

## Domain Layer

```go
// internal/user/domain/user.go
type User struct {
    ID           int64     `db:"id" json:"id"`
    Email        string    `db:"email" json:"email"`
    Name         string    `db:"name" json:"name"`
    PasswordHash string    `db:"password_hash" json:"-"`
    CreatedAt    time.Time `db:"created_at" json:"created_at"`
}

// internal/user/domain/errors.go
var (
    ErrUserNotFound   = errors.New("user not found")
    ErrDuplicateEmail = errors.New("email already registered")
)
```

---

## Use Case Layer

```go
// internal/user/usecase/user_service.go
type UserService struct {
    repo   UserRepository
    logger *slog.Logger
}

func NewUserService(repo UserRepository, logger *slog.Logger) *UserService {
    return &UserService{
        repo:   repo,
        logger: logger.With("component", "user-service"),
    }
}

func (s *UserService) GetByID(ctx context.Context, id int64) (*domain.User, error) {
    user, err := s.repo.GetByID(ctx, id)
    if err != nil {
        return nil, fmt.Errorf("UserService.GetByID: %w", err)
    }
    return user, nil
}
```

---

## Common Mistakes

| Mistake | Why it's wrong | Fix |
|---|---|---|
| Interface next to implementation | Couples consumer to provider | Interface in `usecase/`, implementation in `repository/` |
| Domain type as JSON response | Leaks DB fields, breaks API contract | Use response DTOs in `handler/` |
| Business logic in handlers | Untestable without HTTP, violates SRP | Handlers: bind → call service → respond |
| Interface for everything | Over-abstraction | Only where you need to swap implementations |
| Clean arch for a 5-endpoint MVP | Over-engineering, YAGNI | Start flat, refactor when it hurts |
| Shared DB between services | Hidden coupling | Each service owns its data |
| `domain/` importing `gin` or `sqlx` | Violates dependency rule | Domain is stdlib-only |

---

## When to Use What

| Project Size | Recommended Structure | Clean Arch Level |
|---|---|---|
| MVP, < 10 endpoints, 1-2 devs | Flat: `handler/`, `service/`, `repository/`, `domain/` | Implicit layers only |
| Growing, 10-50 endpoints, 3-8 devs | Feature modules: `internal/user/`, `internal/order/` | Per-module ports in `usecase/` |
| Complex domain, 50+ endpoints, 8+ devs | Full clean arch with bounded contexts | Strict dependency rule, DTOs at boundaries |

**Rule of thumb:** Start flat. When you feel pain (circular deps, fat handlers, tests requiring real DB), introduce the next level. The moment you add a second adapter for an interface (in-memory repo for tests + postgres for prod), clean architecture has paid for itself.

### Gate — you need formal clean architecture when

- [ ] You have 2+ delivery mechanisms (HTTP + gRPC + CLI) calling the same business logic
- [ ] You need to swap infrastructure without touching business logic
- [ ] Multiple teams work on the same service and need clear ownership boundaries
- [ ] Test setup requires complex mocking because business logic is tangled with HTTP/DB

If none apply, flat layout with implicit layering (handler → service → repository) is clean architecture in spirit — without the ceremony.

---

## Cross-Skill References

- Project structure at different scales: `system-design-project-structure.md`
- Complexity budget (is this overkill?): `complexity-assessment-budget.md`
- Repository and ORM patterns: `golang-gin-database` skill
- Handler and middleware implementation: `golang-gin-api` skill
- Testing layers in isolation: `golang-gin-testing` skill
