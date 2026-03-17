# Observability — OTel SDK Setup, Gin Middleware, and Manual Spans

> OTel Go SDK is stable at v1.x. Pin `go.opentelemetry.io/otel` and `go.opentelemetry.io/contrib` versions explicitly — contrib packages release independently.

## OTel SDK Setup

Provider → exporter → processor pipeline. Initialize `TracerProvider` and `MeterProvider` once at startup, register as globals. OTLP gRPC is preferred — vendor-neutral, works with Jaeger, Grafana Tempo, Honeycomb, Datadog.

```go
// internal/telemetry/telemetry.go
type Providers struct {
    TracerProvider *sdktrace.TracerProvider
    MeterProvider  *sdkmetric.MeterProvider
}

func Init(ctx context.Context, serviceName, serviceVersion, otlpEndpoint string) (*Providers, error) {
    res, _ := resource.New(ctx,
        resource.WithAttributes(semconv.ServiceName(serviceName), semconv.ServiceVersion(serviceVersion)),
        resource.WithFromEnv(), resource.WithProcess(), resource.WithOS(),
    )
    conn, _ := grpc.NewClient(otlpEndpoint, grpc.WithTransportCredentials(insecure.NewCredentials()))

    traceExporter, _ := otlptracegrpc.New(ctx, otlptracegrpc.WithGRPCConn(conn))
    tp := sdktrace.NewTracerProvider(
        sdktrace.WithBatcher(traceExporter), sdktrace.WithResource(res),
        sdktrace.WithSampler(sdktrace.ParentBased(sdktrace.TraceIDRatioBased(0.1))),
    )
    otel.SetTracerProvider(tp)
    otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
        propagation.TraceContext{}, propagation.Baggage{},
    ))

    metricExporter, _ := otlpmetricgrpc.New(ctx, otlpmetricgrpc.WithGRPCConn(conn))
    mp := sdkmetric.NewMeterProvider(
        sdkmetric.WithReader(sdkmetric.NewPeriodicReader(metricExporter, sdkmetric.WithInterval(15*time.Second))),
        sdkmetric.WithResource(res),
    )
    otel.SetMeterProvider(mp)
    return &Providers{TracerProvider: tp, MeterProvider: mp}, nil
}
```

**go.mod:** `go.opentelemetry.io/otel v1.33.0`, `otel/sdk v1.33.0`, `otlptracegrpc v1.33.0`, `otlpmetricgrpc v1.33.0`, `contrib/.../otelgin v0.58.0`, `google.golang.org/grpc v1.69.0`.

---

## Gin Middleware

`otelgin.Middleware` creates a span per HTTP request, sets `http.method/route/status_code`, propagates W3C `traceparent`. Register first so auth/logging middleware runs inside the span.

```go
func setupRouter(serviceName string, logger *slog.Logger) *gin.Engine {
    r := gin.New()
    r.Use(otelgin.Middleware(serviceName)) // must be first
    r.Use(middleware.Recovery(logger))
    r.Use(func(c *gin.Context) {
        c.Next()
        logger.InfoContext(c.Request.Context(), "request",
            "method", c.Request.Method, "path", c.Request.URL.Path, "status", c.Writer.Status())
    })
    return r
}
```

`otelgin` uses the route pattern (`/users/:id`) as span name — preventing high-cardinality spans.

---

## Manual Spans in Service Layer

Automatic middleware gives one span per HTTP request. Add child spans for DB queries and external calls to identify slow operations.

```go
var tracer = otel.Tracer("myapp/service/order")

func (s *OrderService) CreateOrder(ctx context.Context, userID string, items []Item) (*Order, error) {
    ctx, span := tracer.Start(ctx, "OrderService.CreateOrder",
        trace.WithAttributes(
            attribute.String("order.user_id", userID),
            attribute.Int("order.item_count", len(items)),
        ),
    )
    defer span.End()

    order, err := s.repo.Insert(ctx, userID, items) // ctx carries the span
    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, err.Error())
        return nil, fmt.Errorf("create order: %w", err)
    }
    span.SetAttributes(attribute.String("order.id", order.ID))
    span.SetStatus(codes.Ok, "")
    return order, nil
}
```

Rules: always `defer span.End()` immediately after `Start`; pass `ctx` down the call chain; name spans `"Package.Method"`; call both `RecordError` and `SetStatus(codes.Error)` — `RecordError` alone does not mark the span as failed in the UI.

> For metrics, log-trace correlation, sampling: see [observability-metrics-logs-sampling.md](observability-metrics-logs-sampling.md).
> For Docker Compose observability stack and graceful shutdown: see [observability-docker-stack-and-shutdown.md](observability-docker-stack-and-shutdown.md).
