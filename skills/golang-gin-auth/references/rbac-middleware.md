# RBAC — Role-Based Middleware & Permission Checks

See also: `rbac-hierarchy-and-tenant.md`, `rbac-resource-and-impersonation.md`

## Role-Based Middleware

`RequireRole` and `RequireAnyRole` sit after `Auth` middleware. `Auth` validates the JWT and injects claims; RBAC middleware reads the role from those claims.

```go
// pkg/middleware/rbac.go
package middleware

import (
    "net/http"

    "github.com/gin-gonic/gin"
    "myapp/internal/auth"
)

func RequireRole(role string) gin.HandlerFunc {
    return func(c *gin.Context) {
        claims := claimsFromContext(c)
        if claims == nil {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "unauthorized"})
            return
        }
        if claims.Role != role {
            c.AbortWithStatusJSON(http.StatusForbidden, gin.H{"error": "insufficient role"})
            return
        }
        c.Next()
    }
}

func RequireAnyRole(roles ...string) gin.HandlerFunc {
    allowed := make(map[string]struct{}, len(roles))
    for _, r := range roles {
        allowed[r] = struct{}{}
    }
    return func(c *gin.Context) {
        claims := claimsFromContext(c)
        if claims == nil {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "unauthorized"})
            return
        }
        if _, ok := allowed[claims.Role]; !ok {
            c.AbortWithStatusJSON(http.StatusForbidden, gin.H{"error": "insufficient role"})
            return
        }
        c.Next()
    }
}

func claimsFromContext(c *gin.Context) *auth.Claims {
    val, exists := c.Get(ClaimsKey)
    if !exists {
        return nil
    }
    claims, _ := val.(*auth.Claims)
    return claims
}
```

**Usage:**

```go
protected := api.Group("")
protected.Use(middleware.Auth(tokenCfg, logger))
{
    protected.GET("/users/me", userHandler.GetMe)

    modGroup := protected.Group("")
    modGroup.Use(middleware.RequireAnyRole("admin", "moderator"))
    { modGroup.PUT("/posts/:id/hide", postHandler.Hide) }

    adminGroup := protected.Group("/admin")
    adminGroup.Use(middleware.RequireRole("admin"))
    {
        adminGroup.GET("/users", userHandler.List)
        adminGroup.DELETE("/users/:id", userHandler.Delete)
    }
}
```

---

## Permission-Based Middleware

**Option A — permissions in JWT claims** (fast, no DB lookup per request):

```go
type Claims struct {
    jwt.RegisteredClaims
    UserID      string   `json:"uid"`
    Email       string   `json:"email"`
    Role        string   `json:"role"`
    Permissions []string `json:"perms"` // e.g. ["posts:write", "users:read"]
}

func RequirePermission(permission string) gin.HandlerFunc {
    return func(c *gin.Context) {
        claims := claimsFromContext(c)
        if claims == nil {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "unauthorized"})
            return
        }
        for _, p := range claims.Permissions {
            if p == permission {
                c.Next()
                return
            }
        }
        c.AbortWithStatusJSON(http.StatusForbidden, gin.H{"error": "permission denied"})
    }
}
```

**Option B — DB lookup per request** (always up-to-date, higher latency):

```go
type PermissionService interface {
    HasPermission(ctx context.Context, userID, permission string) (bool, error)
}

func RequirePermissionDB(permSvc PermissionService, permission string) gin.HandlerFunc {
    return func(c *gin.Context) {
        claims := claimsFromContext(c)
        if claims == nil {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "unauthorized"})
            return
        }
        ok, err := permSvc.HasPermission(c.Request.Context(), claims.UserID, permission)
        if err != nil {
            c.AbortWithStatusJSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
            return
        }
        if !ok {
            c.AbortWithStatusJSON(http.StatusForbidden, gin.H{"error": "permission denied"})
            return
        }
        c.Next()
    }
}
```

**Trade-off:** JWT permissions are fast but stale (user retains permissions until token expires). DB lookup is always current but adds latency. Cache the DB result with a short TTL (e.g. 30s in Redis) as a middle ground.
