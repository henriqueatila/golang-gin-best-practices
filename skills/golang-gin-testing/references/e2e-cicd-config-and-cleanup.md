# E2E Tests — Docker Compose, GitHub Actions CI/CD, Config, and Cleanup

## Testing with docker-compose

For e2e tests needing external services (Redis, message queues):

```yaml
# docker-compose.test.yml
services:
  postgres:
    image: postgres:16-alpine
    environment: { POSTGRES_DB: testdb, POSTGRES_USER: testuser, POSTGRES_PASSWORD: testpass }
    ports: ["5433:5432"]  # 5433 to avoid colliding with dev postgres
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U testuser -d testdb"]
      interval: 5s; timeout: 5s; retries: 5
  redis:
    image: redis:7-alpine; ports: ["6380:6379"]
    healthcheck: { test: ["CMD", "redis-cli", "ping"], interval: 5s, timeout: 3s, retries: 5 }
```

```bash
docker compose -f docker-compose.test.yml up -d --wait
E2E_DATABASE_URL="postgres://testuser:testpass@localhost:5433/testdb?sslmode=disable" \
E2E_REDIS_URL="redis://localhost:6380" \
go test -v -tags=e2e ./internal/e2e/...
docker compose -f docker-compose.test.yml down -v
```

Fall back to testcontainers when env var not set: `if dsn := os.Getenv("E2E_DATABASE_URL"); dsn == "" { /* use testcontainers */ }`.

---

## GitHub Actions CI/CD Integration

Three separate jobs: unit → integration → e2e (sequential with `needs`).

```yaml
# .github/workflows/test.yml
jobs:
  unit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with: { go-version: "1.24", cache: true }
      - name: Run unit tests
        run: go test -v -race -cover -tags='!integration !e2e' ./...
      - name: Check 80% coverage threshold
        run: |
          go test ./... -coverprofile=coverage.out -tags='!integration !e2e'
          COVERAGE=$(go tool cover -func=coverage.out | grep total | awk '{print $3}' | tr -d '%')
          if (( $(echo "$COVERAGE < 80" | bc -l) )); then echo "Coverage ${COVERAGE}% below 80%"; exit 1; fi

  integration:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env: { POSTGRES_DB: testdb, POSTGRES_USER: testuser, POSTGRES_PASSWORD: testpass }
        ports: [5432:5432]
        options: --health-cmd "pg_isready -U testuser -d testdb" --health-interval 5s --health-timeout 5s --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with: { go-version: "1.24", cache: true }
      - name: Run integration tests
        env: { TEST_DATABASE_URL: postgres://testuser:testpass@localhost:5432/testdb?sslmode=disable }
        run: go test -v -race -tags=integration ./internal/repository/...

  e2e:
    runs-on: ubuntu-latest
    needs: [unit, integration]
    services:
      postgres:
        image: postgres:16-alpine
        env: { POSTGRES_DB: e2edb, POSTGRES_USER: e2euser, POSTGRES_PASSWORD: e2epass }
        ports: [5432:5432]
        options: --health-cmd "pg_isready -U e2euser -d e2edb" --health-interval 5s --health-timeout 5s --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with: { go-version: "1.24", cache: true }
      - name: Run e2e tests
        env: { E2E_DATABASE_URL: postgres://e2euser:e2epass@localhost:5432/e2edb?sslmode=disable }
        run: go test -v -race -tags=e2e -timeout=120s ./internal/e2e/...
```

---

## Test Environment Configuration

```go
//go:build e2e
package e2e_test

type e2eConfig struct {
    DatabaseURL string; TokenConfig auth.TokenConfig; Timeout time.Duration
}

func loadE2EConfig() e2eConfig {
    return e2eConfig{
        DatabaseURL: os.Getenv("E2E_DATABASE_URL"), // empty → testcontainers
        TokenConfig: auth.TokenConfig{
            AccessSecret:  []byte(envOr("E2E_JWT_SECRET", "e2e-test-access-secret-32-bytes!!")),
            RefreshSecret: []byte(envOr("E2E_JWT_REFRESH_SECRET", "e2e-test-refresh-secret-32!!!!!")),
            AccessTTL: 15 * time.Minute, RefreshTTL: 7 * 24 * time.Hour,
        },
        Timeout: 30 * time.Second,
    }
}
func envOr(key, fallback string) string { if v := os.Getenv(key); v != "" { return v }; return fallback }
```

---

## Cleanup and Idempotency

Rules: (1) each test owns its data — `t.Cleanup` truncates; (2) never depend on data from another test; (3) unique identifiers per test run; (4) truncate not drop.

```go
func uniqueEmail(t *testing.T) string {
    t.Helper()
    safe := strings.NewReplacer("/", "-", " ", "-").Replace(t.Name())
    return fmt.Sprintf("user+%s@e2e.com", strings.ToLower(safe))
}

func TestUserFlow_ParallelSafe(t *testing.T) {
    t.Parallel()
    email := uniqueEmail(t)
    t.Cleanup(func() { testApp.db.Exec("DELETE FROM users WHERE email = ?", email) })
    testApp.do(t, http.MethodPost, "/api/v1/users", fmt.Sprintf(`{"name":"Test","email":%q,"password":"secret1234"}`, email), "")
}
```

**Idempotency checklist:**
- [ ] Test passes on first and second run without manual cleanup
- [ ] Test passes when run in parallel with other tests
- [ ] `t.Cleanup` registered before any assertions (runs even if test fails)
- [ ] No hardcoded IDs or emails that conflict across runs

> For e2e structure, critical flow, and in-process setup: see [e2e-structure-and-critical-flow.md](e2e-structure-and-critical-flow.md).
