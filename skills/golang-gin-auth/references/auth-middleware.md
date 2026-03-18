# Auth — JWT Gin Middleware

See also: `auth-implementation-core.md`, `auth-implementation-handlers.md`

## JWT Middleware

Extracts the `Authorization: Bearer <token>` header, parses and validates the access token, then stores claims in the Gin context for downstream handlers.

```go
// pkg/middleware/auth.go
package middleware

import (
    "log/slog"
    "net/http"
    "strings"

    "github.com/gin-gonic/gin"
    "myapp/internal/auth"
)

const (
    ClaimsKey = "claims"
    UserIDKey  = "user_id"
)

func Auth(cfg auth.TokenConfig, logger *slog.Logger) gin.HandlerFunc {
    return func(c *gin.Context) {
        header := c.GetHeader("Authorization")
        if header == "" {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "authorization header required"})
            return
        }

        parts := strings.SplitN(header, " ", 2)
        if len(parts) != 2 || !strings.EqualFold(parts[0], "Bearer") {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "invalid authorization header format"})
            return
        }

        claims, err := auth.ParseAccessToken(cfg, parts[1])
        if err != nil {
            logger.Warn("jwt validation failed", "error", err)
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "invalid or expired token"})
            return
        }

        c.Set(ClaimsKey, claims)
        c.Set(UserIDKey, claims.UserID)
        c.Next()
    }
}
```

## Getting Current User in Handlers

```go
// Option 1: string shortcut (safe — returns "" if missing)
userID := c.GetString(middleware.UserIDKey)

// Option 2: full claims with safe type assertion
val, exists := c.Get(middleware.ClaimsKey)
if !exists {
    c.JSON(http.StatusUnauthorized, gin.H{"error": "unauthorized"})
    return
}
claims, ok := val.(*auth.Claims)
if !ok {
    c.JSON(http.StatusInternalServerError, gin.H{"error": "invalid claims"})
    return
}
```

> **WARNING:** Never do raw type assertions on `c.Get()` values without checking `ok`. If the auth middleware is misconfigured or skipped, `val.(*auth.Claims)` will panic. Always use the two-value assertion form. See `golang-gin-api` skill → `references/safe-context-extraction.md` for reusable helper functions.
