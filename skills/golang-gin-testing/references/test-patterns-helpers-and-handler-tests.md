# Test Patterns — Test Helpers and Handler Tests

## Test Helpers

Define reusable helpers in `internal/testutil/` to keep tests DRY.

```go
// internal/testutil/helpers.go
package testutil

func init() { gin.SetMode(gin.TestMode) }

// NewTestRouter creates a bare Gin engine (no Logger/Recovery noise).
func NewTestRouter() *gin.Engine { return gin.New() }

// PerformRequest executes an HTTP request against the router and returns the recorder.
func PerformRequest(t *testing.T, router *gin.Engine, method, path string, body any, headers map[string]string) *httptest.ResponseRecorder {
    t.Helper()
    var reqBody *bytes.Buffer
    if body != nil {
        b, err := json.Marshal(body)
        if err != nil { t.Fatalf("PerformRequest: marshal body: %v", err) }
        reqBody = bytes.NewBuffer(b)
    } else {
        reqBody = bytes.NewBuffer(nil)
    }
    req, err := http.NewRequest(method, path, reqBody)
    if err != nil { t.Fatalf("PerformRequest: create request: %v", err) }
    if body != nil { req.Header.Set("Content-Type", "application/json") }
    for k, v := range headers { req.Header.Set(k, v) }
    w := httptest.NewRecorder()
    router.ServeHTTP(w, req)
    return w
}

// AssertJSON unmarshals the recorder body into dst, failing the test on error.
func AssertJSON(t *testing.T, w *httptest.ResponseRecorder, dst any) {
    t.Helper()
    if err := json.Unmarshal(w.Body.Bytes(), dst); err != nil {
        t.Fatalf("AssertJSON: unmarshal %q: %v", w.Body.String(), err)
    }
}

// BearerHeader returns an Authorization header map for JWT-protected routes.
func BearerHeader(token string) map[string]string {
    return map[string]string{"Authorization": "Bearer " + token}
}
```

---

## Handler Tests with httptest

Wire a real router with a mock service; use `httptest.NewRecorder` + `router.ServeHTTP`.

```go
// mockUserService implements service.UserService for tests.
type mockUserService struct {
    createFn  func(ctx context.Context, req domain.CreateUserRequest) (*domain.User, error)
    getByIDFn func(ctx context.Context, id string) (*domain.User, error)
}
func (m *mockUserService) Create(ctx context.Context, req domain.CreateUserRequest) (*domain.User, error) { return m.createFn(ctx, req) }
func (m *mockUserService) GetByID(ctx context.Context, id string) (*domain.User, error) { return m.getByIDFn(ctx, id) }

func TestUserHandler_Create_Success(t *testing.T) {
    svc := &mockUserService{
        createFn: func(_ context.Context, req domain.CreateUserRequest) (*domain.User, error) {
            return &domain.User{ID: "user-123", Name: req.Name, Email: req.Email, Role: "user"}, nil
        },
    }
    router := setupUserRouter(svc)
    w := testutil.PerformRequest(t, router, http.MethodPost, "/users",
        map[string]any{"name": "Alice", "email": "alice@example.com", "password": "secret123"}, nil)
    if w.Code != http.StatusCreated { t.Errorf("want 201, got %d; body: %s", w.Code, w.Body) }
    var got domain.User
    testutil.AssertJSON(t, w, &got)
    if got.ID != "user-123" { t.Errorf("want user-123, got %q", got.ID) }
}
```

---

## Table-Driven Handler Tests

```go
func TestUserHandler_Create_Validation(t *testing.T) {
    svc := &mockUserService{createFn: func(_ context.Context, req domain.CreateUserRequest) (*domain.User, error) {
        return &domain.User{ID: "1", Name: req.Name, Email: req.Email, Role: "user"}, nil
    }}
    router := setupUserRouter(svc)

    tests := []struct{ name string; body any; wantStatus int }{
        {"valid request",       map[string]any{"name": "Alice", "email": "alice@example.com", "password": "secret123"}, http.StatusCreated},
        {"missing email",       map[string]any{"name": "Alice", "password": "secret123"}, http.StatusBadRequest},
        {"invalid email",       map[string]any{"name": "Alice", "email": "not-an-email", "password": "secret123"}, http.StatusBadRequest},
        {"name too short",      map[string]any{"name": "A", "email": "alice@example.com", "password": "secret123"}, http.StatusBadRequest},
        {"password too short",  map[string]any{"name": "Alice", "email": "alice@example.com", "password": "short"}, http.StatusBadRequest},
        {"invalid role",        map[string]any{"name": "Alice", "email": "alice@example.com", "password": "secret123", "role": "superadmin"}, http.StatusBadRequest},
    }

    for _, tc := range tests {
        t.Run(tc.name, func(t *testing.T) {
            t.Parallel()
            w := testutil.PerformRequest(t, router, http.MethodPost, "/users", tc.body, nil)
            if w.Code != tc.wantStatus {
                t.Errorf("want %d, got %d; body: %s", tc.wantStatus, w.Code, w.Body)
            }
        })
    }
}
```

> For service tests with mocked repository and running tests: see [test-patterns-service-tests-and-running.md](test-patterns-service-tests-and-running.md).
