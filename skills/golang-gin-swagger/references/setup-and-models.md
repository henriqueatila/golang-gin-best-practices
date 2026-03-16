# Swagger Setup, Annotations, and Models

Full code examples for swaggo/swag setup, handler annotations, model documentation, and build configuration.

---

## Dependencies

```bash
# CLI tool (generates docs from annotations)
go install github.com/swaggo/swag/cmd/swag@latest

# Go module dependencies
go get -u github.com/swaggo/gin-swagger
go get -u github.com/swaggo/files
```

Ensure `$(go env GOPATH)/bin` is in your `$PATH` so the `swag` CLI is available.

---

## General API Annotations

Place these directly before `main()` in `cmd/api/main.go`. Only one annotation block per project.

```go
// @title           My API
// @version         1.0
// @description     Production-grade REST API built with Gin.

// @contact.name    API Support
// @contact.email   support@example.com

// @license.name    MIT
// @license.url     https://opensource.org/licenses/MIT

// @host            localhost:8080
// @BasePath        /api/v1
// @schemes         http https

// @securityDefinitions.apikey  BearerAuth
// @in                          header
// @name                        Authorization
// @description                 Enter: Bearer {token}
func main() { ... }
```

---

## Serving Swagger UI

```go
package main

import (
    "os"

    "github.com/gin-gonic/gin"
    swaggerFiles "github.com/swaggo/files"
    ginSwagger   "github.com/swaggo/gin-swagger"

    _ "myapp/docs" // CRITICAL: blank import registers the generated spec
)

func main() {
    r := gin.New()

    // Only expose Swagger UI outside production
    if os.Getenv("GIN_MODE") != "release" {
        r.GET("/swagger/*any", ginSwagger.WrapHandler(swaggerFiles.Handler))
    }

    // ... register routes, start server
}
```

Access at: `http://localhost:8080/swagger/index.html`

**Swagger UI options:**

```go
r.GET("/swagger/*any", ginSwagger.WrapHandler(
    swaggerFiles.Handler,
    ginSwagger.URL("/swagger/doc.json"),
    ginSwagger.DocExpansion("list"),           // "list"|"full"|"none"
    ginSwagger.DeepLinking(true),
    ginSwagger.DefaultModelsExpandDepth(1),    // -1 hides models section
    ginSwagger.PersistAuthorization(true),     // retains Bearer token across reloads
    ginSwagger.DefaultModelExpandDepth(1),    // expand depth for example section
    ginSwagger.DefaultModelRendering("example"), // "example"|"model"
))
```

---

## Dynamic Host Configuration

Override spec values at runtime for multi-environment deploys:

```go
import (
    "os"
    "myapp/docs"
)

func main() {
    docs.SwaggerInfo.Host     = os.Getenv("API_HOST") // e.g. "api.prod.example.com"
    docs.SwaggerInfo.Schemes  = []string{"https"}
    docs.SwaggerInfo.BasePath = "/api/v1"
    // ...
}
```

---

## Handler Annotations

Annotate each handler to document the endpoint. Always start with a Go doc comment.

```go
// CreateUser godoc
//
// @Summary      Create a new user
// @Description  Register a new user account
// @Tags         users
// @Accept       json
// @Produce      json
// @Param        request  body      domain.CreateUserRequest  true  "Create user payload"
// @Success      201      {object}  domain.User
// @Failure      400      {object}  domain.ErrorResponse
// @Failure      409      {object}  domain.ErrorResponse
// @Failure      500      {object}  domain.ErrorResponse
// @Router       /users [post]
func (h *UserHandler) Create(c *gin.Context) { ... }

// GetByID godoc
//
// @Summary      Get user by ID
// @Description  Retrieve a single user by UUID
// @Tags         users
// @Produce      json
// @Security     BearerAuth
// @Param        id   path      string  true  "User ID (UUID)"
// @Success      200  {object}  domain.User
// @Failure      400  {object}  domain.ErrorResponse
// @Failure      401  {object}  domain.ErrorResponse
// @Failure      404  {object}  domain.ErrorResponse
// @Failure      500  {object}  domain.ErrorResponse
// @Router       /users/{id} [get]
func (h *UserHandler) GetByID(c *gin.Context) { ... }

// List godoc
//
// @Summary      List users
// @Description  List users with pagination and optional role filter
// @Tags         users
// @Produce      json
// @Security     BearerAuth
// @Param        page   query     int     false  "Page number"      default(1) minimum(1)
// @Param        limit  query     int     false  "Items per page"   default(20) minimum(1) maximum(100)
// @Param        role   query     string  false  "Filter by role"   Enums(admin, user)
// @Success      200    {array}   domain.User
// @Failure      400    {object}  domain.ErrorResponse
// @Failure      401    {object}  domain.ErrorResponse
// @Failure      500    {object}  domain.ErrorResponse
// @Router       /users [get]
func (h *UserHandler) List(c *gin.Context) { ... }
```

