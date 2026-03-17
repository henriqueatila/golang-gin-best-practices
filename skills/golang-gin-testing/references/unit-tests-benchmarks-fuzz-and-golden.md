# Unit Tests — Benchmarks, Fuzz Tests, Golden Files, Organization, and Testify

## Benchmark Tests

Use `Benchmark*` with `httptest` to measure handler throughput. Run with `go test -bench=. -benchmem ./...`.

```go
func BenchmarkUserHandler_GetByID(b *testing.B) {
    svc := &mockUserService{getByIDFn: func(_ context.Context, id string) (*domain.User, error) {
        return &domain.User{ID: id, Name: "Alice", Email: "alice@example.com", Role: "user"}, nil
    }}
    router := setupUserRouter(svc)
    b.ReportAllocs(); b.ResetTimer()
    for b.Loop() {
        w := httptest.NewRecorder()
        req, _ := http.NewRequest(http.MethodGet, "/users/user-123", nil)
        router.ServeHTTP(w, req)
        if w.Code != http.StatusOK { b.Fatalf("unexpected status %d", w.Code) }
    }
}

func BenchmarkUserHandler_GetByID_Parallel(b *testing.B) {
    // same setup...
    b.ReportAllocs(); b.ResetTimer()
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            w := httptest.NewRecorder()
            req, _ := http.NewRequest(http.MethodGet, "/users/user-123", nil)
            router.ServeHTTP(w, req)
        }
    })
}
```

```bash
go test -bench=BenchmarkUserHandler_GetByID -benchtime=5s -benchmem ./internal/handler/...
go test -bench=. -benchmem -count=5 ./... > before.txt && benchstat before.txt after.txt
```

---

## Fuzz Tests

Go 1.18+ native fuzzing finds edge cases in input parsing that table-driven tests miss.

```go
// go test -fuzz=FuzzUserHandler_Create ./internal/handler/...  (local fuzzing)
// go test ./internal/handler/...                               (seed corpus only, CI-safe)
func FuzzUserHandler_Create(f *testing.F) {
    f.Add(`{"name":"Alice","email":"alice@example.com","password":"secret123"}`)
    f.Add(`{"name":"","email":"bad-email","password":"x"}`)
    f.Add(`{}`); f.Add(`not-json`); f.Add(`{"name":null,"email":null}`)

    svc := &mockUserService{createFn: func(_ context.Context, req domain.CreateUserRequest) (*domain.User, error) {
        return &domain.User{ID: "fuzz-id", Name: req.Name, Email: req.Email, Role: "user"}, nil
    }}
    router := setupUserRouter(svc)
    f.Fuzz(func(t *testing.T, body string) {
        req := httptest.NewRequest(http.MethodPost, "/users", strings.NewReader(body))
        req.Header.Set("Content-Type", "application/json")
        w := httptest.NewRecorder()
        router.ServeHTTP(w, req)
        if w.Code == http.StatusInternalServerError {
            t.Errorf("handler returned 500 for input %q; body: %s", body, w.Body)
        }
    })
}
```

Rules: seed corpus must include valid + edge-case inputs; no non-determinism (no random, no time); found crash inputs auto-saved to `testdata/fuzz/FuzzX/`.

---

## Golden File / Snapshot Testing

Captures expected output as a file. On subsequent runs, output is compared to the stored file.

```go
var update = flag.Bool("update", false, "update golden files")

func TestUserResponse_Golden(t *testing.T) {
    user := domain.User{ID: "test-id", Email: "test@example.com", Name: "Test User"}
    body, err := json.MarshalIndent(user, "", "  ")
    require.NoError(t, err)
    golden := filepath.Join("testdata", t.Name()+".golden")
    if *update { os.MkdirAll("testdata", 0o755); os.WriteFile(golden, body, 0o644) }
    expected, err := os.ReadFile(golden)
    require.NoError(t, err)
    assert.JSONEq(t, string(expected), string(body))
}
```

First run: `go test ./... -update` writes the golden file. Commit it — diffs surface unintended response changes. Re-run with `-update` when the shape intentionally changes.

---

## Test Organization

**Same-package** (`package service`) — white-box: access unexported identifiers, verify internal state.
**External-package** (`package handler_test`) — black-box: only exported API visible; prevents coupling to internals. Use for handlers, middleware, repositories.

**Build tags:**
```go
//go:build integration   // top of file, before package declaration
// Excluded from: go test ./...
// Included in:   go test -tags=integration ./...
```

Use `//go:build integration` for Docker/network tests; `//go:build e2e` for full E2E. Keeps `go test ./...` fast.

---

## Testify as an Alternative

```go
// Standard library
if got.Email != "alice@example.com" { t.Errorf("want email 'alice@example.com', got %q", got.Email) }
if err != nil { t.Fatalf("unexpected error: %v", err) }

// testify/assert — non-fatal, test continues
assert.Equal(t, "alice@example.com", got.Email); assert.NoError(t, err)

// testify/require — fatal, stops immediately (equivalent to t.Fatalf)
require.NoError(t, err); require.Equal(t, "alice@example.com", got.Email)
```

| Package | Behavior | Use when |
|---------|----------|----------|
| `assert` | non-fatal | checking multiple independent fields |
| `require` | fatal | precondition must hold for rest of test |

`require.NoError(t, err)` before accessing the result is the most common pattern.

> For handler testing: see [unit-tests-handler-and-json.md](unit-tests-handler-and-json.md).
> For service mocks and table-driven tests: see [unit-tests-service-mocks-and-table-driven.md](unit-tests-service-mocks-and-table-driven.md).
