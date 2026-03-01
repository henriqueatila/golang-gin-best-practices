# Docker Compose Reference

This file covers docker-compose configurations for Go Gin APIs: development setup with Air hot reload, service health checks, volume mounts, environment variables, networking, integration test compose, and a production-like compose. Use when setting up or customizing local development or CI environments.

> **Architectural recommendation:** docker-compose patterns are not part of the Gin framework. These are mainstream Go community patterns.

## Table of Contents

1. [Development Compose with Air Hot Reload](#development-compose-with-air-hot-reload)
2. [Service Dependencies and Health Checks](#service-dependencies-and-health-checks)
3. [Volume Mounts](#volume-mounts)
4. [Environment Variables](#environment-variables)
5. [Networking](#networking)
6. [Integration Test Compose](#integration-test-compose)
7. [Production-Like Compose](#production-like-compose)
8. [Complete docker-compose.yml](#complete-docker-composeyml)

---

## Development Compose with Air Hot Reload

[Air](https://github.com/air-verse/air) watches Go source files and rebuilds the binary on change — eliminating the manual stop/rebuild/start cycle during development.

**Install Air in the dev container:**

```dockerfile
# Dockerfile.dev — development-only image
FROM golang:1.24-bookworm

WORKDIR /app

# Install Air for hot reload
RUN go install github.com/air-verse/air@latest

# Install migrate CLI for running migrations manually
RUN go install -tags 'postgres' github.com/golang-migrate/migrate/v4/cmd/migrate@latest

COPY go.mod go.sum ./
RUN go mod download

# Source is mounted as a volume — no COPY needed
EXPOSE 8080

CMD ["air", "-c", ".air.toml"]
```

**Air configuration `.air.toml`:**

```toml
root = "."
tmp_dir = "tmp"

[build]
  bin = "./tmp/main"
  cmd = "go build -o ./tmp/main ./cmd/api"
  delay = 500
  exclude_dir = ["tmp", "vendor", "testdata", "plans", "docs"]
  exclude_regex = ["_test.go", "_mock.go"]
  include_ext = ["go", "html", "toml", "env"]
  kill_delay = "0s"
  send_interrupt = true

[log]
  time = true

[color]
  build = "yellow"
  runner = "green"
  watcher = "cyan"
```

---

## Service Dependencies and Health Checks

`depends_on` with `condition: service_healthy` prevents the app from starting before its dependencies are ready. Without health checks, Docker only waits for the container to start — not for the service inside to be ready to accept connections.

```yaml
services:
  app:
    depends_on:
      postgres:
        condition: service_healthy   # waits for postgres healthcheck to pass
      redis:
        condition: service_healthy   # waits for redis healthcheck to pass

  postgres:
    image: postgres:17-alpine
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 5s       # check every 5s
      timeout: 5s        # fail if no response in 5s
      retries: 5         # mark unhealthy after 5 failures
      start_period: 10s  # grace period before checks begin

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
```

**Why `start_period` matters:** PostgreSQL initialization (first run) can take 5-15 seconds. `start_period` gives it time to initialize before failures count against `retries`.

---

## Volume Mounts

```yaml
services:
  app:
    volumes:
      # Source mount for hot reload (dev only — remove in production compose)
      - .:/app
      # Exclude the local binary artifacts from the mount
      - /app/tmp

  postgres:
    volumes:
      # Named volume — data persists across `docker-compose down`
      - postgres_data:/var/lib/postgresql/data
      # Optional: mount init scripts run on first start
      - ./db/init:/docker-entrypoint-initdb.d:ro

  pgadmin:
    volumes:
      - pgadmin_data:/var/lib/pgadmin

volumes:
  postgres_data:
  pgadmin_data:
```

**Warning:** `docker-compose down -v` deletes named volumes — all database data is lost. Use `docker-compose down` (without `-v`) to preserve data between restarts.

**Tip:** The `/app/tmp` anonymous mount shadows the host's `tmp/` directory inside the container. This prevents Air's compiled binaries (written to `tmp/`) from being written back to the host filesystem, which can cause permission issues on Linux.

---

## Environment Variables

**Option A: Inline in compose file (simple, dev-only):**

```yaml
services:
  app:
    environment:
      PORT: "8080"
      DATABASE_URL: "postgres://myapp:myapp@postgres:5432/myapp?sslmode=disable"
      GIN_MODE: "debug"
```

**Option B: `.env` file (recommended for dev):**

```yaml
services:
  app:
    env_file:
      - .env
```

`.env` file (never commit — add to `.gitignore`):

```dotenv
PORT=8080
DATABASE_URL=postgres://myapp:myapp@postgres:5432/myapp?sslmode=disable
REDIS_URL=redis://redis:6379
JWT_SECRET=dev-secret-change-in-production
GIN_MODE=debug
MIGRATIONS_PATH=db/migrations
POSTGRES_DB=myapp
POSTGRES_USER=myapp
POSTGRES_PASSWORD=myapp
```

`.env.example` (commit this — documents required variables):

```dotenv
PORT=8080
DATABASE_URL=postgres://user:password@localhost:5432/dbname?sslmode=disable
REDIS_URL=redis://localhost:6379
JWT_SECRET=replace-with-a-secure-random-string
GIN_MODE=debug
MIGRATIONS_PATH=db/migrations
```

**Option C: Variable substitution (compose reads from shell environment):**

```yaml
services:
  app:
    environment:
      DATABASE_URL: "${DATABASE_URL}"
      JWT_SECRET: "${JWT_SECRET}"
```

Compose substitutes `${VAR}` from the shell environment or `.env` file. Use this when you want CI/CD to inject secrets without an `.env` file on disk.

---

## Networking

By default, Compose creates a single network and attaches all services to it. Services reach each other by service name (DNS).

```yaml
# app can reach postgres at host "postgres", port 5432
DATABASE_URL: "postgres://myapp:myapp@postgres:5432/myapp?sslmode=disable"
#                                        ^^^^^^^
#                                        service name, not localhost
```

**Custom networks (isolate services):**

```yaml
services:
  app:
    networks:
      - frontend
      - backend

  postgres:
    networks:
      - backend  # not reachable from frontend network

  nginx:
    networks:
      - frontend

networks:
  frontend:
  backend:
    internal: true  # no external internet access from backend network
```

**Exposing ports:**

```yaml
services:
  app:
    ports:
      - "8080:8080"   # host:container — accessible from outside Docker

  postgres:
    ports:
      - "5432:5432"   # expose for local psql/pgadmin access (dev only)
      # Remove in production — postgres should only be reachable internally
```

---

## Integration Test Compose

A separate compose file for running integration tests in CI. Uses ephemeral services — no volume persistence needed.

```yaml
# docker-compose.test.yml
services:
  test:
    build:
      context: .
      dockerfile: Dockerfile.dev
    command: >
      sh -c "
        go test -v -race -tags=integration -count=1 ./...
      "
    environment:
      DATABASE_URL: "postgres://test:test@postgres-test:5432/test?sslmode=disable"
      REDIS_URL: "redis://redis-test:6379"
      GIN_MODE: "test"
    depends_on:
      postgres-test:
        condition: service_healthy
      redis-test:
        condition: service_healthy

  postgres-test:
    image: postgres:17-alpine
    environment:
      POSTGRES_DB: test
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
    # No volume — ephemeral, destroyed after tests
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U test -d test"]
      interval: 3s
      timeout: 3s
      retries: 10

  redis-test:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 3s
      timeout: 3s
      retries: 5
```

Run integration tests:

```bash
docker compose -f docker-compose.test.yml up --abort-on-container-exit --exit-code-from test
docker compose -f docker-compose.test.yml down -v  # clean up including volumes
```

In GitHub Actions:

```yaml
- name: Run integration tests
  run: |
    docker compose -f docker-compose.test.yml up \
      --abort-on-container-exit \
      --exit-code-from test

- name: Cleanup
  if: always()
  run: docker compose -f docker-compose.test.yml down -v
```

---

## Production-Like Compose

Mimics production: built image (not dev mount), no pgadmin, resource limits, restart policy.

```yaml
# docker-compose.prod.yml
services:
  app:
    image: myapp:${VERSION:-latest}  # use built image, not build context
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      PORT: "8080"
      GIN_MODE: "release"
      MIGRATIONS_PATH: "db/migrations"
    env_file:
      - .env.production  # contains DATABASE_URL, JWT_SECRET, etc.
    depends_on:
      postgres:
        condition: service_healthy
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 256M
        reservations:
          memory: 128M
    healthcheck:
      test: ["CMD", "/server", "-health-check"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 15s

  postgres:
    image: postgres:17-alpine
    restart: unless-stopped
    environment:
      POSTGRES_DB: "${POSTGRES_DB}"
      POSTGRES_USER: "${POSTGRES_USER}"
      POSTGRES_PASSWORD: "${POSTGRES_PASSWORD}"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5
    # No external port exposure — only app can reach it
    deploy:
      resources:
        limits:
          memory: 512M

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: redis-server --maxmemory 128mb --maxmemory-policy allkeys-lru
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
  redis_data:
```

---

## Complete docker-compose.yml

Full development compose with app (Air hot reload), postgres, redis, and pgadmin.

```yaml
# docker-compose.yml
# Usage: docker compose up
# First run: docker compose up --build

services:
  # ── Application ─────────────────────────────────────────────────────────────
  app:
    build:
      context: .
      dockerfile: Dockerfile.dev
    ports:
      - "${PORT:-8080}:8080"
    env_file:
      - .env
    environment:
      # Override DATABASE_URL to use docker service name (not localhost)
      DATABASE_URL: "postgres://${POSTGRES_USER:-myapp}:${POSTGRES_PASSWORD:-myapp}@postgres:5432/${POSTGRES_DB:-myapp}?sslmode=disable"
      REDIS_URL: "redis://redis:6379"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      # Hot reload: mount source into container
      - .:/app
      # Shadow tmp/ to avoid host/container permission conflicts
      - /app/tmp
    restart: unless-stopped

  # ── PostgreSQL ───────────────────────────────────────────────────────────────
  postgres:
    image: postgres:17-alpine
    ports:
      - "${POSTGRES_PORT:-5432}:5432"
    environment:
      POSTGRES_DB: "${POSTGRES_DB:-myapp}"
      POSTGRES_USER: "${POSTGRES_USER:-myapp}"
      POSTGRES_PASSWORD: "${POSTGRES_PASSWORD:-myapp}"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-myapp} -d ${POSTGRES_DB:-myapp}"]
      interval: 5s
      timeout: 5s
      retries: 5
      start_period: 10s
    restart: unless-stopped

  # ── Redis ────────────────────────────────────────────────────────────────────
  redis:
    image: redis:7-alpine
    ports:
      - "${REDIS_PORT:-6379}:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
    restart: unless-stopped

  # ── pgAdmin (database GUI — dev only) ────────────────────────────────────────
  pgadmin:
    image: dpage/pgadmin4:latest
    ports:
      - "${PGADMIN_PORT:-5050}:80"
    environment:
      PGADMIN_DEFAULT_EMAIL: "${PGADMIN_EMAIL:-admin@example.com}"
      PGADMIN_DEFAULT_PASSWORD: "${PGADMIN_PASSWORD:-admin}"
      PGADMIN_CONFIG_SERVER_MODE: "False"
    volumes:
      - pgadmin_data:/var/lib/pgadmin
    depends_on:
      - postgres
    restart: unless-stopped
    profiles:
      - tools  # opt-in: docker compose --profile tools up

volumes:
  postgres_data:
  redis_data:
  pgadmin_data:
```

**Common commands:**

```bash
# Start all services (build if needed)
docker compose up --build

# Start without pgadmin (default — pgadmin is in "tools" profile)
docker compose up

# Start with pgadmin
docker compose --profile tools up

# Rebuild app only (after go.mod change)
docker compose build app

# View logs for a specific service
docker compose logs -f app

# Run migrations manually inside the app container
docker compose exec app migrate -path db/migrations -database "$DATABASE_URL" up

# Connect to postgres with psql
docker compose exec postgres psql -U myapp -d myapp

# Stop all (preserve volumes)
docker compose down

# Stop and delete volumes (loses all DB data)
docker compose down -v
```

**For migration running in this compose setup**, the app service runs migrations on startup via `repository.RunMigrations` (library mode). See the **gin-database** skill (`references/migrations.md`) for startup vs CI/CD migration strategy.
