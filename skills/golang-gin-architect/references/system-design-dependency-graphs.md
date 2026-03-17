# System Design — Dependency Graphs and Dependency Inversion

No-cycles rule, dependency inversion with interfaces, and main.go wiring.

Companion to `system-design-project-structure.md` (package layout at scale).

---

## The No-Cycles Rule

Go enforces this at the compiler level: circular imports are compile errors. But logical cycles through interfaces are harder to detect.

```
Good dependency direction (arrows show "depends on"):
Handler → Service → Repository → Database
   ↓          ↓           ↓
Domain ←──────────────────────  (domain has no outward deps)
```

```
Bad: Service imports Handler package
Bad: Repository imports Service package
Bad: Domain imports anything from internal/
```

**Rule:** Dependencies flow inward. The domain layer (entities, interfaces, errors) depends on nothing.

---

## Dependency Inversion with Interfaces

```go
// Bad — hard dependency on concrete type
import "myapp/internal/user"
type OrderService struct {
    userSvc *user.Service
}

// Good — order defines what it needs, user satisfies it
// internal/order/ports.go
type UserLookup interface {
    GetByID(ctx context.Context, id string) (UserInfo, error)
}
// internal/order/service.go
type Service struct {
    users UserLookup // interface — testable, no import of user package
}
```

---

## Practical main.go Wiring

`main.go` is the only place that knows about concrete types.

```go
func main() {
    db := platform.NewDB(cfg.DatabaseURL)
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

    userRepo  := user.NewPostgresRepository(db)
    orderRepo := order.NewPostgresRepository(db)

    userSvc  := user.NewService(userRepo, logger)
    orderSvc := order.NewService(orderRepo, userSvc, logger) // userSvc satisfies order.UserLookup

    userHandler  := user.NewHandler(userSvc, logger)
    orderHandler := order.NewHandler(orderSvc, logger)

    r := setupRouter(userHandler, orderHandler)
}
```

---

## Visualizing Your Dependency Graph

```bash
# List what each package imports
go list -f '{{.ImportPath}}: {{join .Imports ", "}}' ./internal/...
```

Red flags: `internal/domain` importing anything from `internal/`, any `internal/X` importing `internal/handler` or `cmd/` packages.

---

## Cross-Skill References

- For package layout at scale: see **[system-design-project-structure.md](system-design-project-structure.md)**
- For bounded context analysis: see **[system-design-bounded-contexts.md](system-design-bounded-contexts.md)**
- For C4 diagrams: see **[system-design-c4-model.md](system-design-c4-model.md)**
