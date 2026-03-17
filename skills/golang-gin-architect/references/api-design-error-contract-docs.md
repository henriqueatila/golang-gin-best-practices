# API Design — Error Response Contract and Documentation

Companion to `api-design-filtering-bulk-evolution.md` (filtering, bulk ops, deprecation).

---

## Error Response Contract

```go
type ErrorResponse struct {
    Error   string            `json:"error"`             // human-readable
    Code    string            `json:"code,omitempty"`    // machine-readable: "USER_NOT_FOUND"
    Details map[string]string `json:"details,omitempty"` // field-level errors
}
```

Validation error (422):
```json
{
    "error": "validation failed",
    "code": "VALIDATION_ERROR",
    "details": {
        "email": "invalid email format",
        "name": "must be at least 2 characters"
    }
}
```

**Standard HTTP status mapping:**

| Situation | Status |
|---|---|
| Invalid input / binding failure | 400 Bad Request |
| Missing or invalid auth token | 401 Unauthorized |
| Authenticated but insufficient permissions | 403 Forbidden |
| Resource not found | 404 Not Found |
| Business rule violation (validation) | 422 Unprocessable Entity |
| Unexpected server error | 500 Internal Server Error |

---

## API Documentation (swaggo)

```go
// @title           My API
// @version         1.0
// @BasePath        /api/v1
// @securityDefinitions.apikey BearerAuth
// @in header
// @name Authorization

// @Summary      Create user
// @Tags         users
// @Accept       json
// @Produce      json
// @Param        body body domain.CreateUserRequest true "User data"
// @Success      201  {object} domain.User
// @Failure      400  {object} ErrorResponse
// @Router       /users [post]
func (h *UserHandler) Create(c *gin.Context) { ... }
```

Generate: `swag init -g cmd/api/main.go`. Serve: `github.com/swaggo/gin-swagger` at `/swagger/*`.

**HATEOAS:** You probably don't need this. Document endpoints in OpenAPI instead.
