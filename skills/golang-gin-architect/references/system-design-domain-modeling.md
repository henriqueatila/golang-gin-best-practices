# System Design — Domain Modeling

Entities with behavior, value objects, and the decision gate for when domain modeling is warranted.

Companion to `system-design-bounded-contexts.md` (bounded context analysis, context mapping).

> **Default stance:** Skip domain modeling for simple CRUD. Use these tools when complexity is real and measured.

---

## The Decision Gate

```
Does my entity have business rules that enforce invariants?
  ├── No  → Struct + repository pattern. Stop here.
  └── Yes → Do I have complex state transitions?
      ├── No  → Methods on structs + service layer.
      └── Yes → Consider value objects and domain events.
                But: measure the complexity first.
```

**Practical heuristic:** If all your service methods look like `Create/Get/Update/Delete` with no branching business logic, you have a data API. Use domain modeling only when services have methods like `ConfirmOrder`, `CancelSubscription`, `ProcessRefund` — operations with rules and side effects.

---

## Entities with Behavior

```go
// internal/order/model.go
package order

type Order struct {
    ID        string
    UserID    string
    Status    OrderStatus
    Items     []OrderItem
    Total     int64 // cents — never float for money
    CreatedAt time.Time
    UpdatedAt time.Time
}

type OrderStatus string

const (
    OrderStatusDraft     OrderStatus = "draft"
    OrderStatusConfirmed OrderStatus = "confirmed"
    OrderStatusShipped   OrderStatus = "shipped"
    OrderStatusDelivered OrderStatus = "delivered"
    OrderStatusCancelled OrderStatus = "cancelled"
)

func (o *Order) Confirm() error {
    if o.Status != OrderStatusDraft {
        return fmt.Errorf("cannot confirm order in status %q", o.Status)
    }
    if len(o.Items) == 0 {
        return fmt.Errorf("cannot confirm empty order")
    }
    o.Status = OrderStatusConfirmed
    o.UpdatedAt = time.Now()
    return nil
}

func (o *Order) Cancel(reason string) error {
    switch o.Status {
    case OrderStatusDraft, OrderStatusConfirmed:
        o.Status = OrderStatusCancelled
        o.UpdatedAt = time.Now()
        return nil
    default:
        return fmt.Errorf("cannot cancel order in status %q", o.Status)
    }
}
```

---

## Value Objects

Immutable, compared by value (not ID). Use for `Money`, `Address`, `Email` with their own validation rules.

```go
// internal/shared/money.go
package shared

type Money struct {
    Amount   int64  // cents
    Currency string // ISO 4217: "USD", "EUR"
}

func NewMoney(amount int64, currency string) (Money, error) {
    if amount < 0 {
        return Money{}, fmt.Errorf("money amount cannot be negative: %d", amount)
    }
    if len(currency) != 3 {
        return Money{}, fmt.Errorf("invalid currency code %q", currency)
    }
    return Money{Amount: amount, Currency: currency}, nil
}

func (m Money) Add(other Money) (Money, error) {
    if m.Currency != other.Currency {
        return Money{}, fmt.Errorf("cannot add %s and %s", m.Currency, other.Currency)
    }
    return Money{Amount: m.Amount + other.Amount, Currency: m.Currency}, nil
}

func (m Money) IsZero() bool { return m.Amount == 0 }
```

---

## Cross-Skill References

- For bounded context analysis: see **[system-design-bounded-contexts.md](system-design-bounded-contexts.md)**
- For project structure by scale: see **[system-design-project-structure.md](system-design-project-structure.md)**
- For complexity budget: see **[complexity-assessment-budget.md](complexity-assessment-budget.md)**
