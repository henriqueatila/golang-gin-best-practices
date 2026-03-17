# Auth Implementation — Login, Register, Routes & Getting Current User

See also: `auth-implementation-core.md`

## Registration — Password Hashing

Always hash passwords with bcrypt before storing. Use cost >= 12 for production.

```go
// internal/handler/auth_handler.go (Register method)
func (h *AuthHandler) Register(c *gin.Context) {
    var req struct {
        Email    string `json:"email"    binding:"required,email"`
        Password string `json:"password" binding:"required,min=8"`
    }
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "invalid request body"})
        return
    }

    hash, err := bcrypt.GenerateFromPassword([]byte(req.Password), 12)
    if err != nil {
        h.logger.Error("failed to hash password", "error", err)
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
        return
    }

    user, err := h.userRepo.Create(c.Request.Context(), domain.CreateUserRequest{
        Email:        req.Email,
        PasswordHash: string(hash),
    })
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
        return
    }

    c.JSON(http.StatusCreated, gin.H{"user_id": user.ID})
}
```

## Login Handler

> **Security note:** Return generic `"invalid credentials"` for both wrong email and wrong password — never leak whether the email exists.

```go
type AuthHandler struct {
    userRepo domain.UserRepository
    tokenCfg auth.TokenConfig
    logger   *slog.Logger
}

func (h *AuthHandler) Login(c *gin.Context) {
    var req struct {
        Email    string `json:"email"    binding:"required,email"`
        Password string `json:"password" binding:"required"`
    }
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "invalid request body"})
        return
    }

    user, err := h.userRepo.GetByEmail(c.Request.Context(), req.Email)
    if err != nil {
        c.JSON(http.StatusUnauthorized, gin.H{"error": "invalid credentials"})
        return
    }

    if err := bcrypt.CompareHashAndPassword([]byte(user.PasswordHash), []byte(req.Password)); err != nil {
        c.JSON(http.StatusUnauthorized, gin.H{"error": "invalid credentials"})
        return
    }

    accessToken, err := auth.GenerateAccessToken(h.tokenCfg, user.ID, user.Email, user.Role)
    if err != nil {
        h.logger.Error("failed to generate access token", "error", err, "user_id", user.ID)
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
        return
    }

    refreshToken, err := auth.GenerateRefreshToken(h.tokenCfg, user.ID)
    if err != nil {
        h.logger.Error("failed to generate refresh token", "error", err, "user_id", user.ID)
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
        return
    }

    c.JSON(http.StatusOK, gin.H{
        "access_token":  accessToken,
        "refresh_token": refreshToken,
    })
}
```

## Protected Route Wiring

```go
r := gin.New()
r.Use(middleware.Logger(logger))
r.Use(middleware.Recovery(logger))

api := r.Group("/api/v1")

authRoutes := api.Group("/auth")
authRoutes.Use(middleware.IPRateLimiter(rate.Every(12*time.Second), 5))
{
    authRoutes.POST("/login", authHandler.Login)
    authRoutes.POST("/register", authHandler.Register)
    authRoutes.POST("/refresh", authHandler.Refresh)
}

protected := api.Group("")
protected.Use(middleware.Auth(cfg, logger))
{
    protected.GET("/users/:id", userHandler.GetByID)
    protected.PUT("/users/:id", userHandler.Update)
    protected.DELETE("/users/:id", userHandler.Delete)

    admin := protected.Group("/admin")
    admin.Use(middleware.RequireRole("admin"))
    { admin.GET("/users", userHandler.List) }
}
```

See also: `auth-middleware.md` for getting the current user from context (`ClaimsKey`, `UserIDKey`).
See also: `auth-ip-rate-limiter.md` for the `IPRateLimiter` middleware used on auth routes.
