# Observability — Metrics Middleware, Log-Trace Correlation, and Sampling

## Metrics Middleware

OTel metrics expose RED metrics (Rate, Errors, Duration) per route. Custom middleware gives control over metric names, label cardinality, and histogram bucket boundaries.

```go
// internal/middleware/metrics_middleware.go
type HTTPMetrics struct {
    requestDuration metric.Float64Histogram
    requestsTotal   metric.Int64Counter
}

func NewHTTPMetrics() (*HTTPMetrics, error) {
    meter := otel.Meter("myapp/http")
    duration, err := meter.Float64Histogram("http.server.request.duration",
        metric.WithDescription("HTTP request duration in seconds"), metric.WithUnit("s"),
        metric.WithExplicitBucketBoundaries(0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10),
    )
    if err != nil { return nil, err }
    total, err := meter.Int64Counter("http.server.requests.total",
        metric.WithDescription("Total HTTP requests"))
    if err != nil { return nil, err }
    return &HTTPMetrics{requestDuration: duration, requestsTotal: total}, nil
}

func (m *HTTPMetrics) Handler() gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()
        c.Next()
        attrs := []attribute.KeyValue{
            attribute.String("http.method", c.Request.Method),
            attribute.String("http.route", c.FullPath()), // pattern, not resolved path
            attribute.String("http.status_code", strconv.Itoa(c.Writer.Status())),
        }
        m.requestDuration.Record(c.Request.Context(), time.Since(start).Seconds(), metric.WithAttributes(attrs...))
        m.requestsTotal.Add(c.Request.Context(), 1, metric.WithAttributes(attrs...))
    }
}
```

Register: `httpMetrics, _ := middleware.NewHTTPMetrics(); r.Use(httpMetrics.Handler())`.

**Cardinality warning:** Never use `c.Request.URL.Path` as a label — unbounded cardinality. Always use `c.FullPath()` (route pattern like `/users/:id`).

---

## Log-Trace Correlation

Inject `trace_id` and `span_id` into every log record to jump from a log line to the full trace in Jaeger.

```go
// internal/telemetry/trace_log_handler.go
type TraceLogHandler struct{ inner slog.Handler }

func NewTraceLogHandler(inner slog.Handler) *TraceLogHandler { return &TraceLogHandler{inner: inner} }

func (h *TraceLogHandler) Handle(ctx context.Context, r slog.Record) error {
    if span := trace.SpanFromContext(ctx); span.SpanContext().IsValid() {
        sc := span.SpanContext()
        r.AddAttrs(slog.String("trace_id", sc.TraceID().String()), slog.String("span_id", sc.SpanID().String()))
    }
    return h.inner.Handle(ctx, r)
}
func (h *TraceLogHandler) Enabled(ctx context.Context, level slog.Level) bool { return h.inner.Enabled(ctx, level) }
func (h *TraceLogHandler) WithAttrs(attrs []slog.Attr) slog.Handler { return &TraceLogHandler{inner: h.inner.WithAttrs(attrs)} }
func (h *TraceLogHandler) WithGroup(name string) slog.Handler { return &TraceLogHandler{inner: h.inner.WithGroup(name)} }
```

Initialize: `logger := slog.New(telemetry.NewTraceLogHandler(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelInfo})))`.

Use `logger.ErrorContext(c.Request.Context(), ...)` in handlers — `trace_id` and `span_id` are injected automatically.

---

## Sampling

`TraceIDRatioBased` is head-based (decision at trace start). Use `ParentBased` to respect upstream sampling decisions from W3C `traceparent` headers.

```go
// internal/telemetry/sampler.go
func SamplerFromEnv() sdktrace.Sampler {
    rate := 1.0
    if v := os.Getenv("OTEL_SAMPLING_RATE"); v != "" {
        if parsed, err := strconv.ParseFloat(v, 64); err == nil { rate = parsed }
    }
    return sdktrace.ParentBased(sdktrace.TraceIDRatioBased(rate))
}
```

Use in `Init`: `sdktrace.WithSampler(SamplerFromEnv())`.

| Environment | `OTEL_SAMPLING_RATE` | Rationale |
|-------------|----------------------|-----------|
| Development | `1.0` (default) | See every trace while debugging |
| Staging | `0.5` | 50% — catch issues without full cost |
| Production | `0.1` | 10% — statistically representative |

For tail-based sampling (keep all error traces, sample successes), use an OTel Collector with the `tail_sampling` processor.

> For SDK setup, Gin middleware, and manual spans: see [observability-otel-sdk-and-middleware.md](observability-otel-sdk-and-middleware.md).
> For Docker Compose observability stack and graceful shutdown: see [observability-docker-stack-and-shutdown.md](observability-docker-stack-and-shutdown.md).
