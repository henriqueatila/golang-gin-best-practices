# Error Flow Architecture — Domain Errors and Layer Rules

How errors flow through clean architecture layers in Go Gin APIs. Covers domain error types, repository and service layer conventions.

**Companion to:** `error-flow-handler-chain.md` (handler mapping, wrapping conventions, complete chain example).

---

## Error Flow Diagram

```
┌──────────────────────────────────────────────────────────────┐
│                    Error Flow (outermost → DB)               │
│  HTTP Request                                                │
│       │                                                      │
│       ▼                                                      │
│  [Handler]  ─── errors.Is / errors.As ──► HTTP status+body  │
│       │  returns domain.ErrNotFound (wrapped)               │
│       ▼                                                      │
│  [Service]  ─── fmt.Errorf("Svc.Method: %w", err)           │
│       │  returns domain.ErrNotFound (wrapped)               │
│       ▼                                                      │
│  [Repository] ─── translates sql.ErrNoRows → domain.Err*    │
│       ▼                                                      │
│  [Database]   ─── sql.ErrNoRows, *pq.Error, etc.            │
│  chain: sql.ErrNoRows → domain.ErrNotFound → "Svc: %w" → 404│
└──────────────────────────────────────────────────────────────┘
```

**Key rule:** Only the handler maps errors to HTTP. Layers below NEVER call `c.JSON` or reference HTTP status codes.

---

## Domain Errors — Innermost Layer

The domain layer owns error definitions. No external dependencies — stdlib only.

```go
// internal/user/domain/errors.go
package domain

import "errors"

// Sentinel errors — checked with errors.Is
var (
    ErrNotFound      = errors.New("not found")
    ErrAlreadyExists = errors.New("already exists")
    ErrInvalidInput  = errors.New("invalid input")
    ErrForbidden     = errors.New("forbidden")
    ErrUnauthorized  = errors.New("unauthorized")
)

// ValidationError carries field-level detail — extracted with errors.As
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string { return e.Field + ": " + e.Message }

// MultiValidationError aggregates field errors
type MultiValidationError struct {
    Errors []*ValidationError
}

func (e *MultiValidationError) Error() string {
    return "validation failed: " + strconv.Itoa(len(e.Errors)) + " error(s)"
}
```

Rules: defined once in domain package; pure stdlib; sentinel errors for simple checks; typed errors for structured data; no HTTP awareness.

---

## Repository Layer — Translate DB Errors

The repository is the ONLY layer that imports `database/sql`, `github.com/lib/pq`, or ORM packages.

```go
func (r *postgresRepo) GetByID(ctx context.Context, id int64) (*domain.User, error) {
    var u domain.User
    err := r.db.QueryRowContext(ctx,
        `SELECT id, email, name FROM users WHERE id = $1`, id,
    ).Scan(&u.ID, &u.Email, &u.Name)
    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, fmt.Errorf("postgresRepo.GetByID(%d): %w", id, domain.ErrNotFound)
        }
        return nil, fmt.Errorf("postgresRepo.GetByID(%d): %w", id, err)
    }
    return &u, nil
}

func (r *postgresRepo) Create(ctx context.Context, u *domain.User) error {
    _, err := r.db.ExecContext(ctx,
        `INSERT INTO users (email, name) VALUES ($1, $2)`, u.Email, u.Name,
    )
    if err != nil {
        var pqErr *pq.Error
        if errors.As(err, &pqErr) && pqErr.Code == "23505" {
            return fmt.Errorf("postgresRepo.Create: %w", domain.ErrAlreadyExists)
        }
        return fmt.Errorf("postgresRepo.Create: %w", err)
    }
    return nil
}
```

**Repository rules:** ALWAYS translate known DB errors to domain errors; wrap with `%w`; NEVER return raw `sql.ErrNoRows` or `*pq.Error`; NEVER log.

---

## Service Layer — Wrap with Context

Services implement business logic. They receive domain errors from repositories, add context, and propagate upward. NEVER handle HTTP.

```go
func (s *UserService) GetByID(ctx context.Context, id int64) (*domain.User, error) {
    user, err := s.repo.GetByID(ctx, id)
    if err != nil {
        return nil, fmt.Errorf("UserService.GetByID(%d): %w", id, err)
    }
    return user, nil
}

func (s *UserService) Register(ctx context.Context, req domain.RegisterRequest) (*domain.User, error) {
    if req.Password != req.PasswordConfirm {
        return nil, fmt.Errorf("UserService.Register: %w",
            &domain.ValidationError{Field: "password_confirm", Message: "passwords do not match"},
        )
    }
    user := &domain.User{Email: req.Email, Name: req.Name}
    if err := s.repo.Create(ctx, user); err != nil {
        return nil, fmt.Errorf("UserService.Register: %w", err)
    }
    return user, nil
}
```

**Service rules:**
- ALWAYS use `%w` — never `%v` — so errors.Is/As work through the chain
- Add meaningful context: operation name, key parameter values
- NEVER call `c.JSON`, reference `net/http` status codes, or log
