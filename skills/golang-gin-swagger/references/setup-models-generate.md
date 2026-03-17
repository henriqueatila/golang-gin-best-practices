# Swagger Setup — Model Documentation, Doc Generation, and Production Builds

Model struct tags, generating docs with swag init, Makefile integration, and excluding Swagger from production binaries.

> **Architecture note:** In clean architecture, domain entities should not carry `json` or `binding` tags. Use separate request/response DTOs in the delivery layer. See **golang-gin-architect skill (`references/clean-architecture.md`)** Golden Rule 4. The tags here are shown for Swagger annotation purposes — apply them to DTO structs, not domain entities.

## Model Documentation

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
