# JWT Patterns — Storage Recommendations & CSRF Protection

See also: `jwt-patterns-tokens.md`, `jwt-patterns-blacklist.md`, `jwt-patterns-rs256.md`

## Storage Recommendations

Where the client stores tokens affects the attack surface:

| Storage         | XSS Risk | CSRF Risk | Notes                                          |
|-----------------|----------|-----------|------------------------------------------------|
| httpOnly cookie | Low      | Medium    | Use `SameSite=Strict` + CSRF token to mitigate |
| localStorage    | High     | Low       | JavaScript-accessible — XSS can steal tokens  |
| sessionStorage  | High     | Low       | Same as localStorage, cleared on tab close     |
| Memory (JS var) | Low      | Low       | Lost on refresh — needs silent refresh flow    |

**Recommended:** httpOnly, Secure cookie for refresh token; short-lived access token in memory or Authorization header.

Setting a secure cookie in Gin — use `http.SetCookie` directly to set `SameSite`, which `c.SetCookie` does not expose:

```go
http.SetCookie(c.Writer, &http.Cookie{
    Name:     "refresh_token",
    Value:    refreshToken,
    Path:     "/auth",                  // restrict to /auth/* endpoints
    HttpOnly: true,                     // not accessible by JavaScript
    Secure:   true,                     // HTTPS only
    SameSite: http.SameSiteStrictMode,  // CSRF mitigation
    MaxAge:   7 * 24 * 3600,           // 7 days in seconds
})
```

Reading the cookie in the refresh endpoint:

```go
refreshToken, err := c.Cookie("refresh_token")
if err != nil {
    c.JSON(http.StatusUnauthorized, gin.H{"error": "refresh token not found"})
    return
}
```

---

## CSRF Protection

When using httpOnly cookies for the refresh token, protect state-mutating endpoints from cross-site request forgery. The double-submit cookie pattern: set a non-httpOnly CSRF token cookie on login, require it in a custom header on every mutating request.

```go
// pkg/middleware/csrf.go
package middleware

import (
    "crypto/hmac"
    "net/http"

    "github.com/gin-gonic/gin"
)

// CSRFProtection implements the double-submit cookie pattern.
func CSRFProtection() gin.HandlerFunc {
    return func(c *gin.Context) {
        method := c.Request.Method
        if method == http.MethodGet || method == http.MethodHead || method == http.MethodOptions {
            c.Next()
            return
        }

        csrfHeader := c.GetHeader("X-CSRF-Token")
        cookieToken, err := c.Cookie("csrf_token")
        if err != nil || csrfHeader == "" || !hmac.Equal([]byte(csrfHeader), []byte(cookieToken)) {
            c.AbortWithStatusJSON(http.StatusForbidden, gin.H{"error": "invalid CSRF token"})
            return
        }
        c.Next()
    }
}
```

**Set the CSRF cookie on login** (non-httpOnly so JavaScript can read it):

```go
func generateCSRFToken() (string, error) {
    b := make([]byte, 32)
    if _, err := rand.Read(b); err != nil {
        return "", err
    }
    return hex.EncodeToString(b), nil
}

// Inside Login, after issuing tokens:
csrfToken, err := generateCSRFToken()
if err != nil {
    c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
    return
}
http.SetCookie(c.Writer, &http.Cookie{
    Name:     "csrf_token",
    Value:    csrfToken,
    Path:     "/",
    Secure:   true,
    HttpOnly: false, // must be readable by JS to send as header
    SameSite: http.SameSiteStrictMode,
    MaxAge:   7 * 24 * 3600,
})
```

Apply the middleware on all mutating routes:

```go
api := r.Group("/api/v1")
api.Use(middleware.CSRFProtection())
```

> **Note:** CSRF protection is only necessary when using cookies for auth. If you use `Authorization: Bearer` headers exclusively, CSRF is not a risk — browsers do not send custom headers cross-origin without explicit CORS preflight.
