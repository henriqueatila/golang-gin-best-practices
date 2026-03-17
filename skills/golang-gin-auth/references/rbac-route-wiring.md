# RBAC — Complete Route Wiring

See also: `rbac-middleware.md`, `rbac-resource-and-impersonation.md`, `rbac-hierarchy-and-tenant.md`

## Route Registration

```go
func registerRoutes(r *gin.Engine, ...) {
    api := r.Group("/api/v1")
    api.POST("/auth/login", authHandler.Login)
    api.POST("/auth/refresh", authHandler.Refresh)
    api.POST("/users", userHandler.Create)

    authed := api.Group("")
    authed.Use(middleware.Auth(tokenCfg, logger))
    {
        authed.POST("/auth/logout", authHandler.Logout)
        authed.GET("/users/me", userHandler.GetMe)
        authed.PUT("/users/:id", userHandler.Update) // resource-level check inside handler

        posts := authed.Group("/posts")
        posts.Use(middleware.RequireMinRole("user"))
        { posts.GET("", postHandler.List); posts.POST("", postHandler.Create) }

        moderation := authed.Group("/moderation")
        moderation.Use(middleware.RequireMinRole("moderator"))
        { moderation.PUT("/posts/:id/hide", postHandler.Hide) }

        admin := authed.Group("/admin")
        admin.Use(middleware.RequireRole("admin"))
        {
            admin.GET("/users", userHandler.List)
            admin.DELETE("/users/:id", userHandler.Delete)
            admin.POST("/impersonate", adminHandler.Impersonate)
        }
    }
}
```

**Key principles:**
- Auth middleware is always first in the protected group
- Role/permission middleware comes immediately after Auth
- Resource-level checks belong in the handler, not middleware
- Use `RequireMinRole` for hierarchy, `RequireAnyRole` for explicit sets, `RequireRole` for exact match
