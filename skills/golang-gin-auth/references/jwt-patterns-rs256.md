# JWT Patterns — RS256 vs HS256

See also: `jwt-patterns-tokens.md`, `jwt-patterns-blacklist.md`, `jwt-patterns-storage-and-csrf.md`

## RS256 vs HS256

| Aspect         | HS256 (HMAC-SHA256)         | RS256 (RSA-SHA256)                       |
|----------------|-----------------------------|------------------------------------------|
| Keys           | One shared secret           | Private key (sign) + Public key (verify) |
| Verification   | Needs the secret            | Needs only the public key                |
| Use case       | Single service / monolith   | Microservices, third-party verification  |
| Key management | Simple                      | More complex (cert rotation)             |

**HS256** — suitable for most single-service APIs (shown in SKILL.md).

**RS256** — use when multiple services or external clients need to verify tokens without the signing secret.

```go
// internal/auth/token_rsa.go
package auth

import (
    "crypto/rsa"
    "fmt"
    "time"

    "github.com/golang-jwt/jwt/v5"
    "github.com/google/uuid"
)

// GenerateAccessTokenRS256 signs with RSA private key.
func GenerateAccessTokenRS256(privateKey *rsa.PrivateKey, userID, email, role, issuer string, audience []string, ttl time.Duration) (string, error) {
    now := time.Now()
    claims := Claims{
        RegisteredClaims: jwt.RegisteredClaims{
            ID:        uuid.NewString(),
            Subject:   userID,
            Issuer:    issuer,
            Audience:  jwt.ClaimStrings(audience),
            IssuedAt:  jwt.NewNumericDate(now),
            NotBefore: jwt.NewNumericDate(now),
            ExpiresAt: jwt.NewNumericDate(now.Add(ttl)),
        },
        UserID: userID,
        Email:  email,
        Role:   role,
    }
    token := jwt.NewWithClaims(jwt.SigningMethodRS256, claims)
    signed, err := token.SignedString(privateKey)
    if err != nil {
        return "", fmt.Errorf("sign RS256 token: %w", err)
    }
    return signed, nil
}

// ParseAccessTokenRS256 verifies with RSA public key (safe to distribute).
func ParseAccessTokenRS256(publicKey *rsa.PublicKey, tokenStr string) (*Claims, error) {
    token, err := jwt.ParseWithClaims(tokenStr, &Claims{}, func(t *jwt.Token) (any, error) {
        if _, ok := t.Method.(*jwt.SigningMethodRSA); !ok {
            return nil, fmt.Errorf("unexpected signing method: %v", t.Header["alg"])
        }
        return publicKey, nil
    })
    if err != nil {
        return nil, fmt.Errorf("parse RS256 token: %w", err)
    }
    claims, ok := token.Claims.(*Claims)
    if !ok || !token.Valid {
        return nil, fmt.Errorf("invalid token claims")
    }
    return claims, nil
}
```

Load RSA keys from PEM files or environment at startup — never embed keys in source code.
