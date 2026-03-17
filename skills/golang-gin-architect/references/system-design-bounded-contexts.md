# System Design — Bounded Context Analysis

When and how to identify bounded contexts in Go Gin APIs.

Companion to `system-design-domain-modeling.md` (entities with behavior, value objects).

> **Default stance:** Skip domain modeling for simple CRUD. Use these tools when complexity is real and measured.

---

## When to Use

- You have a "God model" — one `User` struct imported everywhere, growing to 40+ fields
- Different teams argue about what fields belong in a shared struct
- Changing one part of the domain model breaks unrelated features
- You're evaluating whether to extract a module into a separate service

Skip it when you have a simple app with one team and < 10 domain entities.

---

## Identifying Implicit Bounded Contexts

Symptoms of implicit bounded contexts inside a monolith:

- The word "User" means different things in different features (auth = credentials+roles; billing = payment methods; notification = email+push token)
- Package A and Package B both define a `Product` struct with different fields
- You have a `types` or `models` package that everything imports — and it keeps growing
- Merge conflicts regularly happen in the same files for unrelated features

---

## How to Find Your Bounded Contexts

1. **Noun workshop:** List every noun (entity/concept) in the domain.
2. **Group by team/feature ownership:** Which nouns naturally cluster together?
3. **Identify ubiquitous language:** Do "User" and "Customer" mean the same thing in all clusters?
4. **Look at change frequency:** Files that always change together are in the same context.
5. **Find the seams:** Where do contexts communicate? Those are your integration points — model them as interfaces.

---

## Package Boundary Design

```go
// internal/order/ports.go
// "ports" = interfaces this module needs from outside
package order

import "context"

// UserLookup is order's view of what it needs from the user domain.
// Defined HERE (order module owns its dependencies), not in internal/user.
type UserLookup interface {
    GetByID(ctx context.Context, id string) (UserInfo, error)
}

// UserInfo is order's projection — NOT the full user domain model.
// Adding a field to user.User does NOT require changing this struct.
type UserInfo struct {
    ID    string
    Name  string
    Email string
}
```

```go
// internal/order/service.go
package order

type Service struct {
    repo   Repository
    users  UserLookup  // injected — order doesn't care about the concrete type
    logger *slog.Logger
}

func NewService(repo Repository, users UserLookup, logger *slog.Logger) *Service {
    return &Service{repo: repo, users: users, logger: logger}
}
```

---

## Context Mapping Patterns (Lightweight)

| Pattern | Use when | Go implementation |
|---|---|---|
| **Shared Kernel** | Two modules share a small, stable set of types (e.g., `Money`, `Address`) | `internal/shared/` package — kept tiny, changes require both teams to agree |
| **Anti-Corruption Layer** | Integrating with an external system whose model you don't control (Stripe, external ERP) | Adapter struct in `internal/[domain]/adapters/` |
| **Conformist** | Downstream consumer of an upstream API, too costly to translate | Accept their model, document the dependency explicitly |

```go
// Anti-corruption layer: Stripe payment intent → internal PaymentResult
// internal/billing/adapters/stripe_adapter.go
package adapters

type StripeAdapter struct{ client *stripe.Client }

func (a *StripeAdapter) Charge(amount int64, currency, token string) (*billing.PaymentResult, error) {
    pi, err := a.client.PaymentIntents.New(nil)
    if err != nil {
        return nil, fmt.Errorf("stripe charge: %w", err)
    }
    return &billing.PaymentResult{
        TransactionID: pi.ID,
        Amount:        amount,
        Status:        translateStatus(pi.Status),
    }, nil
}
```

---

## Cross-Skill References

- For domain modeling (entities, value objects): see **[system-design-domain-modeling.md](system-design-domain-modeling.md)**
- For project structure by scale: see **[system-design-project-structure.md](system-design-project-structure.md)**
- For C4 diagrams: see **[system-design-c4-model.md](system-design-c4-model.md)**
