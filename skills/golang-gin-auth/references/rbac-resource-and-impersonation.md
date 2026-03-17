# RBAC — Resource-Level Authorization, Impersonation & Complete Example

See also: `rbac-middleware.md`, `rbac-hierarchy-and-tenant.md`

## Resource-Level Authorization

Verify the authenticated user owns or has access to the specific resource — not just the right role.

```go
// internal/handler/user_handler.go

// Update allows users to edit their own profile, or admins to edit any profile.
func (h *UserHandler) Update(c *gin.Context) {
    type uriParams struct {
        ID string `uri:"id" binding:"required"`
    }
    var params uriParams
    if err := c.ShouldBindURI(&params); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "invalid request"})
        return
    }

    claims := claimsFromContext(c)
    if claims == nil {
        c.JSON(http.StatusUnauthorized, gin.H{"error": "unauthorized"})
        return
    }

    if claims.UserID != params.ID && claims.Role != "admin" {
        c.JSON(http.StatusForbidden, gin.H{"error": "cannot modify another user's profile"})
        return
    }

    var req domain.UpdateUserRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "invalid request body"})
        return
    }

    user, err := h.svc.Update(c.Request.Context(), params.ID, req)
    if err != nil {
        handleServiceError(c, err, h.logger)
        return
    }

    c.JSON(http.StatusOK, user)
}
```

**Why in the handler, not middleware?** Resource ownership depends on business logic (the specific resource ID). Middleware runs before route parameters are bound — the handler is the right layer for this check.

---

## Admin Impersonation

Allows admins to act as another user for support or debugging. Inject `ImpersonatedBy` for audit logs.

```go
type Claims struct {
    jwt.RegisteredClaims
    UserID         string `json:"uid"`
    Email          string `json:"email"`
    Role           string `json:"role"`
    ImpersonatedBy string `json:"imp,omitempty"`
}

func (h *AdminHandler) Impersonate(c *gin.Context) {
    adminClaims := claimsFromContext(c)
    if adminClaims == nil || adminClaims.Role != "admin" {
        c.JSON(http.StatusForbidden, gin.H{"error": "admin access required"})
        return
    }

    var req struct {
        TargetUserID string `json:"target_user_id" binding:"required"`
    }
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "invalid request body"})
        return
    }

    target, err := h.userRepo.GetByID(c.Request.Context(), req.TargetUserID)
    if err != nil {
        handleServiceError(c, err, h.logger)
        return
    }

    // Prevent admin-to-admin privilege escalation
    if target.Role == "admin" || target.Role == "superadmin" {
        c.JSON(http.StatusForbidden, gin.H{"error": "cannot impersonate admin users"})
        return
    }

    claims := auth.Claims{
        RegisteredClaims: jwt.RegisteredClaims{
            Subject:   target.ID,
            IssuedAt:  jwt.NewNumericDate(time.Now()),
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(1 * time.Hour)),
        },
        UserID:         target.ID,
        Email:          target.Email,
        Role:           target.Role,
        ImpersonatedBy: adminClaims.UserID,
    }
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    signed, err := token.SignedString(h.tokenCfg.AccessSecret)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
        return
    }

    h.logger.Info("admin impersonation", "admin_id", adminClaims.UserID, "target_id", target.ID)
    c.JSON(http.StatusOK, gin.H{"access_token": signed})
}
```

**Critical:** Log all impersonation events. Never allow impersonating another admin.

See also: `rbac-route-wiring.md` for the complete RBAC route registration example.
