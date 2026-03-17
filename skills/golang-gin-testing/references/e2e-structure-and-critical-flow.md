# E2E Tests — Structure, Critical User Flow, and In-Process Setup

Write e2e tests for flows spanning multiple services or requiring full wiring: auth token generation, protected route access, cascading deletes. Focus on critical user journeys — not every endpoint needs an e2e test.

## E2E Test Structure

```
internal/e2e/
├── e2e_test.go         # TestMain, app setup
├── user_flow_test.go   # register → login → CRUD
└── auth_flow_test.go   # token refresh, RBAC
```

Build tag: `//go:build e2e`. Run: `go test -v -tags=e2e ./internal/e2e/...`.

---

## Critical Flow: Register → Login → CRUD

```go
//go:build e2e
package e2e_test

var testApp *appUnderTest  // initialized in TestMain

func TestUserFlow_RegisterLoginCRUD(t *testing.T) {
    t.Cleanup(func() { testApp.truncate(t, "users") })

    // Step 1: Register
    w := testApp.do(t, http.MethodPost, "/api/v1/users", `{"name":"Alice","email":"alice@e2e.com","password":"secret1234"}`, "")
    if w.Code != http.StatusCreated { t.Fatalf("register: want 201, got %d; body: %s", w.Code, w.Body) }
    var created domain.User; mustUnmarshal(t, w, &created)
    if created.ID == "" { t.Fatal("register: expected non-empty user ID") }

    // Step 2: Login
    w = testApp.do(t, http.MethodPost, "/api/v1/auth/login", `{"email":"alice@e2e.com","password":"secret1234"}`, "")
    if w.Code != http.StatusOK { t.Fatalf("login: want 200, got %d; body: %s", w.Code, w.Body) }
    var tokens struct{ AccessToken string `json:"access_token"` }
    mustUnmarshal(t, w, &tokens)
    if tokens.AccessToken == "" { t.Fatal("login: expected non-empty access_token") }

    // Step 3: Fetch profile
    w = testApp.do(t, http.MethodGet, "/api/v1/users/"+created.ID, "", tokens.AccessToken)
    if w.Code != http.StatusOK { t.Fatalf("get user: want 200, got %d; body: %s", w.Code, w.Body) }
    var fetched domain.User; mustUnmarshal(t, w, &fetched)
    if fetched.Email != "alice@e2e.com" { t.Errorf("want email 'alice@e2e.com', got %q", fetched.Email) }

    // Step 4: Update
    w = testApp.do(t, http.MethodPut, "/api/v1/users/"+created.ID, `{"name":"Alice Updated","email":"alice@e2e.com"}`, tokens.AccessToken)
    if w.Code != http.StatusOK { t.Fatalf("update: want 200, got %d; body: %s", w.Code, w.Body) }

    // Step 5: Delete
    w = testApp.do(t, http.MethodDelete, "/api/v1/users/"+created.ID, "", tokens.AccessToken)
    if w.Code != http.StatusNoContent { t.Fatalf("delete: want 204, got %d; body: %s", w.Code, w.Body) }

    // Step 6: Confirm gone
    w = testApp.do(t, http.MethodGet, "/api/v1/users/"+created.ID, "", tokens.AccessToken)
    if w.Code != http.StatusNotFound { t.Errorf("after delete: want 404, got %d", w.Code) }
}

func TestUserFlow_LoginWithWrongPassword(t *testing.T) {
    t.Cleanup(func() { testApp.truncate(t, "users") })
    testApp.do(t, http.MethodPost, "/api/v1/users", `{"name":"Bob","email":"bob@e2e.com","password":"secret1234"}`, "")
    w := testApp.do(t, http.MethodPost, "/api/v1/auth/login", `{"email":"bob@e2e.com","password":"wrongpassword"}`, "")
    if w.Code != http.StatusUnauthorized { t.Errorf("wrong password: want 401, got %d", w.Code) }
}

func TestUserFlow_ProtectedRoute_WithoutToken(t *testing.T) {
    w := testApp.do(t, http.MethodGet, "/api/v1/users/any-id", "", "")
    if w.Code != http.StatusUnauthorized { t.Errorf("no token: want 401, got %d", w.Code) }
}

func mustUnmarshal(t *testing.T, w *httptest.ResponseRecorder, dst any) {
    t.Helper()
    if err := json.Unmarshal(w.Body.Bytes(), dst); err != nil { t.Fatalf("mustUnmarshal: %v\nbody: %s", err, w.Body) }
}
```

