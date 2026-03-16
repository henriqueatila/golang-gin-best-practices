# CAPTCHA Middleware

Server-side CAPTCHA verification middleware for Go Gin APIs. Covers reCAPTCHA v2/v3 and hCaptcha via HTTP verification — no external Go library required.

## Dependencies

No additional Go packages needed. Uses only the standard library `net/http` and `encoding/json`.

## reCAPTCHA v2/v3 Verification

Google's `siteverify` endpoint accepts a POST with the secret key and the token submitted by the client.

```go
// internal/middleware/captcha.go
package middleware

import (
    "context"
    "encoding/json"
    "log/slog"
    "net/http"
    "net/url"
    "os"
    "strings"

    "github.com/gin-gonic/gin"
)

const (
    recaptchaVerifyURL = "https://www.google.com/recaptcha/api/siteverify"
    hcaptchaVerifyURL  = "https://hcaptcha.com/siteverify"
)

type captchaResponse struct {
    Success bool    `json:"success"`
    Score   float64 `json:"score"`   // reCAPTCHA v3 only
    Action  string  `json:"action"`  // reCAPTCHA v3 only
}

func verifyCaptcha(ctx context.Context, verifyURL, secretKey, token string) (bool, error) {
    resp, err := http.PostForm(verifyURL, url.Values{
        "secret":   {secretKey},
        "response": {token},
    })
    if err != nil {
        return false, err
    }
    defer resp.Body.Close()

    var result captchaResponse
    if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
        return false, err
    }
    return result.Success, nil
}
```

## Middleware Implementation

`CaptchaMiddleware` extracts the token from the `X-Captcha-Token` header or the `g-recaptcha-response` form field, verifies server-side, and blocks on failure.

```go
// internal/middleware/captcha.go (continued)

// CaptchaMiddleware verifies reCAPTCHA or hCaptcha tokens server-side.
// secretKey: RECAPTCHA_SECRET or HCAPTCHA_SECRET env value
// verifyURL: use recaptchaVerifyURL or hcaptchaVerifyURL
func CaptchaMiddleware(secretKey, verifyURL string, logger *slog.Logger) gin.HandlerFunc {
    return func(c *gin.Context) {
        // Skip in test mode
        if os.Getenv("APP_ENV") == "test" {
            c.Next()
            return
        }

        // Extract token: prefer header, fall back to form field
        token := c.GetHeader("X-Captcha-Token")
        if token == "" {
            token = c.PostForm("g-recaptcha-response")
        }
        if strings.TrimSpace(token) == "" {
            c.AbortWithStatusJSON(http.StatusForbidden, gin.H{"error": "captcha token required"})
            return
        }

        ok, err := verifyCaptcha(c.Request.Context(), verifyURL, secretKey, token)
        if err != nil {
            logger.Error("captcha verification request failed", "error", err)
            c.AbortWithStatusJSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
            return
        }
        if !ok {
            c.AbortWithStatusJSON(http.StatusForbidden, gin.H{"error": "captcha verification failed"})
            return
        }

        c.Next()
    }
}
```

## hCaptcha Alternative

Same pattern, different endpoint and secret env var:

```go
// internal/middleware/captcha.go

func HCaptchaMiddleware(logger *slog.Logger) gin.HandlerFunc {
    secretKey := os.Getenv("HCAPTCHA_SECRET")
    return CaptchaMiddleware(secretKey, hcaptchaVerifyURL, logger)
}

func ReCaptchaMiddleware(logger *slog.Logger) gin.HandlerFunc {
    secretKey := os.Getenv("RECAPTCHA_SECRET")
    return CaptchaMiddleware(secretKey, recaptchaVerifyURL, logger)
}
```

## Route Application

Apply CAPTCHA only to public-facing forms that are abuse vectors. Do not apply to authenticated routes.

```go
// internal/router/router.go
logger := slog.Default()
recaptcha := middleware.ReCaptchaMiddleware(logger)

authGroup := r.Group("/auth")
authGroup.Use(middleware.IPRateLimiter(5, time.Minute))
{
    authGroup.POST("/register", recaptcha, authHandler.Register)
    authGroup.POST("/login", authHandler.Login)             // rate limiting is sufficient here
    authGroup.POST("/forgot-password", recaptcha, authHandler.ForgotPassword)
}

// Public contact/support forms
public := r.Group("/")
{
    public.POST("/contact", recaptcha, contactHandler.Submit)
}
```

## Configuration

Load secret from environment at startup; fail fast if missing in non-test mode.

```go
// internal/config/config.go
type CaptchaConfig struct {
    RecaptchaSecret string // RECAPTCHA_SECRET
    HCaptchaSecret  string // HCAPTCHA_SECRET
}

func LoadCaptchaConfig() CaptchaConfig {
    return CaptchaConfig{
        RecaptchaSecret: os.Getenv("RECAPTCHA_SECRET"),
        HCaptchaSecret:  os.Getenv("HCAPTCHA_SECRET"),
    }
}
```

## Security Notes

- **Never trust client-side validation alone**: The browser-side widget can be bypassed; always verify server-side.
- **Secret key is server-only**: Never expose `RECAPTCHA_SECRET` to the client or commit to source control.
- **Test mode bypass**: Check `APP_ENV == "test"` to skip verification in automated tests without mocking HTTP calls.
- **Score threshold (v3)**: For reCAPTCHA v3, parse `score` from the response and reject requests below your threshold (typically `0.5`).
- **IP forwarding**: If behind a proxy, pass the client IP in the `remoteip` field of the verification request for better accuracy.
