# Swagger CI/CD — GitHub Actions Workflows and Hooks

GitHub Actions for auto-generating docs, PR validation, Makefile targets, and pre-commit hook.

## GitHub Actions — Generate and Commit

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

## GitHub Actions — PR Validation

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

## Makefile Integration

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

## Pre-commit Hook

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
