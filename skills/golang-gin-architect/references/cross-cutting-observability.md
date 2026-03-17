# Cross-Cutting Concerns — Observability

Structured logging, metrics, distributed tracing, and health checks for Go Gin APIs.

---

## Structured Logging

Use `log/slog` (stdlib since Go 1.21). JSON output in production, text in development.

```go
func NewLogger(env string) *slog.Logger {
    var handler slog.Handler
    if env == "production" {
        handler = slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelInfo})
    } else {
        handler = slog.NewTextHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelDebug})
    }
    return slog.New(handler)
}
```

**Request logging middleware:**

```go
func RequestLogger(logger *slog.Logger) gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()
        path := c.Request.URL.Path
        query := c.Request.URL.RawQuery
        c.Next()
        latency := time.Since(start)
        status := c.Writer.Status()

        attrs := []slog.Attr{
            slog.String("method", c.Request.Method),
            slog.String("path", path),
            slog.String("query", query),
            slog.Int("status", status),
            slog.Duration("latency", latency),
            slog.String("ip", c.ClientIP()),
        }
        if reqID := c.GetHeader("X-Request-ID"); reqID != "" {
            attrs = append(attrs, slog.String("request_id", reqID))
        }
        if userID, exists := c.Get("user_id"); exists {
            attrs = append(attrs, slog.Any("user_id", userID))
        }

        level := slog.LevelInfo
        if status >= 500 {
            level = slog.LevelError
        } else if status >= 400 {
            level = slog.LevelWarn
        }
        logger.LogAttrs(c.Request.Context(), level, "request", attrs...)
    }
}
```

---

## Metrics

Use `prometheus/client_golang`. Expose at `/metrics`.

```go
var (
    httpRequestsTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{Name: "http_requests_total", Help: "Total HTTP requests"},
        []string{"method", "path", "status"},
    )
    httpRequestDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{Name: "http_request_duration_seconds", Buckets: prometheus.DefBuckets},
        []string{"method", "path"},
    )
    dbQueryDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "db_query_duration_seconds",
            Buckets: []float64{.001, .005, .01, .025, .05, .1, .25, .5, 1},
        },
        []string{"query"},
    )
)

func init() {
    prometheus.MustRegister(httpRequestsTotal, httpRequestDuration, dbQueryDuration)
}

func MetricsMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()
        c.Next()
        duration := time.Since(start).Seconds()
        status := strconv.Itoa(c.Writer.Status())
        httpRequestsTotal.WithLabelValues(c.Request.Method, c.FullPath(), status).Inc()
        httpRequestDuration.WithLabelValues(c.Request.Method, c.FullPath()).Observe(duration)
    }
}

r.GET("/metrics", gin.WrapH(promhttp.Handler()))
```

**Key metrics:** `http_requests_total`, `http_request_duration_seconds` (p50/p95/p99), `db_query_duration_seconds`, business metrics like `orders_created_total`.

---

## Distributed Tracing

Use OpenTelemetry when you have 2+ services communicating. NOT needed for a monolith.

```go
func initTracer(ctx context.Context, serviceName string) (*trace.TracerProvider, error) {
    exporter, err := otlptracehttp.New(ctx)
    if err != nil {
        return nil, fmt.Errorf("create exporter: %w", err)
    }
    tp := trace.NewTracerProvider(
        trace.WithBatcher(exporter),
        trace.WithResource(resource.NewWithAttributes(
            semconv.SchemaURL,
            semconv.ServiceNameKey.String(serviceName),
        )),
    )
    otel.SetTracerProvider(tp)
    return tp, nil
}

// Add to Gin
r.Use(otelgin.Middleware("myapp"))
```

---

## Health Checks

Health check pattern: see `cross-cutting-health-checks.md`.
