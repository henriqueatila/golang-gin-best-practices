# Docker Compose — Development Setup, Health Checks, Volumes, and Networking

> docker-compose patterns are not part of the Gin framework — mainstream Go community patterns.

## Development Compose with Air Hot Reload

[Air](https://github.com/air-verse/air) watches Go source files and rebuilds on change.

```dockerfile
# Dockerfile.dev
FROM golang:1.24-bookworm
WORKDIR /app
RUN go install github.com/air-verse/air@latest
RUN go install -tags 'postgres' github.com/golang-migrate/migrate/v4/cmd/migrate@latest
COPY go.mod go.sum ./
RUN go mod download
EXPOSE 8080
CMD ["air", "-c", ".air.toml"]
```

`.air.toml` key settings: `bin = "./tmp/main"`, `cmd = "go build -o ./tmp/main ./cmd/api"`, `delay = 500`, `exclude_dir = ["tmp", "vendor", "testdata"]`, `exclude_regex = ["_test.go", "_mock.go"]`, `include_ext = ["go", "html", "toml", "env"]`.

---

## Service Dependencies and Health Checks

`depends_on` with `condition: service_healthy` waits for the service inside the container to be ready, not just the container to start.

```yaml
services:
  app:
    depends_on:
      postgres: { condition: service_healthy }
      redis:    { condition: service_healthy }
  postgres:
    image: postgres:17-alpine
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 5s; timeout: 5s; retries: 5
      start_period: 10s  # grace period for first-run initialization (5–15s)
  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s; timeout: 3s; retries: 5
```

---

## Volume Mounts

```yaml
services:
  app:
    volumes:
      - .:/app        # source mount for hot reload (dev only)
      - /app/tmp      # shadow tmp/ — prevents Air binaries from writing to host
  postgres:
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./db/init:/docker-entrypoint-initdb.d:ro  # init scripts on first start
volumes:
  postgres_data:
  pgadmin_data:
```

**Warning:** `docker-compose down -v` deletes named volumes — all DB data is lost. Use `docker-compose down` to preserve data.

---

## Environment Variables

**Option A — inline (dev only):** `environment: PORT: "8080"`.

**Option B — `.env` file (recommended):** `env_file: [.env]`. Never commit `.env`; commit `.env.example` (documents required vars). Example vars: `PORT`, `DATABASE_URL`, `REDIS_URL`, `JWT_SECRET`, `GIN_MODE`, `POSTGRES_DB/USER/PASSWORD`.

**Option C — shell substitution:** `environment: DATABASE_URL: "${DATABASE_URL}"` — compose reads from shell or `.env`. Use in CI/CD to inject secrets without an `.env` file on disk.

---

## Networking

Services reach each other by service name (DNS). Default: single network for all services.

```yaml
# app reaches postgres at host "postgres", port 5432
DATABASE_URL: "postgres://myapp:myapp@postgres:5432/myapp?sslmode=disable"
```

**Custom networks:**
```yaml
services:
  app:     { networks: [frontend, backend] }
  postgres: { networks: [backend] }          # unreachable from frontend
  nginx:   { networks: [frontend] }
networks:
  frontend:
  backend: { internal: true }  # no external internet access
```

**Ports:** expose `"8080:8080"` for the app. Expose `"5432:5432"` for postgres only in dev — remove in production.

> For integration test compose, production-like compose, and complete example: see [docker-compose-test-and-prod.md](docker-compose-test-and-prod.md).
