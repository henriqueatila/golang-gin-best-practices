# CAPTCHA Middleware — reCAPTCHA v2/v3 & hCaptcha

No additional Go packages needed. Uses only `net/http` and `encoding/json`.

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
    Score   float64 `json:"score"`  // reCAPTCHA v3 only
    Action  string  `json:"action"` // reCAPTCHA v3 only
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

```go
// CaptchaMiddleware verifies reCAPTCHA or hCaptcha tokens server-side.
func CaptchaMiddleware(secretKey, verifyURL string, logger *slog.Logger) gin.HandlerFunc {
    return func(c *gin.Context) {
        if os.Getenv("APP_ENV") == "test" {
            c.Next()
            return
        }

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

func HCaptchaMiddleware(logger *slog.Logger) gin.HandlerFunc {
    return CaptchaMiddleware(os.Getenv("HCAPTCHA_SECRET"), hcaptchaVerifyURL, logger)
}

func ReCaptchaMiddleware(logger *slog.Logger) gin.HandlerFunc {
    return CaptchaMiddleware(os.Getenv("RECAPTCHA_SECRET"), recaptchaVerifyURL, logger)
}
```

## Route Application

Apply CAPTCHA only to public-facing forms that are abuse vectors. Do not apply to authenticated routes.

```go
recaptcha := middleware.ReCaptchaMiddleware(logger)

authGroup := r.Group("/auth")
authGroup.Use(middleware.IPRateLimiter(5, time.Minute))
{
    authGroup.POST("/register", recaptcha, authHandler.Register)
    authGroup.POST("/login", authHandler.Login) // rate limiting is sufficient here
    authGroup.POST("/forgot-password", recaptcha, authHandler.ForgotPassword)
}

public := r.Group("/")
{ public.POST("/contact", recaptcha, contactHandler.Submit) }
```

## Security Notes

- **Never trust client-side validation alone**: The browser widget can be bypassed; always verify server-side.
- **Secret key is server-only**: Never expose `RECAPTCHA_SECRET` to the client or commit to source control.
- **Test mode bypass**: Check `APP_ENV == "test"` to skip verification in automated tests without mocking HTTP calls.
- **Score threshold (v3)**: For reCAPTCHA v3, parse `score` from the response and reject requests below your threshold (typically `0.5`).
- **IP forwarding**: If behind a proxy, pass the client IP in the `remoteip` field of the verification request for better accuracy.
