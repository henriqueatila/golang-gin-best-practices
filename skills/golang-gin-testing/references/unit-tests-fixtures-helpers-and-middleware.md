# Unit Tests — Fixtures, t.Helper/Cleanup/Parallel, and Middleware Testing

## Test Fixtures and Factories

Factories build valid domain objects in one line. Keeps tests focused on what varies, not boilerplate.

```go
// internal/testutil/fixtures.go
package testutil

// UserFixture returns a valid User. Override fields in the caller as needed.
func UserFixture(overrides ...func(*domain.User)) *domain.User {
    u := &domain.User{
        ID: "fixture-user-id", Name: "Test User", Email: "test@example.com", Role: "user",
        CreatedAt: time.Date(2024, 1, 1, 0, 0, 0, 0, time.UTC),
        UpdatedAt: time.Date(2024, 1, 1, 0, 0, 0, 0, time.UTC),
    }
    for _, fn := range overrides { fn(u) }
    return u
}

// CreateUserRequestFixture returns a valid CreateUserRequest.
func CreateUserRequestFixture(overrides ...func(*domain.CreateUserRequest)) domain.CreateUserRequest {
    req := domain.CreateUserRequest{Name: "Test User", Email: "test@example.com", Password: "secret1234", Role: "user"}
    for _, fn := range overrides { fn(&req) }
    return req
}
```

Usage — only specify what differs:

```go
adminUser := testutil.UserFixture(func(u *domain.User) { u.Role = "admin"; u.Email = "admin@example.com" })
noRoleReq := testutil.CreateUserRequestFixture(func(r *domain.CreateUserRequest) { r.Role = "" })
```

---

## t.Helper / t.Cleanup / t.Parallel

```go
// t.Helper — failures show the caller's line, not the helper's
func assertStatus(t *testing.T, w *httptest.ResponseRecorder, want int) {
    t.Helper()
    if w.Code != want { t.Errorf("want status %d, got %d; body: %s", want, w.Code, w.Body) }
}

// t.Cleanup — deferred cleanup that runs even if the test panics
func createTempFile(t *testing.T) string {
    t.Helper()
    f, err := os.CreateTemp("", "test-*")
    if err != nil { t.Fatalf("createTempFile: %v", err) }
    t.Cleanup(func() { os.Remove(f.Name()) })
    return f.Name()
}

// t.Parallel — safe for stateless tests; do NOT use with shared mutable state
func TestHandlerConcurrency(t *testing.T) { t.Parallel(); /* ... */ }
```

**When NOT to use `t.Parallel()`:** tests sharing a database, writing to files, or mutating package-level state.

---

## Testing Middleware in Isolation

Test middleware logic (JWT validation, RBAC) independently of any handler.

```go
// pkg/middleware/auth_test.go
package middleware_test

var sentinelHandler = func(c *gin.Context) { c.JSON(http.StatusOK, gin.H{"reached": true}) }

func setupAuthMiddlewareRouter(cfg auth.TokenConfig) *gin.Engine {
    r := gin.New()
    r.Use(middleware.Auth(cfg, slog.Default()))
    r.GET("/protected", sentinelHandler)
    return r
}

func TestAuthMiddleware_RejectsNoHeader(t *testing.T) {
    cfg := auth.TokenConfig{AccessSecret: []byte("test-secret-32-bytes-exactly-!!!"), AccessTTL: 15 * time.Minute}
    req := httptest.NewRequest(http.MethodGet, "/protected", nil)
    w := httptest.NewRecorder()
    setupAuthMiddlewareRouter(cfg).ServeHTTP(w, req)
    if w.Code != http.StatusUnauthorized { t.Errorf("want 401, got %d", w.Code) }
}

func TestAuthMiddleware_RejectsMalformedHeader(t *testing.T) {
    cfg := auth.TokenConfig{AccessSecret: []byte("test-secret-32-bytes-exactly-!!!")}
    router := setupAuthMiddlewareRouter(cfg)
    for _, h := range []string{"token-without-bearer", "Basic abc123", "Bearer", ""} {
        t.Run(h, func(t *testing.T) {
            req := httptest.NewRequest(http.MethodGet, "/protected", nil)
            if h != "" { req.Header.Set("Authorization", h) }
            w := httptest.NewRecorder()
            router.ServeHTTP(w, req)
            if w.Code != http.StatusUnauthorized { t.Errorf("header=%q: want 401, got %d", h, w.Code) }
        })
    }
}

func TestAuthMiddleware_InjectsClaimsOnSuccess(t *testing.T) {
    cfg := auth.TokenConfig{AccessSecret: []byte("test-secret-32-bytes-exactly-!!!"), AccessTTL: 15 * time.Minute}
    var capturedUserID string
    r := gin.New()
    r.Use(middleware.Auth(cfg, slog.Default()))
    r.GET("/me", func(c *gin.Context) { capturedUserID = c.GetString(middleware.UserIDKey); c.JSON(http.StatusOK, gin.H{"ok": true}) })
    token, err := auth.GenerateAccessToken(cfg, "user-xyz", "u@example.com", "user")
    if err != nil { t.Fatal(err) }
    req := httptest.NewRequest(http.MethodGet, "/me", nil)
    req.Header.Set("Authorization", "Bearer "+token)
    w := httptest.NewRecorder()
    r.ServeHTTP(w, req)
    if w.Code != http.StatusOK { t.Fatalf("want 200, got %d; body: %s", w.Code, w.Body) }
    if capturedUserID != "user-xyz" { t.Errorf("want user_id 'user-xyz', got %q", capturedUserID) }
}
```

> For benchmarks, fuzz tests, golden files, and test organization: see [unit-tests-benchmarks-fuzz-and-golden.md](unit-tests-benchmarks-fuzz-and-golden.md).
