# Swagger Setup — Dependencies, Serving UI, and Handler Annotations

Installation, serving Swagger UI in Gin, dynamic host configuration, and core handler annotation examples.

## Dependencies

```bash
# CLI tool (generates docs from annotations)
go install github.com/swaggo/swag/cmd/swag@latest

# Go module dependencies
go get -u github.com/swaggo/gin-swagger
go get -u github.com/swaggo/files
```

Ensure `$(go env GOPATH)/bin` is in your `$PATH` so the `swag` CLI is available.

## General API Annotations

Place directly before `main()` in `cmd/api/main.go`. Only one annotation block per project.

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
    ginSwagger.DocExpansion("list"),              // "list"|"full"|"none"
    ginSwagger.DeepLinking(true),
    ginSwagger.DefaultModelsExpandDepth(1),       // -1 hides models section
    ginSwagger.PersistAuthorization(true),        // retains Bearer token across reloads
    ginSwagger.DefaultModelExpandDepth(1),
    ginSwagger.DefaultModelRendering("example"),  // "example"|"model"
))
```

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

For handler annotation examples (Create, GetByID, List) and model struct tags: see [setup-models-generate.md](setup-models-generate.md).
