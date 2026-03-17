# Auth Implementation — Claims, Token Generation & JWT Middleware

See also: `auth-implementation-handlers.md`

## Dependencies

```bash
go get github.com/golang-jwt/jwt/v5
go get golang.org/x/crypto
go get github.com/google/uuid
go get golang.org/x/time/rate
```

## Claims Struct

```go
// internal/auth/claims.go
package auth

import "github.com/golang-jwt/jwt/v5"

type Claims struct {
    jwt.RegisteredClaims
    UserID string `json:"uid"`
    Email  string `json:"email"`
    Role   string `json:"role"`
    // Add Roles []string for multi-role, TenantID for multi-tenancy
}

type RefreshClaims struct {
    jwt.RegisteredClaims
    // RegisteredClaims.ID carries the jti for per-token blacklisting.
}
```

## Token Generation and Validation

```go
// internal/auth/token.go
package auth

import (
    "errors"
    "fmt"
    "time"

    "github.com/golang-jwt/jwt/v5"
    "github.com/google/uuid"
)

type TokenConfig struct {
    AccessSecret  []byte
    RefreshSecret []byte
    AccessTTL     time.Duration
    RefreshTTL    time.Duration
    Issuer        string
    Audience      []string
}

func GenerateAccessToken(cfg TokenConfig, userID, email, role string) (string, error) {
    now := time.Now()
    claims := Claims{
        RegisteredClaims: jwt.RegisteredClaims{
            ID:        uuid.NewString(), // jti — required for blacklisting
            Subject:   userID,
            Issuer:    cfg.Issuer,
            Audience:  jwt.ClaimStrings(cfg.Audience),
            IssuedAt:  jwt.NewNumericDate(now),
            NotBefore: jwt.NewNumericDate(now),
            ExpiresAt: jwt.NewNumericDate(now.Add(cfg.AccessTTL)),
        },
        UserID: userID,
        Email:  email,
        Role:   role,
    }
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    signed, err := token.SignedString(cfg.AccessSecret)
    if err != nil {
        return "", fmt.Errorf("sign access token: %w", err)
    }
    return signed, nil
}

func GenerateRefreshToken(cfg TokenConfig, userID string) (string, error) {
    now := time.Now()
    claims := RefreshClaims{
        RegisteredClaims: jwt.RegisteredClaims{
            ID:        uuid.NewString(),
            Subject:   userID,
            Issuer:    cfg.Issuer,
            IssuedAt:  jwt.NewNumericDate(now),
            NotBefore: jwt.NewNumericDate(now),
            ExpiresAt: jwt.NewNumericDate(now.Add(cfg.RefreshTTL)),
        },
    }
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    signed, err := token.SignedString(cfg.RefreshSecret)
    if err != nil {
        return "", fmt.Errorf("sign refresh token: %w", err)
    }
    return signed, nil
}

func ParseAccessToken(cfg TokenConfig, tokenStr string) (*Claims, error) {
    token, err := jwt.ParseWithClaims(tokenStr, &Claims{}, func(t *jwt.Token) (any, error) {
        if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
            return nil, fmt.Errorf("unexpected signing method: %v", t.Header["alg"])
        }
        return cfg.AccessSecret, nil
    })
    if err != nil {
        if errors.Is(err, jwt.ErrTokenExpired) {
            return nil, fmt.Errorf("token expired: %w", err)
        }
        return nil, fmt.Errorf("invalid token: %w", err)
    }
    claims, ok := token.Claims.(*Claims)
    if !ok || !token.Valid {
        return nil, fmt.Errorf("invalid token claims")
    }
    return claims, nil
}
```

See also: `auth-middleware.md` for the `Auth` gin middleware that extracts and validates the Bearer token.
