# Golden main.go — Medium Project Template

Multiple domains (users, orders, products). Each feature module owns its handler and route registration.

Companion to `golden-main-medium-startup.md` (startup sequence, graceful shutdown, RegisterRoutes pattern).

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

	"github.com/gin-gonic/gin"
	"github.com/jmoiron/sqlx"
	_ "github.com/lib/pq"

	"myapp/internal/config"
	"myapp/internal/features/order/orderhandler"
	"myapp/internal/features/order/orderpostgres"
	"myapp/internal/features/order/orderusecase"
	"myapp/internal/features/user/userhandler"
	"myapp/internal/features/user/userpostgres"
	"myapp/internal/features/user/userusecase"
	"myapp/pkg/middleware"
)

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
		Level: slog.LevelInfo,
	}))

	cfg, err := config.Load()
	if err != nil {
		logger.Error("failed to load config", "error", err)
		os.Exit(1)
	}

	db, err := sqlx.Connect("postgres", cfg.DatabaseURL)
	if err != nil {
		logger.Error("failed to connect to database", "error", err)
		os.Exit(1)
	}
	db.SetMaxOpenConns(cfg.DBMaxOpenConns)
	db.SetMaxIdleConns(cfg.DBMaxIdleConns)
	db.SetConnMaxLifetime(cfg.DBConnMaxLifetime)
	defer db.Close()

	// Wire each feature module independently — no cross-module repo sharing
	userRepo := userpostgres.NewRepository(db)
	userSvc := userusecase.NewService(userRepo, logger)
	userH := userhandler.NewHandler(userSvc, logger)

	orderRepo := orderpostgres.NewRepository(db)
	orderSvc := orderusecase.NewService(orderRepo, userSvc, logger)
	orderH := orderhandler.NewHandler(orderSvc, logger)

	if cfg.GinMode == "release" {
		gin.SetMode(gin.ReleaseMode)
	}
	r := gin.New()
	r.Use(gin.Recovery())
	r.Use(middleware.RequestID())
	r.Use(middleware.Logger(logger))

	r.GET("/health", func(c *gin.Context) {
		if err := db.PingContext(c.Request.Context()); err != nil {
			c.JSON(http.StatusServiceUnavailable, gin.H{"status": "unhealthy"})
			return
		}
		c.JSON(http.StatusOK, gin.H{"status": "ok"})
	})

	authMiddleware := middleware.JWT(cfg.JWTSecret)
	api := r.Group("/api/v1")
	api.Use(authMiddleware)

	// Each handler package registers its own routes
	userhandler.RegisterRoutes(api, userH)
	orderhandler.RegisterRoutes(api, orderH)

	srv := &http.Server{
		Addr:         ":" + cfg.Port,
		Handler:      r,
		ReadTimeout:  cfg.ReadTimeout,
		WriteTimeout: cfg.WriteTimeout,
		IdleTimeout:  cfg.IdleTimeout,
	}

	go func() {
		logger.Info("server starting", "port", cfg.Port)
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
