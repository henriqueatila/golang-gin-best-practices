# Cross-Cutting Concerns — Security, Config, and Feature Flags

Security architecture, configuration management, feature flags, and graceful degradation for Go Gin APIs.

Companion to `cross-cutting-observability.md` (logging, metrics, tracing, health checks).
Caching decision tree: see `cross-cutting-observability.md` or `redis-caching-patterns.md`.

---

## Security Architecture

### Defense in Depth

```
Internet → Load Balancer (TLS termination, DDoS protection)
         → Gin Middleware (rate limiting, CORS, auth)
           → Handler (input validation, sanitization)
             → Service (business rule enforcement, authorization)
               → Repository (parameterized queries, RLS)
                 → PostgreSQL (encryption at rest, network isolation)
```

### Secret Management

```go
type Config struct {
    DatabaseURL string `env:"DATABASE_URL,required"`
    JWTSecret   string `env:"JWT_SECRET,required"`
    RedisURL    string `env:"REDIS_URL"`
    Port        string `env:"PORT" envDefault:"8080"`
}

func LoadConfig() (*Config, error) {
    cfg := &Config{}
    if err := env.Parse(cfg); err != nil {
        return nil, fmt.Errorf("parse config: %w", err)
    }
    return cfg, nil
}
```

**Rules:** Never hardcode secrets. Never commit `.env` files. Use `.env.example` with placeholders. In production: Docker secrets, Kubernetes secrets, or a vault (HashiCorp, AWS SSM).

### Request Signing (Service-to-Service)

```go
func SignRequest(body []byte, secret string) string {
    mac := hmac.New(sha256.New, []byte(secret))
    mac.Write(body)
    return hex.EncodeToString(mac.Sum(nil))
}

func VerifySignature(secret string) gin.HandlerFunc {
    return func(c *gin.Context) {
        signature := c.GetHeader("X-Signature")
        if signature == "" {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "missing signature"})
            return
        }
        body, err := io.ReadAll(c.Request.Body)
        if err != nil {
            c.AbortWithStatusJSON(http.StatusBadRequest, gin.H{"error": "cannot read body"})
            return
        }
        c.Request.Body = io.NopCloser(bytes.NewBuffer(body))
        expected := SignRequest(body, secret)
        if !hmac.Equal([]byte(signature), []byte(expected)) {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "invalid signature"})
            return
        }
        c.Next()
    }
}
```

---

## Configuration Management

Single `Config` struct loaded once in `main.go`, passed via dependency injection.

```go
type Config struct {
    Port         string        `env:"PORT"          envDefault:"8080"`
    Environment  string        `env:"ENVIRONMENT"   envDefault:"development"`
    DatabaseURL  string        `env:"DATABASE_URL,required"`
    MaxOpenConns int           `env:"DB_MAX_OPEN"   envDefault:"25"`
    JWTSecret    string        `env:"JWT_SECRET,required"`
    TokenTTL     time.Duration `env:"TOKEN_TTL"     envDefault:"15m"`
}
```

**Rules:** Never read `os.Getenv` in service/repository code. Use `envDefault` for development defaults. Required fields fail fast at startup (not at first request).

---

## Feature Flags

**Start here — env vars cover 90% of use cases:**

```go
type FeatureFlags struct {
    EnableNewCheckout bool `env:"FF_NEW_CHECKOUT" envDefault:"false"`
    EnableBetaSearch  bool `env:"FF_BETA_SEARCH"  envDefault:"false"`
}
```

**User-targeted flags when you outgrow env vars:**

```sql
CREATE TABLE feature_flags (
    name       TEXT PRIMARY KEY,
    enabled    BOOLEAN NOT NULL DEFAULT false,
    rules      JSONB,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Graceful Degradation

Classify each dependency as **critical** (propagate error) or **non-critical** (degrade + log + return partial result).

```go
func (s *ProductService) GetWithRecommendations(ctx context.Context, id string) (*ProductPage, error) {
    product, err := s.repo.GetByID(ctx, id)
    if err != nil {
        return nil, fmt.Errorf("get product: %w", err) // critical — propagate
    }

    recommendations, err := s.recommender.Get(ctx, id)
    if err != nil {
        s.logger.WarnContext(ctx, "recommendations unavailable, returning empty",
            "product_id", id, "error", err)
        recommendations = []Product{} // non-critical — degrade gracefully
    }

    return &ProductPage{Product: product, Recommendations: recommendations}, nil
}
```
