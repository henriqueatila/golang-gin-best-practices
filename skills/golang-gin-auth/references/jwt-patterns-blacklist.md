# JWT Patterns — Token Blacklisting (Redis)

See also: `jwt-patterns-tokens.md`, `jwt-patterns-storage-and-csrf.md`, `jwt-patterns-rs256.md`

## Token Blacklisting (Redis)

Tokens are stateless — you can't "delete" one. Blacklisting stores revoked token IDs in Redis with TTL matching the token's remaining lifetime. On every request, middleware checks Redis before accepting the token.

```go
// internal/auth/blacklist.go
package auth

import (
    "context"
    "fmt"
    "time"

    "github.com/redis/go-redis/v9"
)

type Blacklist struct {
    rdb    *redis.Client
    prefix string
}

func NewBlacklist(rdb *redis.Client) *Blacklist {
    return &Blacklist{rdb: rdb, prefix: "jwt:revoked:"}
}

// Revoke adds a token ID to the blacklist until its expiry.
func (b *Blacklist) Revoke(ctx context.Context, tokenID string, expiry time.Time) error {
    ttl := time.Until(expiry)
    if ttl <= 0 {
        return nil // already expired — no need to blacklist
    }
    key := b.prefix + tokenID
    if err := b.rdb.Set(ctx, key, "1", ttl).Err(); err != nil {
        return fmt.Errorf("blacklist.Revoke: %w", err)
    }
    return nil
}

// IsRevoked returns true if the token ID has been revoked.
func (b *Blacklist) IsRevoked(ctx context.Context, tokenID string) (bool, error) {
    key := b.prefix + tokenID
    val, err := b.rdb.Exists(ctx, key).Result()
    if err != nil {
        return false, fmt.Errorf("blacklist.IsRevoked: %w", err)
    }
    return val > 0, nil
}
```

**Middleware integration:** After `ParseAccessToken`, check `blacklist.IsRevoked(ctx, claims.ID)` before calling `c.Next()`. The `claims.ID` maps to the `jti` field in `RegisteredClaims`.

**Logout endpoint:**

```go
func (h *AuthHandler) Logout(c *gin.Context) {
    val, exists := c.Get(middleware.ClaimsKey)
    if !exists {
        c.JSON(http.StatusUnauthorized, gin.H{"error": "unauthorized"})
        return
    }
    claims, ok := val.(*auth.Claims)
    if !ok {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
        return
    }

    expiry := claims.ExpiresAt.Time
    if err := h.blacklist.Revoke(c.Request.Context(), claims.ID, expiry); err != nil {
        h.logger.Error("failed to revoke token", "error", err, "jti", claims.ID)
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
        return
    }

    c.JSON(http.StatusOK, gin.H{"message": "logged out"})
}
```

## Complete Auth Flow Wiring

```go
// cmd/api/main.go — protected routes with blacklist check
protected := api.Group("")
protected.Use(middleware.Auth(tokenCfg, logger))
{
    protected.POST("/auth/logout", authHandler.Logout)
    protected.GET("/users/me", userHandler.GetMe)
    protected.GET("/users/:id", userHandler.GetByID)

    admin := protected.Group("/admin")
    admin.Use(middleware.RequireRole("admin"))
    {
        admin.GET("/users", userHandler.List)
        admin.DELETE("/users/:id", userHandler.Delete)
    }
}
```

**Sequence for a protected request:**

1. Client sends `Authorization: Bearer <access_token>`
2. `Auth` middleware extracts and validates the token
3. Middleware checks `blacklist.IsRevoked(ctx, claims.ID)` → 401 if revoked
4. Claims injected via `c.Set(ClaimsKey, claims)` and `c.Set(UserIDKey, claims.UserID)`
5. Handler calls `c.GetString(UserIDKey)` or `c.Get(ClaimsKey)` to read identity
6. Handler passes `c.Request.Context()` to all downstream calls
