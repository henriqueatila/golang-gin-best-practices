# Configuration — .dockerignore, Dockerfile, and docker-compose Quick Reference

> These are condensed quick-reference snippets. For full patterns see the dedicated dockerfile-* and docker-compose-* reference files.

## .dockerignore

```dockerignore
.git; .gitignore
*.test; *.out; coverage.html
.env; .env.*
docker-compose*.yml; air.toml
*.md; docs/; plans/
.github/
```

**Why:** Excluding `.git` and `*.md` keeps the build context small. Excluding `.env` prevents secrets from leaking into the image. See [dockerfile-size-optimization-and-complete-example.md](dockerfile-size-optimization-and-complete-example.md) for full `.dockerignore`.

---

## Multi-Stage Dockerfile (Quick Reference)

```dockerfile
# syntax=docker/dockerfile:1

# Stage 1: Builder
FROM golang:1.24-bookworm AS builder
WORKDIR /build
COPY go.mod go.sum ./
RUN go mod download
COPY . .
ARG TARGETARCH
RUN CGO_ENABLED=0 GOOS=linux GOARCH=${TARGETARCH} go build \
    -ldflags="-s -w" -trimpath -o /app/server ./cmd/api

# Stage 2: Runtime
FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=builder /app/server /server
EXPOSE 8080
ENTRYPOINT ["/server"]
```

`CGO_ENABLED=0` is mandatory for distroless — produces a statically linked binary. Final image ~10 MB vs ~800 MB with the golang base. See [dockerfile-multistage-and-base-images.md](dockerfile-multistage-and-base-images.md) for full explanation with BuildKit cache mounts, build args, and base image comparison.

---

## docker-compose for Local Development (Quick Reference)

```yaml
# docker-compose.yml
services:
  app:
    build: .
    ports: ["8080:8080"]
    environment:
      PORT: "8080"; DATABASE_URL: "postgres://myapp:myapp@postgres:5432/myapp?sslmode=disable"
      REDIS_URL: "redis://redis:6379"; JWT_SECRET: "dev-secret"; GIN_MODE: "debug"
    depends_on:
      postgres: { condition: service_healthy }
      redis:    { condition: service_healthy }
    volumes: [.:/app]  # for Air hot reload — remove in production
  postgres:
    image: postgres:17-alpine
    environment: { POSTGRES_DB: myapp, POSTGRES_USER: myapp, POSTGRES_PASSWORD: myapp }
    ports: ["5432:5432"]
    volumes: [postgres_data:/var/lib/postgresql/data]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myapp -d myapp"]
      interval: 5s; timeout: 5s; retries: 5
  redis:
    image: redis:7-alpine; ports: ["6379:6379"]
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]; interval: 5s; timeout: 3s; retries: 5
volumes:
  postgres_data:
```

For Air hot reload dev image, integration test compose, production-like compose, and full example: see [docker-compose-dev-setup.md](docker-compose-dev-setup.md) and [docker-compose-test-and-prod.md](docker-compose-test-and-prod.md).

For migration running in Docker: see the **golang-gin-database** skill (`references/migrations.md`).

> For health check handler and config loader: see [configuration-health-and-config-loader.md](configuration-health-and-config-loader.md).
