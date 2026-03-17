# Golden main.go — Small Project Template

Single-domain app (e.g. a users API). All wiring in one place, no feature module splitting.

---

## Complete main.go

```go
package main

import (
	"context"
	"log/slog"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/gin-gonic/gin"
	"github.com/jmoiron/sqlx"
	_ "github.com/lib/pq"

	"myapp/internal/config"
	"myapp/internal/handler"
	"myapp/internal/repository"
	"myapp/internal/service"
	"myapp/pkg/middleware"
)

func main() {
	// 1. Logger — initialized first so everything else can log
	logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
		Level: slog.LevelInfo,
	}))

	// 2. Config
	cfg, err := config.Load()
	if err != nil {
		logger.Error("failed to load config", "error", err)
		os.Exit(1)
	}

	// 3. Database — repositories depend on it
	db, err := sqlx.Connect("postgres", cfg.DatabaseURL)
	if err != nil {
		logger.Error("failed to connect to database", "error", err)
		os.Exit(1)
	}
	db.SetMaxOpenConns(cfg.DBMaxOpenConns)
	db.SetMaxIdleConns(cfg.DBMaxIdleConns)
	db.SetConnMaxLifetime(cfg.DBConnMaxLifetime)
	defer db.Close()

	// 4. Wire dependencies — manual DI: repo → service → handler
	userRepo := repository.NewUserRepository(db)
	userSvc := service.NewUserService(userRepo, logger)
	userHandler := handler.NewUserHandler(userSvc, logger)
	healthHandler := handler.NewHealthHandler(db, logger)

	// 5. Router — gin.New() never gin.Default()
	if cfg.GinMode == "release" {
		gin.SetMode(gin.ReleaseMode)
	}
	r := gin.New()
	r.Use(gin.Recovery())
	r.Use(middleware.RequestID())
	r.Use(middleware.Logger(logger))
	r.Use(middleware.CORS(cfg.AllowedOrigins))

	// 6. Routes
	r.GET("/health", healthHandler.Check)
	api := r.Group("/api/v1")
	api.GET("/users/:id", userHandler.GetByID)
	api.POST("/users", userHandler.Create)
	api.PUT("/users/:id", userHandler.Update)
	api.DELETE("/users/:id", userHandler.Delete)

	// 7. HTTP server with explicit timeouts
	srv := &http.Server{
		Addr:         ":" + cfg.Port,
		Handler:      r,
		ReadTimeout:  cfg.ReadTimeout,
		WriteTimeout: cfg.WriteTimeout,
		IdleTimeout:  cfg.IdleTimeout,
	}

	go func() {
		logger.Info("server starting", "port", cfg.Port, "mode", cfg.GinMode)
		if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			logger.Error("server error", "error", err)
			os.Exit(1)
		}
	}()

	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	<-quit
	logger.Info("shutdown signal received")

	ctx, cancel := context.WithTimeout(context.Background(), cfg.ShutdownTimeout)
	defer cancel()
	if err := srv.Shutdown(ctx); err != nil {
		logger.Error("server forced shutdown", "error", err)
		os.Exit(1)
	}
	logger.Info("server stopped")
}
```

---

## Config Struct

`internal/config/config.go`:

```go
type Config struct {
	Port              string
	GinMode           string
	DatabaseURL       string
	DBMaxOpenConns    int
	DBMaxIdleConns    int
	DBConnMaxLifetime time.Duration
	ReadTimeout       time.Duration
	WriteTimeout      time.Duration
	IdleTimeout       time.Duration
	ShutdownTimeout   time.Duration
	AllowedOrigins    []string
}

func Load() (*Config, error) {
	return &Config{
		Port:              getEnv("PORT", "8080"),
		GinMode:           getEnv("GIN_MODE", "debug"),
		DatabaseURL:       mustEnv("DATABASE_URL"),
		DBMaxOpenConns:    getEnvInt("DB_MAX_OPEN_CONNS", 25),
		DBMaxIdleConns:    getEnvInt("DB_MAX_IDLE_CONNS", 5),
		DBConnMaxLifetime: getEnvDuration("DB_CONN_MAX_LIFETIME", 5*time.Minute),
		ReadTimeout:       getEnvDuration("READ_TIMEOUT", 10*time.Second),
		WriteTimeout:      getEnvDuration("WRITE_TIMEOUT", 10*time.Second),
		IdleTimeout:       getEnvDuration("IDLE_TIMEOUT", 60*time.Second),
		ShutdownTimeout:   getEnvDuration("SHUTDOWN_TIMEOUT", 30*time.Second),
	}, nil
}
```

**Conventions enforced:** `gin.New()` (not `gin.Default()`), `log/slog` JSON handler, `ShouldBind*` in handlers, `c.Request.Context()` propagated, explicit `http.Server` timeouts, pool configured before first query. See also: `golden-main-medium-project.md`.
