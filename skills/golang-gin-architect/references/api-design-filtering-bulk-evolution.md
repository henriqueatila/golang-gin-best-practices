# API Design — Filtering, Bulk Operations, and Evolution

Companion to `api-design-versioning-pagination.md` (versioning, cursor/offset pagination).

---

## Filtering and Sorting

```go
// GET /api/v1/orders?status=confirmed&min_total=1000&created_after=2026-01-01
type OrderFilter struct {
    Status       string `form:"status"        binding:"omitempty,oneof=draft confirmed shipped delivered"`
    MinTotal     *int64 `form:"min_total"     binding:"omitempty,min=0"`
    MaxTotal     *int64 `form:"max_total"     binding:"omitempty,min=0"`
    CreatedAfter string `form:"created_after" binding:"omitempty,datetime=2006-01-02"`
}
```

Use pointer types for optional filters (`*int64` → nil means "not filtered"). Validate enums with `oneof`.

```go
// GET /api/v1/orders?sort=created_at&order=desc
type SortQuery struct {
    Sort  string `form:"sort"  binding:"omitempty,oneof=created_at total status"`
    Order string `form:"order" binding:"omitempty,oneof=asc desc"`
}

func (q SortQuery) OrderClause() string {
    col := q.Sort
    if col == "" { col = "created_at" }
    dir := q.Order
    if dir == "" { dir = "desc" }
    return col + " " + dir // safe — values validated by binding tags
}
```

**Security:** Always allowlist sortable columns. Never interpolate raw user input into `ORDER BY`.

---

## Bulk Operations

```go
// POST /api/v1/users/bulk
type BulkCreateRequest struct {
    Users []domain.CreateUserRequest `json:"users" binding:"required,min=1,max=100,dive"`
}

type BulkCreateResponse struct {
    Created []domain.User `json:"created"`
    Errors  []BulkError   `json:"errors,omitempty"`
}

type BulkError struct {
    Index   int    `json:"index"`
    Message string `json:"message"`
}

// DELETE /api/v1/users/bulk
type BulkDeleteRequest struct {
    IDs []string `json:"ids" binding:"required,min=1,max=100"`
}
```

Rules: cap batch size (100 default); use `dive` tag to validate each item; return partial results; use DB transactions for all-or-nothing semantics when needed.

---

## Partial Updates (PATCH)

```go
// PATCH /api/v1/users/:id
// Body: {"name": "New Name"} — only updates name, keeps other fields
type UpdateUserRequest struct {
    Name  *string `json:"name"  binding:"omitempty,min=2,max=100"`
    Email *string `json:"email" binding:"omitempty,email"`
    Role  *string `json:"role"  binding:"omitempty,oneof=admin user"`
}
```

**Key:** Use pointer types (`*string`) to distinguish "not sent" (nil) from "sent as empty" (`""`).

---

## API Evolution and Deprecation

```go
func DeprecatedEndpoint(sunset string) gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Header("Deprecation", "true")
        c.Header("Sunset", sunset) // RFC 7231 date: "Sat, 01 Jun 2026 00:00:00 GMT"
        c.Header("Link", `</api/v2/users>; rel="successor-version"`)
        c.Next()
    }
}

apiV1.GET("/users", DeprecatedEndpoint("Sat, 01 Jun 2026 00:00:00 GMT"), v1.ListUsers)
```

Migration steps: ship v2 alongside v1 → add deprecation headers to v1 → log v1 usage → communicate sunset date (minimum 3 months for external APIs) → remove v1 after sunset.

---

## Backwards Compatibility Rules

**Safe changes (no version bump):**

| Change | Why Safe |
|---|---|
| Add new response field | Clients should ignore unknown fields |
| Add new endpoint | Doesn't affect existing endpoints |
| Add new optional query parameter | Existing calls work without it |
| Widen validation | Previously valid input is still valid |

**Breaking changes (require version bump):**

| Change | Why Breaking |
|---|---|
| Remove/rename response field | Clients may depend on it |
| Change field type | Deserialization breaks |
| Narrow validation | Existing clients may break |
| Remove endpoint or change URL | Existing calls break |

> Be conservative in what you send, be liberal in what you accept.
