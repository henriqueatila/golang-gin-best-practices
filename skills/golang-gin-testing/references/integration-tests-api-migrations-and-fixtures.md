# Integration Tests — Migration Tests and Fixture Loading

## Testing Migrations

Verify migrations apply cleanly and the schema matches expected structure.

```go
//go:build integration
package repository_test

func TestMigrations_UpAndDown(t *testing.T) {
    ctx := context.Background()
    pgc, err := tcpostgres.RunContainer(ctx,
        testcontainers.WithImage("postgres:16-alpine"),
        tcpostgres.WithDatabase("migrate_test"), tcpostgres.WithUsername("u"), tcpostgres.WithPassword("p"),
        testcontainers.WithWaitStrategy(
            wait.ForLog("database system is ready to accept connections").WithOccurrence(2).WithStartupTimeout(20*time.Second)),
    )
    if err != nil { t.Fatalf("start container: %v", err) }
    t.Cleanup(func() { pgc.Terminate(ctx) })

    host, _ := pgc.Host(ctx)
    port, _ := pgc.MappedPort(ctx, "5432")
    dsn := fmt.Sprintf("host=%s port=%s user=u password=p dbname=migrate_test sslmode=disable", host, port.Port())

    db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{})
    if err != nil { t.Fatalf("gorm.Open: %v", err) }
    sqlDB, _ := db.DB()

    driver, err := migratepg.WithInstance(sqlDB, &migratepg.Config{})
    if err != nil { t.Fatalf("migrate driver: %v", err) }
    m, err := migrate.NewWithDatabaseInstance("file://../../migrations", "postgres", driver)
    if err != nil { t.Fatalf("migrate.New: %v", err) }

    // Apply all migrations
    if err := m.Up(); err != nil && err != migrate.ErrNoChange { t.Fatalf("migrate Up: %v", err) }

    // Verify expected tables exist
    var tableName string
    row := sqlDB.QueryRow("SELECT table_name FROM information_schema.tables WHERE table_schema='public' AND table_name='users'")
    if err := row.Scan(&tableName); err != nil { t.Fatalf("users table not found after migration: %v", err) }

    // Roll back all migrations
    if err := m.Down(); err != nil && err != migrate.ErrNoChange { t.Fatalf("migrate Down: %v", err) }
}
```

---

## Fixture Loading

Load SQL fixture files to populate a known database state before tests.

```go
//go:build integration
package testdb

// LoadFixture executes SQL from a fixture file against the given db.
// Fixture files live in internal/testutil/fixtures/*.sql
func LoadFixture(t *testing.T, db *gorm.DB, path string) {
    t.Helper()
    sql, err := os.ReadFile(path)
    if err != nil { t.Fatalf("LoadFixture: read %q: %v", path, err) }
    if err := db.Exec(string(sql)).Error; err != nil { t.Fatalf("LoadFixture: exec %q: %v", path, err) }
}
```

Example fixture `internal/testutil/fixtures/users.sql`:

```sql
INSERT INTO users (id, name, email, role, created_at, updated_at) VALUES
    ('seed-user-1', 'Alice', 'alice@example.com', 'user',  NOW(), NOW()),
    ('seed-user-2', 'Bob',   'bob@example.com',   'user',  NOW(), NOW()),
    ('seed-admin',  'Admin', 'admin@example.com', 'admin', NOW(), NOW());
```

Usage:

```go
//go:build integration

func TestUserRepository_List_WithFixtures(t *testing.T) {
    t.Cleanup(func() { testDB.Exec("TRUNCATE TABLE users RESTART IDENTITY CASCADE") })
    testdb.LoadFixture(t, testDB, "../../testutil/fixtures/users.sql")

    users, total, err := repository.NewUserRepository(testDB).List(context.Background(), domain.ListOptions{Page: 1, Limit: 10})
    if err != nil { t.Fatalf("List: %v", err) }
    if total != 3 { t.Errorf("want 3 users from fixture, got %d", total) }
    _ = users
}
```

> For setup, TestMain, and DB helper: see [integration-tests-setup-and-testmain.md](integration-tests-setup-and-testmain.md).
> For cleanup patterns and repository/API tests: see [integration-tests-cleanup-and-repository.md](integration-tests-cleanup-and-repository.md).
