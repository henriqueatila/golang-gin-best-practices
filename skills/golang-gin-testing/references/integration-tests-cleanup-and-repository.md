# Integration Tests — Cleanup Patterns, Repository Tests, and API Integration Tests

## Cleanup Between Tests

Each test must start with clean database state.

**Pattern 1: Truncate in t.Cleanup (recommended):**
```go
func TestRepository_Create(t *testing.T) {
    t.Cleanup(func() { testDB.Exec("TRUNCATE TABLE users RESTART IDENTITY CASCADE") })
    repo := repository.NewUserRepository(testDB)
    if err := repo.Create(context.Background(), &domain.User{ID: "u1", Name: "Alice", Email: "alice@example.com", Role: "user"}); err != nil {
        t.Fatalf("Create: %v", err)
    }
}
```

**Pattern 2: Transaction rollback per test** — fast, but doesn't test commit behavior:
```go
func withTx(t *testing.T, db *gorm.DB, fn func(tx *gorm.DB)) {
    t.Helper()
    tx := db.Begin()
    if tx.Error != nil { t.Fatalf("begin tx: %v", tx.Error) }
    t.Cleanup(func() { tx.Rollback() })
    fn(tx)
}
func TestRepository_GetByID_InTx(t *testing.T) {
    withTx(t, testDB, func(tx *gorm.DB) {
        repo := repository.NewUserRepository(tx)
        _ = repo.Create(context.Background(), &domain.User{ID: "tmp", Name: "Temp", Email: "tmp@example.com", Role: "user"})
        got, err := repo.GetByID(context.Background(), "tmp")
        if err != nil { t.Fatalf("GetByID: %v", err) }
        if got.Name != "Temp" { t.Errorf("want 'Temp', got %q", got.Name) }
    })
}
```

---

## Repository Integration Tests

Test actual SQL behavior: constraints, not-found semantics, list filtering, pagination.

```go
//go:build integration
package repository_test

func TestUserRepository_Create_AndGetByID(t *testing.T) {
    t.Cleanup(func() { testDB.Exec("TRUNCATE TABLE users RESTART IDENTITY CASCADE") })
    repo := repository.NewUserRepository(testDB); ctx := context.Background()
    user := &domain.User{ID: "int-test-id", Name: "Alice", Email: "alice@example.com", Role: "user"}
    if err := repo.Create(ctx, user); err != nil { t.Fatalf("Create: %v", err) }
    got, err := repo.GetByID(ctx, user.ID)
    if err != nil { t.Fatalf("GetByID: %v", err) }
    if got.Email != user.Email { t.Errorf("want email %q, got %q", user.Email, got.Email) }
}

func TestUserRepository_GetByID_NotFound(t *testing.T) {
    _, err := repository.NewUserRepository(testDB).GetByID(context.Background(), "does-not-exist")
    if !errors.Is(err, domain.ErrNotFound) { t.Errorf("want ErrNotFound, got %v", err) }
}

func TestUserRepository_Create_DuplicateEmail_ReturnsConflict(t *testing.T) {
    t.Cleanup(func() { testDB.Exec("TRUNCATE TABLE users RESTART IDENTITY CASCADE") })
    repo := repository.NewUserRepository(testDB); ctx := context.Background()
    _ = repo.Create(ctx, &domain.User{ID: "u1", Name: "Alice", Email: "alice@example.com", Role: "user"})
    err := repo.Create(ctx, &domain.User{ID: "u2", Name: "Alice2", Email: "alice@example.com", Role: "user"})
    if !errors.Is(err, domain.ErrConflict) { t.Errorf("want ErrConflict on duplicate email, got %v", err) }
}

func TestUserRepository_List_FiltersByRole(t *testing.T) {
    t.Cleanup(func() { testDB.Exec("TRUNCATE TABLE users RESTART IDENTITY CASCADE") })
    repo := repository.NewUserRepository(testDB); ctx := context.Background()
    _ = repo.Create(ctx, &domain.User{ID: "u1", Name: "Admin", Email: "admin@example.com", Role: "admin"})
    _ = repo.Create(ctx, &domain.User{ID: "u2", Name: "User1", Email: "user1@example.com", Role: "user"})
    _ = repo.Create(ctx, &domain.User{ID: "u3", Name: "User2", Email: "user2@example.com", Role: "user"})
    users, total, err := repo.List(ctx, domain.ListOptions{Page: 1, Limit: 10, Role: "user"})
    if err != nil { t.Fatalf("List: %v", err) }
    if total != 2 { t.Errorf("want 2 users with role 'user', got %d", total) }
    if len(users) != 2 { t.Errorf("want 2 returned users, got %d", len(users)) }
}
```

---

## API Integration Tests

Test the full stack — real router + real database — using `router.ServeHTTP`. No network socket needed.

```go
//go:build integration
package handler_test

func buildIntegrationRouter(t *testing.T) (*gin.Engine, func()) {
    t.Helper()
    db := testdb.NewPostgres(t, &repository.UserModel{})
    repo := repository.NewUserRepository(db)
    svc := service.NewUserService(repo, slog.Default())
    h := handler.NewUserHandler(svc, slog.Default())
    r := gin.New(); r.POST("/users", h.Create); r.GET("/users/:id", h.GetByID)
    return r, func() { db.Exec("TRUNCATE TABLE users RESTART IDENTITY CASCADE") }
}

func TestUserAPI_CreateAndRetrieve(t *testing.T) {
    router, cleanup := buildIntegrationRouter(t); t.Cleanup(cleanup)
    body := `{"name":"Alice","email":"alice@example.com","password":"secret1234"}`
    req := httptest.NewRequest(http.MethodPost, "/users", strings.NewReader(body))
    req.Header.Set("Content-Type", "application/json")
    w := httptest.NewRecorder(); router.ServeHTTP(w, req)
    if w.Code != http.StatusCreated { t.Fatalf("Create: want 201, got %d; body: %s", w.Code, w.Body) }
    var created domain.User
    if err := json.Unmarshal(w.Body.Bytes(), &created); err != nil { t.Fatalf("parse create response: %v", err) }
    if created.ID == "" { t.Fatal("expected non-empty ID in response") }

    req2 := httptest.NewRequest(http.MethodGet, "/users/"+created.ID, nil)
    w2 := httptest.NewRecorder(); router.ServeHTTP(w2, req2)
    if w2.Code != http.StatusOK { t.Fatalf("GetByID: want 200, got %d; body: %s", w2.Code, w2.Body) }
    var fetched domain.User
    if err := json.Unmarshal(w2.Body.Bytes(), &fetched); err != nil { t.Fatalf("parse get response: %v", err) }
    if fetched.Email != "alice@example.com" { t.Errorf("want email 'alice@example.com', got %q", fetched.Email) }
}
```

> For setup, TestMain, and DB helper: see [integration-tests-setup-and-testmain.md](integration-tests-setup-and-testmain.md).
> For migration tests and fixture loading: see [integration-tests-api-migrations-and-fixtures.md](integration-tests-api-migrations-and-fixtures.md).
