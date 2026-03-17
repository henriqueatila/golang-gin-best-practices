# Docker Compose — Integration Tests, Production, and Complete Example

## Integration Test Compose

Ephemeral services for CI — no volume persistence needed.

```yaml
# docker-compose.test.yml
services:
  test:
    build: { context: ., dockerfile: Dockerfile.dev }
    command: sh -c "go test -v -race -tags=integration -count=1 ./..."
    environment:
      DATABASE_URL: "postgres://test:test@postgres-test:5432/test?sslmode=disable"
      REDIS_URL: "redis://redis-test:6379"; GIN_MODE: "test"
    depends_on:
      postgres-test: { condition: service_healthy }
      redis-test:    { condition: service_healthy }
  postgres-test:
    image: postgres:17-alpine
    environment: { POSTGRES_DB: test, POSTGRES_USER: test, POSTGRES_PASSWORD: test }
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U test -d test"]
      interval: 3s; timeout: 3s; retries: 10
  redis-test:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 3s; timeout: 3s; retries: 5
```

```bash
docker compose -f docker-compose.test.yml up --abort-on-container-exit --exit-code-from test
docker compose -f docker-compose.test.yml down -v
```

GitHub Actions: same commands, with `if: always()` on the cleanup step.

---

## Production-Like Compose

Mimics production: built image, no dev mounts, resource limits, restart policy.

```yaml
# docker-compose.prod.yml
services:
  app:
    image: myapp:${VERSION:-latest}
    restart: unless-stopped
    ports: ["8080:8080"]
    environment: { PORT: "8080", GIN_MODE: "release", MIGRATIONS_PATH: "db/migrations" }
    env_file: [.env.production]
    depends_on: { postgres: { condition: service_healthy } }
    deploy:
      resources:
        limits:     { cpus: "1.0", memory: 256M }
        reservations: { memory: 128M }
    healthcheck:
      test: ["CMD", "/server", "-health-check"]
      interval: 30s; timeout: 5s; retries: 3; start_period: 15s
  postgres:
    image: postgres:17-alpine
    restart: unless-stopped
    environment: { POSTGRES_DB: "${POSTGRES_DB}", POSTGRES_USER: "${POSTGRES_USER}", POSTGRES_PASSWORD: "${POSTGRES_PASSWORD}" }
    volumes: [postgres_data:/var/lib/postgresql/data]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s; timeout: 5s; retries: 5
    deploy: { resources: { limits: { memory: 512M } } }
  redis:
    image: redis:7-alpine; restart: unless-stopped
    command: redis-server --maxmemory 128mb --maxmemory-policy allkeys-lru
    volumes: [redis_data:/data]
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]; interval: 10s; timeout: 5s; retries: 5
volumes:
  postgres_data:; redis_data:
```

---

## Complete docker-compose.yml

Full dev compose: app (Air hot reload), postgres, redis, pgadmin (tools profile).

```yaml
# docker-compose.yml — docker compose up
services:
  app:
    build: { context: ., dockerfile: Dockerfile.dev }
    ports: ["${PORT:-8080}:8080"]
    env_file: [.env]
    environment:
      DATABASE_URL: "postgres://${POSTGRES_USER:-myapp}:${POSTGRES_PASSWORD:-myapp}@postgres:5432/${POSTGRES_DB:-myapp}?sslmode=disable"
      REDIS_URL: "redis://redis:6379"
    depends_on:
      postgres: { condition: service_healthy }
      redis:    { condition: service_healthy }
    volumes: [.:/app, /app/tmp]
    restart: unless-stopped
  postgres:
    image: postgres:17-alpine
    ports: ["${POSTGRES_PORT:-5432}:5432"]
    environment: { POSTGRES_DB: "${POSTGRES_DB:-myapp}", POSTGRES_USER: "${POSTGRES_USER:-myapp}", POSTGRES_PASSWORD: "${POSTGRES_PASSWORD:-myapp}" }
    volumes: [postgres_data:/var/lib/postgresql/data]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-myapp} -d ${POSTGRES_DB:-myapp}"]
      interval: 5s; timeout: 5s; retries: 5; start_period: 10s
    restart: unless-stopped
  redis:
    image: redis:7-alpine; ports: ["${REDIS_PORT:-6379}:6379"]
    volumes: [redis_data:/data]
    healthcheck: { test: ["CMD", "redis-cli", "ping"], interval: 5s, timeout: 3s, retries: 5 }
    restart: unless-stopped
  pgadmin:
    image: dpage/pgadmin4:latest; ports: ["${PGADMIN_PORT:-5050}:80"]
    environment: { PGADMIN_DEFAULT_EMAIL: "${PGADMIN_EMAIL:-admin@example.com}", PGADMIN_DEFAULT_PASSWORD: "${PGADMIN_PASSWORD:-admin}", PGADMIN_CONFIG_SERVER_MODE: "False" }
    volumes: [pgadmin_data:/var/lib/pgadmin]
    depends_on: [postgres]; restart: unless-stopped
    profiles: [tools]  # opt-in: docker compose --profile tools up
volumes:
  postgres_data:; redis_data:; pgadmin_data:
```

Common commands: `docker compose up --build`, `docker compose --profile tools up`, `docker compose build app`, `docker compose logs -f app`, `docker compose exec app migrate -path db/migrations -database "$DATABASE_URL" up`, `docker compose exec postgres psql -U myapp -d myapp`, `docker compose down` (preserve volumes), `docker compose down -v` (delete volumes).

For migration strategy (startup vs CI/CD): see the **golang-gin-database** skill (`references/migrations.md`).

> For development setup, health checks, volumes, and networking: see [docker-compose-dev-setup.md](docker-compose-dev-setup.md).
