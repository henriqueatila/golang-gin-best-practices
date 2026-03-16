# OAuth2 / Social Login

OAuth2 social login integration for Go Gin APIs. Covers GitHub and Google flows with CSRF state management, token exchange, and JWT generation.

## Dependencies

```bash
go get golang.org/x/oauth2
go get golang.org/x/oauth2/github
go get golang.org/x/oauth2/google
```

## OAuth2 Config

Load all secrets from environment — never hardcode.

```go
// internal/auth/oauth2.go
package auth

import (
    "os"

    "golang.org/x/oauth2"
    "golang.org/x/oauth2/github"
    "golang.org/x/oauth2/google"
)

func NewGitHubConfig() *oauth2.Config {
    return &oauth2.Config{
        ClientID:     os.Getenv("GITHUB_CLIENT_ID"),
        ClientSecret: os.Getenv("GITHUB_CLIENT_SECRET"),
        RedirectURL:  os.Getenv("GITHUB_REDIRECT_URL"), // e.g. https://example.com/auth/github/callback
        Endpoint:     github.Endpoint,
        Scopes:       []string{"user:email", "read:user"},
    }
}

func NewGoogleConfig() *oauth2.Config {
    return &oauth2.Config{
        ClientID:     os.Getenv("GOOGLE_CLIENT_ID"),
        ClientSecret: os.Getenv("GOOGLE_CLIENT_SECRET"),
        RedirectURL:  os.Getenv("GOOGLE_REDIRECT_URL"),
        Endpoint:     google.Endpoint,
        Scopes:       []string{"openid", "email", "profile"},
    }
}
```

## CSRF State Management

Generate a random state token, store it in Redis (or in-memory cache) with a short TTL, and validate it on callback to prevent CSRF attacks.

```go
// internal/auth/state.go
package auth

import (
    "context"
    "crypto/rand"
    "encoding/hex"
    "fmt"
    "time"

    "github.com/redis/go-redis/v9"
)

const stateTTL = 10 * time.Minute

func GenerateState() (string, error) {
    b := make([]byte, 16)
    if _, err := rand.Read(b); err != nil {
        return "", err
    }
    return hex.EncodeToString(b), nil
}

func StoreState(ctx context.Context, rdb *redis.Client, state string) error {
    key := fmt.Sprintf("oauth2:state:%s", state)
    return rdb.Set(ctx, key, "1", stateTTL).Err()
}

func ValidateAndDeleteState(ctx context.Context, rdb *redis.Client, state string) (bool, error) {
    key := fmt.Sprintf("oauth2:state:%s", state)
    n, err := rdb.Del(ctx, key).Result()
    if err != nil {
        return false, err
    }
    return n == 1, nil
}
```

## GitHub OAuth2 Flow

### Initiate Handler — `/auth/github`

```go
// internal/handler/oauth2_handler.go
package handler

import (
    "log/slog"
    "net/http"

    "github.com/gin-gonic/gin"
    "yourmodule/internal/auth"
)

type OAuth2Handler struct {
    githubCfg *oauth2.Config
    stateStore auth.StateStore // wraps Redis calls
    userService UserService
    tokenSvc   auth.TokenService
    logger     *slog.Logger
}

func (h *OAuth2Handler) GitHubLogin(c *gin.Context) {
    state, err := auth.GenerateState()
    if err != nil {
        h.logger.Error("failed to generate oauth2 state", "error", err)
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
        return
    }

    if err := auth.StoreState(c.Request.Context(), h.rdb, state); err != nil {
        h.logger.Error("failed to store oauth2 state", "error", err)
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
        return
    }

    url := h.githubCfg.AuthCodeURL(state)
    c.Redirect(http.StatusTemporaryRedirect, url)
}
```

### Callback Handler — `/auth/github/callback`

