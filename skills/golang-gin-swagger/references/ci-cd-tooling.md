# Swagger CI/CD — Tooling and Advanced Configuration

OpenAPI 3.0 conversion, multiple swagger instances, swag init flags reference, Docker integration, and troubleshooting.

## OpenAPI 3.0 Conversion

swag v1.x generates Swagger 2.0. If OpenAPI 3.0 is required (e.g., for external API gateways), convert as a post-processing step:

```bash
# Install converter
npm install -g swagger2openapi

# Convert after swag init
swag init -g cmd/api/main.go
swagger2openapi docs/swagger.json -o docs/openapi3.json
swagger2openapi docs/swagger.yaml -o docs/openapi3.yaml
```

Add to Makefile:

```makefile
docs-openapi3: docs
	swagger2openapi docs/swagger.json -o docs/openapi3.json
	swagger2openapi docs/swagger.yaml -o docs/openapi3.yaml
```

**Note:** swag v2 (OpenAPI 3.1 native) is still RC and not production-ready. Use the conversion approach until v2 reaches stable.

## Multiple Swagger Instances

Serve separate docs for API v1 and v2 on the same server:

```bash
# Generate separate docs
swag init --instanceName v1 -g cmd/api/main.go -o ./docs/v1
swag init --instanceName v2 -g cmd/api/v2/main.go -o ./docs/v2
```

```go
import (
    _ "myapp/docs/v1"
    _ "myapp/docs/v2"
    swaggerFiles "github.com/swaggo/files"
    ginSwagger   "github.com/swaggo/gin-swagger"
)

r.GET("/swagger/v1/*any", ginSwagger.WrapHandler(
    swaggerFiles.NewHandler(),
    ginSwagger.InstanceName("v1"),
))
r.GET("/swagger/v2/*any", ginSwagger.WrapHandler(
    swaggerFiles.NewHandler(),
    ginSwagger.InstanceName("v2"),
))
```

## swag init Flags Reference

| Flag | Short | Default | Description |
|------|-------|---------|-------------|
| `--generalInfo` | `-g` | `main.go` | Go file with general API annotations |
| `--dir` | `-d` | `./` | Directories to parse (comma-separated) |
| `--exclude` | | | Paths to exclude (comma-separated) |
| `--output` | `-o` | `./docs` | Output directory |
| `--outputTypes` | `--ot` | `go,json,yaml` | File types to generate |
| `--parseDependency` | `--pd` | `false` | Parse types in dependency modules |
| `--parseDependencyLevel` | `--pdl` | `0` | Depth: 0=off, 1=models, 2=ops, 3=all |
| `--parseInternal` | | `false` | Parse `internal/` packages |
| `--parseVendor` | | `false` | Parse `vendor/` directory |
| `--propertyStrategy` | `-p` | `CamelCase` | Field naming: SnakeCase, CamelCase, PascalCase |
| `--instanceName` | | | Unique name for multiple swagger instances |
| `--tags` | `-t` | | Filter by tag (prefix `!` to exclude) |
| `--useStructName` | | `false` | Use struct name only (fixes `internal_` prefix) |
| `--requiredByDefault` | | `false` | Mark all fields required unless optional |
| `--quiet` | `-q` | `false` | Suppress logging |

### Common Combos

```bash
# Standard cmd/ layout
swag init -g cmd/api/main.go -d ./,./internal/handler,./internal/domain

# Fast — Go file only (skip JSON/YAML)
swag init -g cmd/api/main.go --outputTypes go

# Parse internal + external types
swag init -g cmd/api/main.go --parseInternal --parseDependency --parseDependencyLevel 1

# Exclude test and vendor dirs
swag init -g cmd/api/main.go --exclude ./vendor,./test

# Filter to specific tag group
swag init -g cmd/api/main.go -t users
swag init -g cmd/api/main.go -t "!internal"
```

## Docker Integration

Add `swag init` to the build stage of a multi-stage Dockerfile:

```dockerfile
FROM golang:1.24-alpine AS builder

RUN go install github.com/swaggo/swag/cmd/swag@latest

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN swag init -g cmd/api/main.go -d ./,./internal/...
RUN CGO_ENABLED=0 go build -o /app/server ./cmd/api

FROM gcr.io/distroless/static-debian12
COPY --from=builder /app/server /server
COPY --from=builder /app/docs /docs
ENTRYPOINT ["/server"]
```

```dockerfile
# Dev/staging — includes Swagger UI
RUN CGO_ENABLED=0 go build -tags swagger -o /app/server ./cmd/api

# Production — excludes Swagger UI
RUN CGO_ENABLED=0 go build -o /app/server ./cmd/api
```

## Troubleshooting CI Failures

| Symptom | Cause | Fix |
|---------|-------|-----|
| `swag: command not found` | `GOPATH/bin` not in PATH | Add `export PATH=$(go env GOPATH)/bin:$PATH` |
| `cannot find type definition` | Type in `internal/` | Add `--parseInternal` flag |
| `cannot find type definition` | Type in external dep | Add `--parseDependency --parseDependencyLevel 1` |
| `git diff --exit-code docs/` fails | Dev forgot to run `swag init` | Run `make docs` and commit |
| `docs/docs.go` has different timestamp | `--generatedTime` enabled | Remove `--generatedTime` flag |
| Slow CI step (>30s) | Parsing too many directories | Narrow `-d` to only handler/domain dirs |
| `internal_domain_User` in docs | Known bug with `--parseInternal` | Add `--useStructName` flag |
