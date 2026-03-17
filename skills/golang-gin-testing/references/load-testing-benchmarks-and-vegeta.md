# Load Testing — Go Benchmarks and Vegeta

## Go Benchmark Tests

Use `testing.B` with `httptest` to benchmark handlers without a running server.

```go
// internal/handler/user_handler_bench_test.go
package handler_test

func BenchmarkGetUserHandler(b *testing.B) {
    gin.SetMode(gin.TestMode)
    svc := &mockUserService{
        getByIDFn: func(ctx context.Context, id uint) (*domain.User, error) {
            return &domain.User{ID: id, Name: "Alice"}, nil
        },
    }
    router := gin.New()
    h := NewUserHandler(svc)
    router.GET("/users/:id", h.GetByID)

    b.ReportAllocs()
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        w := httptest.NewRecorder()
        req, _ := http.NewRequest(http.MethodGet, "/users/1", nil)
        router.ServeHTTP(w, req)
    }
}
```

```bash
go test -bench=BenchmarkGetUserHandler -benchmem -count=3 ./internal/handler/...
# BenchmarkGetUserHandler-8   120000   9823 ns/op   2048 B/op   18 allocs/op
```

Flags: `-benchmem` shows B/op + allocs/op; `-count=3` for stable results; `-benchtime=5s` for slow handlers.

---

## Vegeta Load Testing

CLI-based HTTP load tester. Good for quick rate/duration sweeps.

```bash
go install github.com/tsenart/vegeta@latest
```

**GET — 30s at 100 RPS:**
```bash
echo "GET http://localhost:8080/api/v1/users/1" | vegeta attack -duration=30s -rate=100 | vegeta report
```

**POST with JSON body** — create `targets.txt`:
```
POST http://localhost:8080/api/v1/users
Content-Type: application/json
@body.json
```
```bash
vegeta attack -targets=targets.txt -duration=30s -rate=50 | vegeta report
```

**Plot latency histogram:**
```bash
echo "GET http://localhost:8080/api/v1/users/1" \
  | vegeta attack -duration=30s -rate=100 | tee results.bin | vegeta report
vegeta plot results.bin > report.html
```

**Ramp up rate:**
```bash
for rate in 50 100 200 500; do
  echo "GET http://localhost:8080/api/v1/users/1" \
    | vegeta attack -duration=10s -rate=$rate | vegeta report --type=text
done
```

> For k6, performance targets, and CI benchmark regression detection: see [load-testing-k6-and-ci-regression.md](load-testing-k6-and-ci-regression.md).