---

## In-Process E2E with httptest

Wire the full application in-process — no network socket. Fast, deterministic, reproducible.

```go
//go:build e2e
package e2e_test

type appUnderTest struct{ router *gin.Engine; db *gorm.DB }

func (a *appUnderTest) do(t *testing.T, method, path, body, token string) *httptest.ResponseRecorder {
    t.Helper()
    r := strings.NewReader(body)
    req, err := http.NewRequest(method, path, r)
    if err != nil { t.Fatalf("do: http.NewRequest: %v", err) }
    if body != "" { req.Header.Set("Content-Type", "application/json") }
    if token != "" { req.Header.Set("Authorization", "Bearer "+token) }
    w := httptest.NewRecorder(); a.router.ServeHTTP(w, req); return w
}

// truncate clears rows between tests. SAFETY: pass only compile-time constant table names.
func (a *appUnderTest) truncate(t *testing.T, tables ...string) {
    t.Helper()
    for _, tbl := range tables { a.db.Exec("TRUNCATE TABLE " + tbl + " RESTART IDENTITY CASCADE") }
}

func buildApp(db *gorm.DB) *appUnderTest {
    gin.SetMode(gin.TestMode)
    logger := slog.New(slog.NewTextHandler(os.Stderr, &slog.HandlerOptions{Level: slog.LevelWarn}))
    tokenCfg := auth.TokenConfig{
        AccessSecret: []byte("e2e-test-access-secret-32-bytes!!"), RefreshSecret: []byte("e2e-test-refresh-secret-32-bytes!"),
        AccessTTL: 15 * time.Minute, RefreshTTL: 7 * 24 * time.Hour,
    }
    userRepo := repository.NewUserRepository(db); userSvc := service.NewUserService(userRepo, logger)
    userHandler := handler.NewUserHandler(userSvc, logger); authHandler := handler.NewAuthHandler(userRepo, tokenCfg, logger)
    r := gin.New(); r.Use(middleware.Recovery(logger))
    api := r.Group("/api/v1")
    public := api.Group("")
    public.POST("/users", userHandler.Create); public.POST("/auth/login", authHandler.Login); public.POST("/auth/refresh", authHandler.Refresh)
    protected := api.Group(""); protected.Use(middleware.Auth(tokenCfg, logger))
    protected.GET("/users/:id", userHandler.GetByID); protected.PUT("/users/:id", userHandler.Update); protected.DELETE("/users/:id", userHandler.Delete)
    return &appUnderTest{router: r, db: db}
}

func TestMain(m *testing.M) {
    ctx := context.Background()
    pgc, err := tcpostgres.RunContainer(ctx,
        testcontainers.WithImage("postgres:16-alpine"),
        tcpostgres.WithDatabase("e2edb"), tcpostgres.WithUsername("e2euser"), tcpostgres.WithPassword("e2epass"),
        testcontainers.WithWaitStrategy(wait.ForLog("database system is ready to accept connections").WithOccurrence(2).WithStartupTimeout(30*time.Second)),
    )
    if err != nil { slog.Error("e2e: start container", "error", err); os.Exit(1) }
    host, _ := pgc.Host(ctx); port, _ := pgc.MappedPort(ctx, "5432")
    dsn := fmt.Sprintf("host=%s port=%s user=e2euser password=e2epass dbname=e2edb sslmode=disable", host, port.Port())
    db, err := gorm.Open(pgdriver.Open(dsn), &gorm.Config{})
    if err != nil { slog.Error("e2e: gorm.Open", "error", err); pgc.Terminate(ctx); os.Exit(1) }
    if err := db.AutoMigrate(&repository.UserModel{}); err != nil { slog.Error("e2e: AutoMigrate", "error", err); pgc.Terminate(ctx); os.Exit(1) }
    testApp = buildApp(db)
    code := m.Run(); pgc.Terminate(ctx); os.Exit(code)
}
```

> For docker-compose testing, GitHub Actions CI/CD, config, and cleanup: see [e2e-cicd-config-and-cleanup.md](e2e-cicd-config-and-cleanup.md).
