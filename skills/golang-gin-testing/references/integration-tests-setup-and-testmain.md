# Integration Tests — Dependencies, Build Tags, TestMain, and DB Helper

Write integration tests when you need to verify actual SQL behavior — migrations, GORM/sqlx queries, constraint enforcement, transaction rollbacks. Unit tests with mocked repositories cannot catch these.

## Dependencies

```bash
go get github.com/testcontainers/testcontainers-go
go get github.com/testcontainers/testcontainers-go/modules/postgres
go get gorm.io/driver/postgres
go get github.com/golang-migrate/migrate/v4
```

Requires Docker on the host or in CI.

---

## Build Tags

Place at the top of every integration test file, before the `package` declaration:

```go
//go:build integration
```

```bash
go test -v -race -cover -tags='!integration' ./...         # unit tests only (fast)
go test -v -race -tags=integration ./internal/repository/... # integration tests
```

---

## TestMain — DB Lifecycle

`TestMain` runs once per package — amortizes container startup cost across all tests.

```go
//go:build integration
package repository_test

var (testDB *gorm.DB; testContainer testcontainers.Container)

func TestMain(m *testing.M) {
    ctx := context.Background()
    pgContainer, err := tcpostgres.RunContainer(ctx,
        testcontainers.WithImage("postgres:16-alpine"),
        tcpostgres.WithDatabase("testdb"), tcpostgres.WithUsername("testuser"), tcpostgres.WithPassword("testpass"),
        testcontainers.WithWaitStrategy(
            wait.ForLog("database system is ready to accept connections").WithOccurrence(2).WithStartupTimeout(30*time.Second)),
    )
    if err != nil { slog.Error("start postgres container", "error", err); os.Exit(1) }
    testContainer = pgContainer

    host, _ := pgContainer.Host(ctx)
    port, _ := pgContainer.MappedPort(ctx, "5432")
    dsn := fmt.Sprintf("host=%s port=%s user=testuser password=testpass dbname=testdb sslmode=disable", host, port.Port())

    db, err := gorm.Open(gormpostgres.Open(dsn), &gorm.Config{})
    if err != nil { slog.Error("connect to test db", "error", err); testContainer.Terminate(ctx); os.Exit(1) }
    testDB = db

    if err := db.AutoMigrate(&repository.UserModel{}); err != nil {
        slog.Error("migrations failed", "error", err); testContainer.Terminate(ctx); os.Exit(1)
    }
    code := m.Run()
    testContainer.Terminate(ctx)
    os.Exit(code)
}
```

---

## Test Database Helper

Reusable helper so multiple packages share the same setup pattern:

```go
//go:build integration
package testdb

// NewPostgres starts a PostgreSQL container, auto-migrates models, registers t.Cleanup to terminate.
func NewPostgres(t *testing.T, models ...any) *gorm.DB {
    t.Helper()
    ctx := context.Background()
    pgContainer, err := postgres.RunContainer(ctx,
        testcontainers.WithImage("postgres:16-alpine"),
        postgres.WithDatabase("testdb"), postgres.WithUsername("testuser"), postgres.WithPassword("testpass"),
        testcontainers.WithWaitStrategy(
            wait.ForLog("database system is ready to accept connections").WithOccurrence(2).WithStartupTimeout(30*time.Second)),
    )
    if err != nil { t.Fatalf("start container: %v", err) }
    t.Cleanup(func() { pgContainer.Terminate(ctx) })

    host, _ := pgContainer.Host(ctx)
    port, _ := pgContainer.MappedPort(ctx, "5432")
    dsn := fmt.Sprintf("host=%s port=%s user=testuser password=testpass dbname=testdb sslmode=disable", host, port.Port())

    db, err := gorm.Open(pgdriver.Open(dsn), &gorm.Config{})
    if err != nil { t.Fatalf("gorm.Open: %v", err) }
    if len(models) > 0 {
        if err := db.AutoMigrate(models...); err != nil { t.Fatalf("AutoMigrate: %v", err) }
    }
    return db
}

// Truncate clears rows from tables. SAFETY: pass only compile-time constant table names — never user input.
func Truncate(t *testing.T, db *gorm.DB, tables ...string) {
    t.Helper()
    for _, table := range tables {
        if err := db.Exec("TRUNCATE TABLE " + table + " RESTART IDENTITY CASCADE").Error; err != nil {
            t.Fatalf("Truncate(%q): %v", table, err)
        }
    }
}
```

> For cleanup patterns and repository integration tests: see [integration-tests-cleanup-and-repository.md](integration-tests-cleanup-and-repository.md).
> For API tests, migration tests, and fixture loading: see [integration-tests-api-migrations-and-fixtures.md](integration-tests-api-migrations-and-fixtures.md).
