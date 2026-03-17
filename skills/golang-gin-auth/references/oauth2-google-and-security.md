# OAuth2 / Social Login — Google Flow, Route Registration & Security Checklist

See also: `oauth2-flows.md`

## Google OAuth2 Flow

Config differences only — callback pattern is identical to GitHub. Use the Google userinfo endpoint instead of the GitHub API:

```go
// internal/auth/google_userinfo.go
// GET https://www.googleapis.com/oauth2/v3/userinfo
// Authorization: Bearer <access_token>
// Response fields: sub (ID), email, name, picture
```

Implement `GoogleLogin` and `GoogleCallback` handlers following the same pattern as `GitHubLogin` / `GitHubCallback` in `oauth2-flows.md`, substituting `h.googleCfg` and the Google userinfo endpoint.

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

| Risk | Mitigation |
|------|-----------|
| CSRF on callback | Always verify and delete state in a single atomic Redis DEL operation |
| Insecure redirect URL | Set `RedirectURL` to `https://` only; validate `REDIRECT_URL` env var at startup |
| Token leakage | Store OAuth2 access tokens server-side only; never send to client |
| Open redirect | Register exact redirect URLs in the OAuth2 provider dashboard; reject any deviation |
| Excessive scope | Request only the scopes you actually need (`user:email` not full `user`) |
| Provider error leakage | Return generic errors — never expose provider error details to clients |
| Long CSRF window | Keep state TTL short (10 minutes max) |
