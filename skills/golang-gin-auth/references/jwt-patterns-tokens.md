# JWT Patterns — Token Architecture, Claims & Refresh Endpoint

See also: `jwt-patterns-blacklist.md`, `jwt-patterns-storage-and-csrf.md`, `jwt-patterns-rs256.md`

## Access + Refresh Token Architecture

Two-token strategy: short-lived access tokens reduce attack surface; long-lived refresh tokens enable session continuity without re-login.

```
┌──────────┐   POST /auth/login    ┌───────────┐
│  Client  │──────────────────────▶│  Gin API  │
│          │◀──────────────────────│           │
│          │  {access, refresh}    └───────────┘
│          │
│          │   GET /api/v1/users   ┌───────────┐
│          │   Authorization: Bearer <access>
│          │──────────────────────▶│  Gin API  │
│          │◀──────────────────────│           │
│          │   200 OK              └───────────┘
│          │
│          │   POST /auth/refresh  ┌───────────┐
│          │   {refresh_token}     │           │
│          │──────────────────────▶│  Gin API  │
│          │◀──────────────────────│           │
│          │   {new_access}        └───────────┘
└──────────┘
```

| Token         | TTL        | Payload       | Stored where    |
|---------------|------------|---------------|-----------------|
| Access token  | 15 minutes | UserID, Role  | Memory / header |
| Refresh token | 7–30 days  | UserID only   | httpOnly cookie |

---

## Custom Claims

Embed `jwt.RegisteredClaims` for standard fields. Add only what handlers need.

```go
// internal/auth/claims.go
package auth

import "github.com/golang-jwt/jwt/v5"

type Claims struct {
    jwt.RegisteredClaims            // sub, iat, exp, iss, aud
    UserID   string   `json:"uid"`
    Email    string   `json:"email"`
    Role     string   `json:"role"`
}

// RefreshClaims is the minimal payload for refresh tokens.
// TokenID (jti) enables per-token revocation without invalidating all sessions.
type RefreshClaims struct {
    jwt.RegisteredClaims
    TokenID string `json:"jti"`
}
```

---

## Token Refresh Endpoint

Two rotation strategies — pick one:

- **Option A: Always rotate** — new refresh token on every call. More secure but breaks parallel requests using the same refresh token.
- **Option B: Rotate past half-lifetime (recommended)** — issue a new refresh token only when the current one is more than halfway through its lifetime.

```go
// internal/handler/auth_handler.go (Refresh method)
func (h *AuthHandler) Refresh(c *gin.Context) {
    var req struct {
        RefreshToken string `json:"refresh_token" binding:"required"`
    }
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "invalid request body"})
        return
    }

    token, err := jwt.ParseWithClaims(req.RefreshToken, &auth.RefreshClaims{}, func(t *jwt.Token) (any, error) {
        if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
            return nil, fmt.Errorf("unexpected signing method: %v", t.Header["alg"])
        }
        return h.tokenCfg.RefreshSecret, nil
    })
    if err != nil {
        c.JSON(http.StatusUnauthorized, gin.H{"error": "invalid or expired refresh token"})
        return
    }

    rc, ok := token.Claims.(*auth.RefreshClaims)
    if !ok || !token.Valid {
        c.JSON(http.StatusUnauthorized, gin.H{"error": "invalid refresh token claims"})
        return
    }

    user, err := h.userRepo.GetByID(c.Request.Context(), rc.Subject)
    if err != nil {
        c.JSON(http.StatusUnauthorized, gin.H{"error": "invalid or expired refresh token"})
        return
    }

    accessToken, err := auth.GenerateAccessToken(h.tokenCfg, user.ID, user.Email, user.Role)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
        return
    }

    resp := gin.H{
        "access_token": accessToken,
        "expires_in":   int(h.tokenCfg.AccessTTL / time.Second),
    }
    // Option B: rotate only past half-lifetime
    halfLife := rc.IssuedAt.Time.Add(h.tokenCfg.RefreshTTL / 2)
    if time.Now().After(halfLife) {
        newRefresh, err := auth.GenerateRefreshToken(h.tokenCfg, user.ID)
        if err != nil {
            c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
            return
        }
        resp["refresh_token"] = newRefresh
    }

    c.JSON(http.StatusOK, resp)
}
```
