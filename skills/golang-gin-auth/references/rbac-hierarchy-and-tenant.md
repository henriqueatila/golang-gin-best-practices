# RBAC — Role Hierarchy & Multi-Tenant Authorization

See also: `rbac-middleware.md`, `rbac-resource-and-impersonation.md`

## Role Hierarchy

Encode hierarchy as a map — higher-ranked roles inherit lower-ranked permissions.

```go
// internal/auth/roles.go
package auth

var roleRank = map[string]int{
    "guest":      0,
    "user":       1,
    "moderator":  2,
    "admin":      3,
    "superadmin": 4,
}

// HasRoleAtLeast returns true if userRole has rank >= requiredRole.
// Unknown roles are denied — never defaulted to guest.
func HasRoleAtLeast(userRole, requiredRole string) bool {
    ur, ok1 := roleRank[userRole]
    rr, ok2 := roleRank[requiredRole]
    return ok1 && ok2 && ur >= rr
}
```

```go
// pkg/middleware/rbac.go

// RequireMinRole allows any role with rank >= minRole.
func RequireMinRole(minRole string) gin.HandlerFunc {
    return func(c *gin.Context) {
        claims := claimsFromContext(c)
        if claims == nil {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "unauthorized"})
            return
        }
        if !auth.HasRoleAtLeast(claims.Role, minRole) {
            c.AbortWithStatusJSON(http.StatusForbidden, gin.H{"error": "insufficient privileges"})
            return
        }
        c.Next()
    }
}
```

---

## Multi-Tenant Authorization

In multi-tenant systems, users belong to a tenant. Requests must be validated for both authentication AND tenant membership.

**Add TenantID to claims:**

```go
type Claims struct {
    jwt.RegisteredClaims
    UserID   string `json:"uid"`
    Email    string `json:"email"`
    Role     string `json:"role"`
    TenantID string `json:"tid"`
}
```

**Tenant middleware:**

```go
// pkg/middleware/tenant.go
const TenantIDKey = "tenant_id"

func RequireTenant() gin.HandlerFunc {
    return func(c *gin.Context) {
        claims := claimsFromContext(c)
        if claims == nil || claims.TenantID == "" {
            c.AbortWithStatusJSON(http.StatusForbidden, gin.H{"error": "tenant context required"})
            return
        }
        c.Set(TenantIDKey, claims.TenantID)
        c.Next()
    }
}
```

**Enforce tenant isolation in repositories — always scope queries to the tenant:**

```go
func (r *gormPostRepository) GetByID(ctx context.Context, tenantID, postID string) (*domain.Post, error) {
    var m PostModel
    err := r.db.WithContext(ctx).
        Where("tenant_id = ? AND id = ?", tenantID, postID).
        First(&m).Error
    if err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, domain.ErrNotFound
        }
        return nil, domain.ErrInternal
    }
    return m.ToDomain(), nil
}
```

**Handler reads tenant from context:**

```go
func (h *PostHandler) GetByID(c *gin.Context) {
    tenantID := c.GetString(middleware.TenantIDKey)

    type uriParams struct {
        ID string `uri:"id" binding:"required"`
    }
    var params uriParams
    if err := c.ShouldBindURI(&params); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "invalid request"})
        return
    }

    post, err := h.svc.GetByID(c.Request.Context(), tenantID, params.ID)
    if err != nil {
        handleServiceError(c, err, h.logger)
        return
    }

    c.JSON(http.StatusOK, post)
}
```
