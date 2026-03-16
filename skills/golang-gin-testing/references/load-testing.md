# Load Testing

Load and performance testing for Go Gin APIs: Go benchmarks, vegeta, k6, performance targets, and CI regression detection.

## Go Benchmark Tests

Use `testing.B` with `httptest` to benchmark handlers without a running server.

```go
// internal/handler/user_handler_bench_test.go
package handler_test

import (
    "net/http"
    "net/http/httptest"
    "testing"

    "github.com/gin-gonic/gin"
)

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

Run benchmarks:

```bash
go test -bench=BenchmarkGetUserHandler -benchmem -count=3 ./internal/handler/...
```

Example output:

```
BenchmarkGetUserHandler-8   120000   9823 ns/op   2048 B/op   18 allocs/op
```

- `-benchmem` shows allocations per operation (`B/op`, `allocs/op`)
- `-count=3` runs the benchmark 3 times for stable results
- `-benchtime=5s` extends run duration for slow handlers

## Vegeta Load Testing

CLI-based HTTP load tester. Good for quick rate/duration sweeps.

**Install:**

```bash
go install github.com/tsenart/vegeta@latest
```

**GET endpoint — 30s at 100 RPS:**

```bash
echo "GET http://localhost:8080/api/v1/users/1" \
  | vegeta attack -duration=30s -rate=100 \
  | vegeta report
```

**POST with JSON body:**

```bash
# targets.txt
POST http://localhost:8080/api/v1/users
Content-Type: application/json
@body.json
```

```bash
vegeta attack -targets=targets.txt -duration=30s -rate=50 | vegeta report
```

**Plot latency histogram (outputs SVG):**

```bash
echo "GET http://localhost:8080/api/v1/users/1" \
  | vegeta attack -duration=30s -rate=100 \
  | tee results.bin \
  | vegeta report

vegeta plot results.bin > report.html
```

**Ramp up rate:**

```bash
for rate in 50 100 200 500; do
  echo "GET http://localhost:8080/api/v1/users/1" \
    | vegeta attack -duration=10s -rate=$rate \
    | vegeta report --type=text
done
```

## k6 Load Testing

Script-based load tester with thresholds and stages. Good for realistic traffic patterns.

**Install:**

```bash
# macOS
brew install k6
# Linux
sudo snap install k6
```

**Basic GET/POST script:**

```js
// load-test.js
import http from "k6/http";
import { check, sleep } from "k6";

export const options = {
  stages: [
    { duration: "30s", target: 50 },   // ramp up to 50 VUs
    { duration: "1m",  target: 50 },   // sustain
    { duration: "15s", target: 0 },    // ramp down
  ],
  thresholds: {
    http_req_duration: ["p(95)<200", "p(99)<500"], // latency gates
    http_req_failed:   ["rate<0.01"],              // <1% error rate
  },
};

export default function () {
  // GET
  const res = http.get("http://localhost:8080/api/v1/users/1", {
    headers: { Authorization: `Bearer ${__ENV.TOKEN}` },
  });
  check(res, { "status 200": (r) => r.status === 200 });

  // POST
  const payload = JSON.stringify({ name: "Alice", email: "alice@example.com" });
  const postRes = http.post("http://localhost:8080/api/v1/users", payload, {
    headers: { "Content-Type": "application/json" },
  });
  check(postRes, { "status 201": (r) => r.status === 201 });

  sleep(1);
}
```

**Run:**

```bash
TOKEN=your-jwt k6 run load-test.js
```

k6 exits non-zero if any threshold is breached — suitable for CI gates.

## Performance Targets

Typical targets for a single Go Gin instance on commodity hardware (2 vCPU / 4 GB RAM):

| Metric | Target |
|--------|--------|
| p50 latency | < 50 ms |
| p95 latency | < 200 ms |
| p99 latency | < 500 ms |
| Error rate | < 0.1% |
| RPS per instance | > 1 000 |

Adjust targets based on: external I/O (DB, Redis), payload size, auth middleware overhead.

## CI Integration — Benchmark Regression Detection

Use `benchstat` to catch performance regressions between commits.

**Install:**

```bash
go install golang.org/x/perf/cmd/benchstat@latest
```

**GitHub Actions example:**

```yaml
# .github/workflows/benchmarks.yml
name: Benchmarks

on: [push, pull_request]

jobs:
  benchmark:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }

      - uses: actions/setup-go@v5
        with: { go-version: "1.22" }

      - name: Run benchmarks (HEAD)
        run: go test -bench=. -benchmem -count=6 ./... | tee bench-head.txt

      - name: Checkout base branch
        run: git checkout ${{ github.base_ref }}

      - name: Run benchmarks (base)
        run: go test -bench=. -benchmem -count=6 ./... | tee bench-base.txt

      - name: Compare with benchstat
        run: |
          benchstat bench-base.txt bench-head.txt
          # Fail if any benchmark regressed by >10%
          benchstat -html bench-base.txt bench-head.txt > bench-report.html

      - uses: actions/upload-artifact@v4
        with:
          name: bench-report
          path: bench-report.html
```

**Interpreting benchstat output:**

```
name               old time/op  new time/op  delta
GetUserHandler-8   9.82µs ± 2%  8.91µs ± 1%  -9.26%  (p=0.008 n=6+6)
```

- `delta` shows change; negative is an improvement
- `p=` is the p-value — values below 0.05 indicate a statistically significant change
- `-count=6` gives benchstat enough samples for reliable statistics
