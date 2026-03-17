# Load Testing — k6, Performance Targets, and CI Benchmark Regression

## k6 Load Testing

Script-based load tester with thresholds and stages. Good for realistic traffic patterns.

```bash
brew install k6        # macOS
sudo snap install k6   # Linux
```

```js
// load-test.js
import http from "k6/http";
import { check, sleep } from "k6";

export const options = {
  stages: [
    { duration: "30s", target: 50 },  // ramp up to 50 VUs
    { duration: "1m",  target: 50 },  // sustain
    { duration: "15s", target: 0 },   // ramp down
  ],
  thresholds: {
    http_req_duration: ["p(95)<200", "p(99)<500"], // latency gates
    http_req_failed:   ["rate<0.01"],              // <1% error rate
  },
};

export default function () {
  const res = http.get("http://localhost:8080/api/v1/users/1", {
    headers: { Authorization: `Bearer ${__ENV.TOKEN}` },
  });
  check(res, { "status 200": (r) => r.status === 200 });

  const payload = JSON.stringify({ name: "Alice", email: "alice@example.com" });
  const postRes = http.post("http://localhost:8080/api/v1/users", payload, {
    headers: { "Content-Type": "application/json" },
  });
  check(postRes, { "status 201": (r) => r.status === 201 });

  sleep(1);
}
```

```bash
TOKEN=your-jwt k6 run load-test.js
```

k6 exits non-zero if any threshold is breached — suitable for CI gates.

---

## Performance Targets

Typical targets for a single Go Gin instance on commodity hardware (2 vCPU / 4 GB RAM):

| Metric | Target |
|--------|--------|
| p50 latency | < 50 ms |
| p95 latency | < 200 ms |
| p99 latency | < 500 ms |
| Error rate | < 0.1% |
| RPS per instance | > 1 000 |

Adjust based on: external I/O (DB, Redis), payload size, auth middleware overhead.

---

## CI Integration — Benchmark Regression Detection

Use `benchstat` to catch performance regressions between commits.

```bash
go install golang.org/x/perf/cmd/benchstat@latest
```

```yaml
# .github/workflows/benchmarks.yml
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
          benchstat -html bench-base.txt bench-head.txt > bench-report.html
      - uses: actions/upload-artifact@v4
        with: { name: bench-report, path: bench-report.html }
```

**Interpreting benchstat:**
```
name               old time/op  new time/op  delta
GetUserHandler-8   9.82µs ± 2%  8.91µs ± 1%  -9.26%  (p=0.008 n=6+6)
```
- Negative `delta` = improvement; `p<0.05` = statistically significant change
- Use `-count=6` for reliable statistics

> For Go benchmarks and Vegeta: see [load-testing-benchmarks-and-vegeta.md](load-testing-benchmarks-and-vegeta.md).
