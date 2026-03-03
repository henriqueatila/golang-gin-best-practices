# ci-cd.md — CI/CD, Tooling & Advanced Configuration

GitHub Actions workflows, PR validation, pre-commit hooks, OpenAPI 3.0 conversion, multiple swagger instances, and the complete `swag init` flags reference.

## Table of Contents

1. [GitHub Actions — Generate & Commit](#1-github-actions--generate--commit)
2. [GitHub Actions — PR Validation](#2-github-actions--pr-validation)
3. [Makefile Integration](#3-makefile-integration)
4. [Pre-commit Hook](#4-pre-commit-hook)
5. [OpenAPI 3.0 Conversion](#5-openapi-30-conversion)
6. [Multiple Swagger Instances](#6-multiple-swagger-instances)
7. [swag init Flags Reference](#7-swag-init-flags-reference)
8. [Docker Integration](#8-docker-integration)
9. [Troubleshooting CI Failures](#9-troubleshooting-ci-failures)

---

## 1. GitHub Actions — Generate & Commit

Auto-generate and commit docs on push to `main`:

```yaml
name: swagger-docs

on:
  push:
    branches: [main]
    paths: ['**/*.go']

jobs:
  generate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}

      - uses: actions/setup-go@v5
        with:
          go-version: '1.24'
          cache: true

      - name: Install swag
        run: go install github.com/swaggo/swag/cmd/swag@latest

      - name: Generate docs
        run: swag init -g cmd/api/main.go -d ./,./internal/... --exclude ./vendor

      - name: Commit if changed
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add docs/
          git diff --staged --quiet || git commit -m "docs: regenerate swagger"
          git push
```

---

## 2. GitHub Actions — PR Validation

Fail the PR if swagger docs are stale:

```yaml
name: swagger-check

on:
  pull_request:
    paths: ['**/*.go']

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version: '1.24'
          cache: true

      - name: Install swag
        run: go install github.com/swaggo/swag/cmd/swag@latest

      - name: Check docs are up to date
        run: |
          swag init -g cmd/api/main.go -d ./,./internal/... --exclude ./vendor
          git diff --exit-code docs/ || {
            echo "::error::Swagger docs are stale. Run 'make docs' and commit the changes."
            exit 1
          }
```

---

## 3. Makefile Integration

```makefile
.PHONY: docs docs-check docs-serve

# Generate swagger docs
docs:
	swag fmt
	swag init -g cmd/api/main.go -d ./,./internal/... --exclude ./vendor

# Validate docs are committed (used in CI)
docs-check:
	swag init -g cmd/api/main.go -d ./,./internal/... --exclude ./vendor
	git diff --exit-code docs/

# Serve swagger UI locally (requires the app to be running)
docs-serve:
	@echo "Swagger UI: http://localhost:8080/swagger/index.html"
```

---

## 4. Pre-commit Hook

Auto-regenerate docs before each commit:

```bash
#!/bin/sh
# .git/hooks/pre-commit

# Check if any Go files changed
STAGED_GO=$(git diff --cached --name-only --diff-filter=ACM | grep '\.go$')
if [ -z "$STAGED_GO" ]; then
    exit 0
fi

# Regenerate docs
swag fmt
swag init -g cmd/api/main.go -d ./,./internal/... --exclude ./vendor

# Stage updated docs
git add docs/
```

Make executable: `chmod +x .git/hooks/pre-commit`

---

## 5. OpenAPI 3.0 Conversion

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

---

## 6. Multiple Swagger Instances

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

// Serve each version at a different path
r.GET("/swagger/v1/*any", ginSwagger.WrapHandler(
    swaggerFiles.NewHandler(),
    ginSwagger.InstanceName("v1"),
))
r.GET("/swagger/v2/*any", ginSwagger.WrapHandler(
    swaggerFiles.NewHandler(),
    ginSwagger.InstanceName("v2"),
))
```

---

## 7. swag init Flags Reference

```
swag init [flags]
```

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
| `--parseGoList` | | `true` | Resolve deps via `go list` |
| `--propertyStrategy` | `-p` | `CamelCase` | Field naming: SnakeCase, CamelCase, PascalCase |
| `--instanceName` | | | Unique name for multiple swagger instances |
| `--tags` | `-t` | | Filter by tag (prefix `!` to exclude) |
| `--markdownFiles` | `--md` | | Folder with markdown for `@Description` |
| `--packageName` | | | Override `docs.go` package name |
| `--requiredByDefault` | | `false` | Mark all fields required unless optional |
| `--quiet` | `-q` | `false` | Suppress logging |
| `--generatedTime` | | `false` | Add timestamp to `docs.go` |
| `--overridesFile` | | `.swaggo` | File for global type overrides |
| `--collectionFormat` | `--cf` | `csv` | Default array collection format |
| `--useStructName` | | `false` | Use struct name only (fixes `internal_` prefix) |

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
swag init -g cmd/api/main.go -t "!internal"  # exclude internal tag
```

---

## 8. Docker Integration

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

For production builds without Swagger UI (using build tags):

```dockerfile
# Dev/staging — includes Swagger UI
RUN CGO_ENABLED=0 go build -tags swagger -o /app/server ./cmd/api

# Production — excludes Swagger UI
RUN CGO_ENABLED=0 go build -o /app/server ./cmd/api
```

---

## 9. Troubleshooting CI Failures

| Symptom | Cause | Fix |
|---------|-------|-----|
| `swag: command not found` | `GOPATH/bin` not in PATH | Add `export PATH=$(go env GOPATH)/bin:$PATH` |
| `cannot find type definition` | Type in `internal/` | Add `--parseInternal` flag |
| `cannot find type definition` | Type in external dep | Add `--parseDependency --parseDependencyLevel 1` |
| `git diff --exit-code docs/` fails | Dev forgot to run `swag init` | Run `make docs` and commit |
| `docs/docs.go` has different timestamp | `--generatedTime` enabled | Remove `--generatedTime` flag |
| Slow CI step (>30s) | Parsing too many directories | Narrow `-d` to only handler/domain dirs |
| `internal_domain_User` in docs | Known bug with `--parseInternal` | Add `--useStructName` flag |
