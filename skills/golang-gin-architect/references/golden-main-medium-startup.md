# Golden main.go — Medium Project: Startup Sequence and Patterns

RegisterRoutes pattern, startup order, and graceful shutdown for medium projects.

Companion to `golden-main-medium-project.md` (complete main.go).

---

## RegisterRoutes Pattern

Each handler package owns its route registration:

```go
// internal/features/user/userhandler/routes.go
func RegisterRoutes(rg *gin.RouterGroup, h *Handler) {
	users := rg.Group("/users")
	users.GET("/:id", h.GetByID)
	users.POST("", h.Create)
	users.PUT("/:id", h.Update)
	users.DELETE("/:id", h.Delete)
}

// internal/features/order/orderhandler/routes.go
func RegisterRoutes(rg *gin.RouterGroup, h *Handler) {
	orders := rg.Group("/orders")
	orders.GET("/:id", h.GetByID)
	orders.POST("", h.Create)
	admin := orders.Group("", middleware.RequireRole("admin"))
	admin.DELETE("/:id", h.Delete)
}
```

---

## Startup Sequence (Strict Order)

```
1. Logger       → needed by every subsequent step
2. Config       → all components read from it; fail fast on missing required vars
3. Database     → repos depend on *sqlx.DB; configure pool immediately after connect
4. Repositories → services depend on repo interfaces
5. Services     → handlers depend on service interfaces
6. Handlers     → routes depend on handler methods
7. Router       → gin.New(); global middleware: Recovery → RequestID → Logger → CORS
8. Routes       → health (no auth), then authenticated groups
9. http.Server  → explicit timeouts; start in goroutine
10. Signal wait → block on SIGINT/SIGTERM; graceful shutdown with context timeout
```

**Why `gin.New()` not `gin.Default()`:** `gin.Default()` adds a non-structured logger conflicting with `slog`, and silently adds `Recovery()`. Use `gin.New()` so middleware stack is explicit and auditable.

---

## Graceful Shutdown

```
Signal received
      │
      ▼
srv.Shutdown(ctx)   ← stops new connections, drains in-flight
      │
      ▼
db.Close()          ← via defer, runs after Shutdown returns
      │
      ▼
logger.Info("stopped")
```

```go
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()
if err := srv.Shutdown(ctx); err != nil {
	logger.Error("shutdown timeout, forcing close", "error", err)
	_ = srv.Close()
}
```

Rules: `ShutdownTimeout` > longest expected request. Never call `srv.Close()` before `srv.Shutdown()`. Handlers must propagate `c.Request.Context()` so in-flight DB queries terminate cleanly.

---

## Cross-Skill References

| Topic | Skill |
|---|---|
| Handler patterns, `ShouldBind*` | `golang-gin-api` |
| Repository patterns, sqlx, migrations | `golang-gin-database` |
| JWT middleware, auth groups | `golang-gin-auth` |
| Docker, env config, health checks | `golang-gin-deploy` |
| Clean architecture folder structure | `clean-architecture-feature-module.md` |
| Small project template | `golden-main-small-project.md` |