```go
func (h *OAuth2Handler) GitHubCallback(c *gin.Context) {
    ctx := c.Request.Context()

    // 1. Validate CSRF state
    state := c.Query("state")
    ok, err := auth.ValidateAndDeleteState(ctx, h.rdb, state)
    if err != nil || !ok {
        c.JSON(http.StatusBadRequest, gin.H{"error": "invalid state"})
        return
    }

    // 2. Exchange code for token
    code := c.Query("code")
    token, err := h.githubCfg.Exchange(ctx, code)
    if err != nil {
        h.logger.Error("oauth2 code exchange failed", "error", err)
        c.JSON(http.StatusUnauthorized, gin.H{"error": "authentication failed"})
        return
    }

    // 3. Fetch GitHub user profile
    profile, err := h.fetchGitHubProfile(ctx, token.AccessToken)
    if err != nil {
        h.logger.Error("failed to fetch github profile", "error", err)
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
        return
    }

    // 4. Find or create user in DB
    user, err := h.userService.FindOrCreateOAuth(ctx, "github", profile.ID, profile.Email, profile.Login)
    if err != nil {
        h.logger.Error("FindOrCreateOAuth failed", "error", err)
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
        return
    }

    // 5. Generate JWT pair
    accessToken, refreshToken, err := h.tokenSvc.GeneratePair(user.ID, user.Email, user.Role)
    if err != nil {
        h.logger.Error("token generation failed", "error", err)
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
        return
    }

    c.JSON(http.StatusOK, gin.H{
        "access_token":  accessToken,
        "refresh_token": refreshToken,
    })
}
```

### Fetch GitHub Profile Helper

```go
// internal/auth/github_profile.go
package auth

import (
    "context"
    "encoding/json"
    "fmt"
    "net/http"
)

type GitHubProfile struct {
    ID    int64  `json:"id"`
    Login string `json:"login"`
    Email string `json:"email"`
}

func FetchGitHubProfile(ctx context.Context, accessToken string) (*GitHubProfile, error) {
    req, err := http.NewRequestWithContext(ctx, http.MethodGet, "https://api.github.com/user", nil)
    if err != nil {
        return nil, err
    }
    req.Header.Set("Authorization", "Bearer "+accessToken)
    req.Header.Set("Accept", "application/vnd.github+json")

    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusOK {
        return nil, fmt.Errorf("github api returned %d", resp.StatusCode)
    }

    var profile GitHubProfile
    if err := json.NewDecoder(resp.Body).Decode(&profile); err != nil {
        return nil, err
    }
    return &profile, nil
}
```

## Google OAuth2 Flow

Config differences only — callback pattern is identical to GitHub:

```go
// internal/auth/google_userinfo.go
// Google userinfo endpoint (use instead of github API call):
// GET https://www.googleapis.com/oauth2/v3/userinfo
// Authorization: Bearer <access_token>
// Response fields: sub (ID), email, name, picture
```

## Route Registration

Wire into the Gin router alongside existing auth routes:

```go
// internal/router/router.go
authGroup := r.Group("/auth")
authGroup.Use(middleware.IPRateLimiter(5, time.Minute))
{
    authGroup.POST("/login", authHandler.Login)
    authGroup.POST("/register", authHandler.Register)
    authGroup.POST("/refresh", authHandler.Refresh)

    // OAuth2 routes — no rate limiter needed (redirects, not form submissions)
    authGroup.GET("/github", oauth2Handler.GitHubLogin)
    authGroup.GET("/github/callback", oauth2Handler.GitHubCallback)
    authGroup.GET("/google", oauth2Handler.GoogleLogin)
    authGroup.GET("/google/callback", oauth2Handler.GoogleCallback)
}
```

## Security Checklist

- **State validation**: Always verify and delete state in a single atomic operation (use Redis DEL return value)
- **HTTPS in production**: Set `RedirectURL` to `https://` — never `http://` in prod; validate `REDIRECT_URL` env var at startup
- **Token storage**: Store OAuth2 access tokens server-side only; never send to client
- **Redirect URL validation**: Register exact redirect URLs in the OAuth2 provider dashboard; reject any deviation
- **Scope minimization**: Request only the scopes you actually need (`user:email` not full `user`)
- **Error messages**: Return generic errors — never expose provider error details to clients
- **State TTL**: Keep state TTL short (10 minutes max) to limit CSRF window
