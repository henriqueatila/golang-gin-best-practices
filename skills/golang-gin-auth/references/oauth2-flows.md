# OAuth2 / Social Login — Config, CSRF State & GitHub Flow

See also: `oauth2-google-and-security.md`

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
        RedirectURL:  os.Getenv("GITHUB_REDIRECT_URL"),
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

Generate a random state token, store in Redis with short TTL, validate atomically on callback.

```go
// internal/auth/state.go
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
func (h *OAuth2Handler) GitHubLogin(c *gin.Context) {
    state, err := auth.GenerateState()
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
        return
    }
    if err := auth.StoreState(c.Request.Context(), h.rdb, state); err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
        return
    }
    c.Redirect(http.StatusTemporaryRedirect, h.githubCfg.AuthCodeURL(state))
}
```

### Callback Handler — `/auth/github/callback`

```go
func (h *OAuth2Handler) GitHubCallback(c *gin.Context) {
    ctx := c.Request.Context()

    ok, err := auth.ValidateAndDeleteState(ctx, h.rdb, c.Query("state"))
    if err != nil || !ok {
        c.JSON(http.StatusBadRequest, gin.H{"error": "invalid state"})
        return
    }

    token, err := h.githubCfg.Exchange(ctx, c.Query("code"))
    if err != nil {
        c.JSON(http.StatusUnauthorized, gin.H{"error": "authentication failed"})
        return
    }

    profile, err := auth.FetchGitHubProfile(ctx, token.AccessToken)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
        return
    }

    user, err := h.userService.FindOrCreateOAuth(ctx, "github", profile.ID, profile.Email, profile.Login)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
        return
    }

    accessToken, refreshToken, err := h.tokenSvc.GeneratePair(user.ID, user.Email, user.Role)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
        return
    }

    c.JSON(http.StatusOK, gin.H{"access_token": accessToken, "refresh_token": refreshToken})
}
```

See also: `oauth2-github-profile.md` for the `FetchGitHubProfile` helper.