---

## Model Documentation

> **Architecture note:** In clean architecture, domain entities should not carry `json` or `binding` tags. Use separate request/response DTOs in the delivery layer. See **golang-gin-clean-arch** Golden Rule 4. The tags here are shown for Swagger annotation purposes — in a clean-arch project, apply them to DTO structs, not domain entities.

Add `example`, `format`, and constraint tags to struct fields for rich Swagger docs:

```go
// internal/domain/user.go
package domain

import "time"

// User represents an authenticated system user.
type User struct {
    ID        string    `json:"id"         example:"550e8400-e29b-41d4-a716-446655440000" format:"uuid"`
    Name      string    `json:"name"       example:"Jane Doe"    minLength:"2" maxLength:"100"`
    Email     string    `json:"email"      example:"jane@example.com" format:"email"`
    Role      string    `json:"role"       example:"admin"       enums:"admin,user"`
    CreatedAt time.Time `json:"created_at" example:"2024-01-15T10:30:00Z" format:"date-time"`
    UpdatedAt time.Time `json:"updated_at" example:"2024-01-15T10:30:00Z" format:"date-time"`

    PasswordHash string `json:"-" swaggerignore:"true"`
}

// CreateUserRequest is the payload for POST /users.
type CreateUserRequest struct {
    Name     string `json:"name"     example:"Jane Doe"        binding:"required,min=2,max=100"`
    Email    string `json:"email"    example:"jane@example.com" binding:"required,email"`
    Password string `json:"password" example:"s3cur3P@ss!"      binding:"required,min=8"`
    Role     string `json:"role"     example:"user"             binding:"omitempty,oneof=admin user" enums:"admin,user"`
}

// ErrorResponse is the standard API error envelope.
type ErrorResponse struct {
    Error string `json:"error" example:"resource not found"`
}
```

---

## Generating Docs

```bash
# Standard cmd/ layout
swag init -g cmd/api/main.go -d ./,./internal/handler,./internal/domain

# Format annotations first (recommended)
swag fmt && swag init -g cmd/api/main.go

# Parse types from internal/ packages
swag init -g cmd/api/main.go --parseInternal

# Output: docs/docs.go, docs/swagger.json, docs/swagger.yaml
```

**Commit the generated `docs/` directory.** Re-run `swag init` after every handler or model change.

---

## Makefile Integration

```makefile
.PHONY: docs docs-check

docs:
	swag fmt
	swag init -g cmd/api/main.go -d ./,./internal/... --exclude ./vendor

docs-check:
	swag init -g cmd/api/main.go -d ./,./internal/... --exclude ./vendor
	git diff --exit-code docs/
```

---

## Excluding Swagger from Production Binary

Use build tags to strip Swagger UI from production builds entirely:

```go
// file: swagger.go
//go:build swagger

package main

import (
    _ "myapp/docs"
    ginSwagger   "github.com/swaggo/gin-swagger"
    swaggerFiles "github.com/swaggo/files"
    "github.com/gin-gonic/gin"
)

func registerSwagger(r *gin.Engine) {
    r.GET("/swagger/*any", ginSwagger.WrapHandler(swaggerFiles.Handler))
}
```

```go
// file: swagger_noop.go
//go:build !swagger

package main

import "github.com/gin-gonic/gin"

func registerSwagger(r *gin.Engine) {} // no-op in production
```

Build with `go build -tags swagger .` for dev/staging, `go build .` for production.
