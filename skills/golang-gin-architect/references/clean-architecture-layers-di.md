# Clean Architecture — Layer Mapping, Dependency Rule, and DI

Practical guide to applying Clean Architecture in Go Gin APIs.

**Default stance:** Start with flat package layout. Adopt clean architecture layering when the codebase outgrows it — not before.

---

## Layer Mapping: Uncle Bob → Go

```
┌─────────────────────────────────────────────────────┐
│  cmd/api/main.go          ← Wiring (imports all)    │
│  ┌─────────────────────────────────────────────┐    │
│  │  handler/              ← Driving Adapters    │    │
│  │  ┌─────────────────────────────────────┐    │    │
│  │  │  usecase/ or service/               │    │    │
│  │  │  ┌─────────────────────────────┐    │    │    │
│  │  │  │  domain/  (stdlib only)     │    │    │    │
│  │  │  └─────────────────────────────┘    │    │    │
│  │  └─────────────────────────────────────┘    │    │
│  └─────────────────────────────────────────────┘    │
│  repository/              ← Driven Adapters          │
└─────────────────────────────────────────────────────┘
```

| Uncle Bob Layer | Go Package | Imports | Contains |
|---|---|---|---|
| Entities | `domain/` | stdlib only | Structs, value objects, domain errors |
| Use Cases | `usecase/` or `service/` | `domain/` only | Business logic, port interfaces |
| Interface Adapters | `handler/`, `repository/` | `usecase/`, `domain/` | Gin handlers, sqlx repos |
| Frameworks & Drivers | `cmd/api/main.go` | everything | Wiring, config, server startup |

**Go-specific twist:** Interfaces are defined in the **consuming** package (`usecase/`), not the implementing package (`repository/`). This is idiomatic Go and structurally enforces the dependency rule.

---

## The Dependency Rule

> Inner layers NEVER import outer layers. Dependencies point inward.

```
domain/    → imports: nothing (stdlib only)
usecase/   → imports: domain/
handler/   → imports: usecase/, domain/
repository/→ imports: usecase/ (for interface), domain/ (for entities)
cmd/       → imports: everything (wiring only)
```

**Compile-time interface check** — always add in the adapter package:

```go
// repository/postgres/user_repository.go
var _ usecase.UserRepository = (*UserRepository)(nil)
```

Fails at compile time if `UserRepository` doesn't satisfy the interface. Zero runtime cost.

---

## Ports & Adapters

A **port** is a Go interface. An **adapter** is an implementation.

Port (defined in usecase package):

```go
// internal/user/usecase/ports.go
type UserRepository interface {
    GetByID(ctx context.Context, id int64) (*domain.User, error)
    Create(ctx context.Context, user *domain.User) error
    GetByEmail(ctx context.Context, email string) (*domain.User, error)
}
```

Driven adapter (sqlx implementation):

```go
var _ usecase.UserRepository = (*UserRepository)(nil)

func (r *UserRepository) GetByID(ctx context.Context, id int64) (*domain.User, error) {
    var u domain.User
    err := r.db.GetContext(ctx, &u,
        `SELECT id, email, name, created_at FROM users WHERE id = $1`, id)
    if err != nil {
        return nil, fmt.Errorf("UserRepository.GetByID: %w", err)
    }
    return &u, nil
}
```

Driving adapter (Gin handler):

```go
func (h *UserHandler) GetByID(c *gin.Context) {
    id, err := strconv.ParseInt(c.Param("id"), 10, 64)
    if err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "invalid user id"})
        return
    }
    user, err := h.svc.GetByID(c.Request.Context(), id)
    if err != nil {
        c.JSON(http.StatusNotFound, gin.H{"error": "user not found"})
        return
    }
    c.JSON(http.StatusOK, user)
}
```

---

## Wiring with Manual DI

**Manual DI in `main.go`** is the Go-idiomatic choice. Wire/fx only justifies itself at 20+ services.

```go
func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

    db, err := sqlx.Connect("postgres", os.Getenv("DATABASE_URL"))
    if err != nil {
        logger.Error("database connection failed", "error", err)
        os.Exit(1)
    }

    // Wire: repo → service → handler
    userRepo := userPostgres.NewUserRepository(db)
    userSvc  := userUsecase.NewUserService(userRepo, logger)
    userH    := userHandler.NewUserHandler(userSvc)

    r := gin.New()
    r.Use(gin.Recovery())

    api := r.Group("/api/v1")
    api.GET("/users/:id", userH.GetByID)

    r.Run(":8080")
}
```

**The wiring rule:** `main.go` is the only file that knows about ALL packages.

---

## See Also

- `clean-architecture-feature-module.md` — complete feature module example, common mistakes, when to use what
