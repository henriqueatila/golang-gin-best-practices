# Observability — Docker Compose Stacks and Graceful Shutdown

## Docker Compose — Local Observability Stack (Jaeger)

Add Jaeger and an OTel Collector as a compose overlay. The Collector receives OTLP from the app and forwards to Jaeger.

```yaml
# docker-compose.observability.yml
# docker compose -f docker-compose.yml -f docker-compose.observability.yml up
services:
  otel-collector:
    image: otel/opentelemetry-collector-contrib:0.117.0
    command: ["--config=/etc/otel/config.yaml"]
    volumes: ["./otel-collector-config.yaml:/etc/otel/config.yaml:ro"]
    ports: ["4317:4317", "4318:4318", "8888:8888"]
    depends_on: [jaeger]
  jaeger:
    image: jaegertracing/all-in-one:1.65
    environment: [COLLECTOR_OTLP_ENABLED=true]
    ports: ["16686:16686", "4317"]
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:16686"]
      interval: 10s; timeout: 5s; retries: 5
```

`otel-collector-config.yaml`: receivers `otlp` (grpc: `0.0.0.0:4317`, http: `0.0.0.0:4318`), processors `batch` (timeout: 5s, send_batch_size: 512), exporters `otlp/jaeger` (endpoint: `jaeger:4317`, tls.insecure: true), pipeline `traces + metrics: [otlp] → [batch] → [otlp/jaeger]`.

App env vars: `OTLP_ENDPOINT=otel-collector:4317`, `OTEL_SAMPLING_RATE=1.0`. Jaeger UI: `http://localhost:16686`.

---

## Prometheus + Grafana Stack

Add a `prometheus` exporter to the OTel Collector, then run Prometheus + Grafana as an overlay.

**Collector config diff** — add to `exporters` and `metrics` pipeline:
```yaml
exporters:
  prometheus:
    endpoint: "0.0.0.0:8889"
    namespace: gin_app   # prefix: gin_app_http_request_duration_seconds …
# metrics pipeline: exporters: [otlp/jaeger, prometheus]
```

**`docker-compose.monitoring.yml`** — adds `prom/prometheus:latest` (port 9090, `./prometheus.yml` volume) and `grafana/grafana:latest` (port 3000, `GF_AUTH_ANONYMOUS_ENABLED=true` for local dev, `./grafana/provisioning` volume).

**`prometheus.yml`** scrapes `otel-collector:8888` (collector self-metrics) and `otel-collector:8889` (app metrics).

**Grafana provisioning:** datasource at `grafana/provisioning/datasources/prometheus.yml` points to `http://prometheus:9090`. Dashboard provider loads JSON files from `grafana/provisioning/dashboards/`. Save RED dashboard JSON as `gin-red.json` with three panels:
- Request Rate: `rate(gin_app_http_request_duration_seconds_count[1m])`
- Error Rate %: `rate(...{status_code=~"5.."}[1m]) / rate(...[1m]) * 100`
- P99 Latency: `histogram_quantile(0.99, rate(gin_app_http_request_duration_seconds_bucket[5m]))`

Run: `docker compose -f docker-compose.yml -f docker-compose.monitoring.yml up`. Endpoints: Jaeger `:16686`, Prometheus `:9090`, Grafana `:3000`, collector self-metrics `:8888`, app metrics `:8889`.

---

## Graceful Shutdown

OTel providers buffer telemetry in memory and flush in batches. Skipping shutdown drops the last batch — losing visibility into the final seconds. Kubernetes sends `SIGTERM`, waits (default 30s), then `SIGKILL`.

```go
// cmd/api/main.go — shutdown sequence
quit := make(chan os.Signal, 1)
signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
<-quit

shutdownCtx, cancel := context.WithTimeout(context.Background(), cfg.ShutdownTimeout)
defer cancel()

srv.Shutdown(shutdownCtx)                       // 1. stop accepting new requests
providers.TracerProvider.Shutdown(shutdownCtx)  // 2. flush buffered spans
providers.MeterProvider.Shutdown(shutdownCtx)   // 3. flush buffered metrics
```

**Shutdown order:** HTTP server first (drains in-flight requests) → OTel providers (flushes telemetry from completed requests). Reversing risks flushing incomplete spans.

**Timeout budget:** `ShutdownTimeout` should be 25–28s when K8s `terminationGracePeriodSeconds` is 30s — must cover both HTTP drain and OTel flush.

> For SDK setup, middleware, and manual spans: see [observability-otel-sdk-and-middleware.md](observability-otel-sdk-and-middleware.md).
> For metrics, log-trace correlation, and sampling: see [observability-metrics-logs-sampling.md](observability-metrics-logs-sampling.md).
