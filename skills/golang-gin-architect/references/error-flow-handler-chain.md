# Error Flow Architecture — Handler Mapping and Complete Chain

Companion to `error-flow-domain-layers.md` (domain errors, repository, service layer rules).

---

## Handler Layer — Map to HTTP

Handlers map domain errors to HTTP responses. This is the ONLY layer that knows about HTTP status codes.

```go
func (h *UserHandler) GetUser(c *gin.Context) {
    id, err := strconv.ParseInt(c.Param("id"), 10, 64)
    if err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "invalid id"})
        return
    }

    user, err := h.svc.GetByID(c.Request.Context(), id)
    if err != nil {
        status, msg := mapDomainError(err)
        if status == http.StatusInternalServerError {
            slog.ErrorContext(c.Request.Context(), "unexpected error",
                "op", "GetUser", "id", id, "err", err)
        }
        c.JSON(status, gin.H{"error": msg})
        return
    }
    c.JSON(http.StatusOK, user)
}

// mapDomainError converts domain errors to HTTP status + message pairs.
func mapDomainError(err error) (int, string) {
    var valErr *domain.ValidationError
    if errors.As(err, &valErr) {
        return http.StatusBadRequest, valErr.Error()
    }
    var multiErr *domain.MultiValidationError
    if errors.As(err, &multiErr) {
        return http.StatusBadRequest, multiErr.Error()
    }
    switch {
    case errors.Is(err, domain.ErrNotFound):
        return http.StatusNotFound, "resource not found"
    case errors.Is(err, domain.ErrAlreadyExists):
        return http.StatusConflict, "resource already exists"
    case errors.Is(err, domain.ErrInvalidInput):
        return http.StatusBadRequest, "invalid input"
    case errors.Is(err, domain.ErrForbidden):
        return http.StatusForbidden, "access denied"
    case errors.Is(err, domain.ErrUnauthorized):
        return http.StatusUnauthorized, "unauthorized"
    default:
        return http.StatusInternalServerError, "internal error"
    }
}
```

**Handler rules:** NEVER wrap errors — only map them. Log only at handler level. Log only 5xx; 4xx are normal flow. `mapDomainError` is a pure function — easy to test independently.

---

## Error Wrapping Convention

| Layer      | Convention                                     | Purpose                   |
|------------|------------------------------------------------|---------------------------|
| Domain     | `errors.New("description")`                    | Define sentinel/root errors |
| Repository | `fmt.Errorf("Repo.Method(args): %w", err)`     | Translate + add call site |
| Service    | `fmt.Errorf("Service.Method(args): %w", err)`  | Add operation context     |
| Handler    | No wrapping — call `mapDomainError(err)`        | Translate to HTTP         |

**Always use `%w`, never `%v`.** `%v` stringifies — `errors.Is` and `errors.As` cannot unwrap it.

```go
// CORRECT — chain preserved
return fmt.Errorf("UserService.GetByID(%d): %w", id, err)

// WRONG — chain broken, errors.Is will not find domain.ErrNotFound
return fmt.Errorf("UserService.GetByID(%d): %v", id, err)
```

---

## errors.Is and errors.As Usage

```go
// errors.Is — checks identity/equality through the chain
errors.Is(err, domain.ErrNotFound) // true even if err is 3 levels deep

// errors.As — extracts the first matching type from the chain
var valErr *domain.ValidationError
if errors.As(err, &valErr) {
    slog.Warn("validation failed", "field", valErr.Field, "msg", valErr.Message)
}

// Check specific types before broad sentinels
var ve *domain.ValidationError
switch {
case errors.As(err, &ve):
    return http.StatusBadRequest, ve.Message
case errors.Is(err, domain.ErrNotFound):
    return http.StatusNotFound, "not found"
}
```

**Pitfall:** never unwrap manually with `.(type)` assertions — they do not traverse wrapped chains.

---

## Complete Error Chain Example

```go
// Step 1 — Database
err = sql.ErrNoRows

// Step 2 — Repository translates
return nil, fmt.Errorf("postgresRepo.GetByID(42): %w", domain.ErrNotFound)

// Step 3 — Service adds context
return nil, fmt.Errorf("UserService.GetByID(42): %w", err)
// domain.ErrNotFound still reachable via errors.Is

// Step 4 — Handler maps to HTTP
status, msg := mapDomainError(err)
// errors.Is(err, domain.ErrNotFound) → true → status=404
c.JSON(http.StatusNotFound, gin.H{"error": "resource not found"})
// 404 is expected — no log emitted
```

Full error string in logs (5xx path):
```
UserService.GetByID(42): postgresRepo.GetByID(42): not found
```

---

## Summary of Responsibilities

```
domain/errors.go     → define error types (no deps)
repository/*.go      → translate DB errors → domain errors, wrap with context
service/*.go         → wrap errors with operation context, propagate
handler/error_map.go → single mapDomainError function, log 5xx, call c.JSON
```
