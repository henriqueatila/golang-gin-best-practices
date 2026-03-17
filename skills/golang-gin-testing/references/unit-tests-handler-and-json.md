# Unit Tests — Handler Testing, JSON Responses, and Authenticated Routes

All examples use the `User` domain model and `AppError` pattern from **golang-gin-api**.

## Handler Testing with httptest

Test the HTTP contract (status codes, JSON shape, headers) — not business logic. Business logic belongs in service tests.

```go
// internal/handler/user_handler_test.go
package handler_test

func init() { gin.SetMode(gin.TestMode) }

func setupUserRouter(svc service.UserService) *gin.Engine {
    r := gin.New() // gin.New(), not gin.Default() — no Logger noise in test output
    h := handler.NewUserHandler(svc, slog.Default())
    r.POST("/users", h.Create); r.GET("/users/:id", h.GetByID)
    r.PUT("/users/:id", h.Update); r.DELETE("/users/:id", h.Delete)
    return r
}

func TestUserHandler_GetByID(t *testing.T) {
    svc := &mockUserService{
        getByIDFn: func(_ context.Context, id string) (*domain.User, error) {
            if id == "user-123" {
                return &domain.User{ID: "user-123", Name: "Alice", Email: "alice@example.com", Role: "user"}, nil
            }
            return nil, domain.ErrNotFound
        },
    }
    req := httptest.NewRequest(http.MethodGet, "/users/user-123", nil)
    w := httptest.NewRecorder()
    setupUserRouter(svc).ServeHTTP(w, req)
    if w.Code != http.StatusOK { t.Fatalf("want 200, got %d; body: %s", w.Code, w.Body) }
    if ct := w.Header().Get("Content-Type"); !strings.Contains(ct, "application/json") {
        t.Errorf("want Content-Type application/json, got %q", ct)
    }
}
```

---

## Testing JSON and Error Responses

```go
func TestUserHandler_Create_ReturnsUserJSON(t *testing.T) {
    svc := &mockUserService{createFn: func(_ context.Context, req domain.CreateUserRequest) (*domain.User, error) {
        return &domain.User{ID: "new-id", Name: req.Name, Email: req.Email, Role: "user"}, nil
    }}
    body := `{"name":"Bob","email":"bob@example.com","password":"secret123"}`
    req := httptest.NewRequest(http.MethodPost, "/users", strings.NewReader(body))
    req.Header.Set("Content-Type", "application/json")
    w := httptest.NewRecorder()
    setupUserRouter(svc).ServeHTTP(w, req)
    if w.Code != http.StatusCreated { t.Fatalf("want 201, got %d; body: %s", w.Code, w.Body) }
    var resp domain.User
    if err := json.Unmarshal(w.Body.Bytes(), &resp); err != nil { t.Fatalf("parse JSON: %v", err) }
    if resp.ID != "new-id" { t.Errorf("want ID 'new-id', got %q", resp.ID) }
    if strings.Contains(w.Body.String(), "secret123") { t.Error("response must not contain plain-text password") }
}

func TestUserHandler_GetByID_ErrorResponse(t *testing.T) {
    svc := &mockUserService{getByIDFn: func(_ context.Context, id string) (*domain.User, error) {
        return nil, domain.ErrNotFound
    }}
    req := httptest.NewRequest(http.MethodGet, "/users/ghost", nil)
    w := httptest.NewRecorder()
    setupUserRouter(svc).ServeHTTP(w, req)
    if w.Code != http.StatusNotFound { t.Fatalf("want 404, got %d", w.Code) }
    var errResp map[string]string
    if err := json.Unmarshal(w.Body.Bytes(), &errResp); err != nil { t.Fatalf("parse error JSON: %v", err) }
    if errResp["error"] == "" { t.Error("error response must contain 'error' field") }
}
```

---

## Testing Authenticated Routes

Inject a real JWT token generated with the test secret, apply `Auth` middleware in the test router.

```go
func testTokenConfig() auth.TokenConfig {
    return auth.TokenConfig{
        AccessSecret: []byte("test-access-secret-32-bytes-long!!"),
        AccessTTL: 15 * time.Minute,
    }
}
func generateTestToken(t *testing.T, userID, email, role string) string {
    t.Helper()
    token, err := auth.GenerateAccessToken(testTokenConfig(), userID, email, role)
    if err != nil { t.Fatalf("generate test token: %v", err) }
    return token
}
func setupProtectedRouter(svc service.UserService) *gin.Engine {
    r := gin.New()
    protected := r.Group("")
    protected.Use(middleware.Auth(testTokenConfig(), slog.Default()))
    protected.GET("/users/:id", handler.NewUserHandler(svc, slog.Default()).GetByID)
    return r
}

func TestProtectedRoute_WithValidToken(t *testing.T) {
    svc := &mockUserService{getByIDFn: func(_ context.Context, id string) (*domain.User, error) {
        return &domain.User{ID: id, Name: "Alice", Email: "alice@example.com", Role: "user"}, nil
    }}
    req := httptest.NewRequest(http.MethodGet, "/users/user-123", nil)
    req.Header.Set("Authorization", "Bearer "+generateTestToken(t, "user-123", "alice@example.com", "user"))
    w := httptest.NewRecorder()
    setupProtectedRouter(svc).ServeHTTP(w, req)
    if w.Code != http.StatusOK { t.Errorf("want 200, got %d; body: %s", w.Code, w.Body) }
}

func TestProtectedRoute_MissingToken(t *testing.T) {
    req := httptest.NewRequest(http.MethodGet, "/users/user-123", nil)
    w := httptest.NewRecorder()
    setupProtectedRouter(&mockUserService{}).ServeHTTP(w, req)
    if w.Code != http.StatusUnauthorized { t.Errorf("want 401, got %d", w.Code) }
}

func TestProtectedRoute_ExpiredToken(t *testing.T) {
    cfg := auth.TokenConfig{AccessSecret: []byte("test-access-secret-32-bytes-long!!"), AccessTTL: -1 * time.Minute}
    token, _ := auth.GenerateAccessToken(cfg, "u1", "a@b.com", "user")
    req := httptest.NewRequest(http.MethodGet, "/users/u1", nil)
    req.Header.Set("Authorization", "Bearer "+token)
    w := httptest.NewRecorder()
    setupProtectedRouter(&mockUserService{}).ServeHTTP(w, req)
    if w.Code != http.StatusUnauthorized { t.Errorf("want 401 for expired token, got %d", w.Code) }
}
```

> For service mocks, table-driven tests, fixtures, middleware, benchmarks, fuzz, golden files: see [unit-tests-service-mocks-and-table-driven.md](unit-tests-service-mocks-and-table-driven.md), [unit-tests-fixtures-helpers-and-middleware.md](unit-tests-fixtures-helpers-and-middleware.md), [unit-tests-benchmarks-fuzz-and-golden.md](unit-tests-benchmarks-fuzz-and-golden.md).
