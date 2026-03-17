# System Design — C4 Model for Go APIs

When and how to use C4 diagrams for Go Gin API architecture documentation.

> **Default stance:** A CRUD API with a single team does NOT need C4 diagrams. Use when complexity is real and measured, not anticipated.

---

## When to Use Each Level

| Level | Use when |
|---|---|
| **Context** | Always — one diagram, 5 min to draw, answers "what does this system do and who talks to it?" |
| **Container** | You have multiple deployable units (API + worker + frontend + DB) |
| **Component** | A single container has enough internal complexity that new devs get lost |
| **Code** | Almost never. Use Go doc comments + package structure instead. |

If you have a single Gin API talking to one PostgreSQL database: draw Context, stop there.

---

## Level 1 — Context Diagram

Shows the system boundary and its actors/external systems. One box per external thing.

```mermaid
C4Context
    title System Context — Order API

    Person(customer, "Customer", "Browses catalog, places orders")
    Person(admin, "Admin", "Manages inventory, views reports")

    System(orderApi, "Order API", "Go/Gin REST API handling orders, catalog, payments")

    System_Ext(paymentGw, "Payment Gateway", "Stripe — processes card payments")
    System_Ext(emailSvc, "Email Service", "SendGrid — transactional email")
    System_Ext(shippingSvc, "Shipping API", "FedEx — label generation, tracking")

    Rel(customer, orderApi, "Uses", "HTTPS/JSON")
    Rel(admin, orderApi, "Uses", "HTTPS/JSON")
    Rel(orderApi, paymentGw, "Charges cards", "HTTPS")
    Rel(orderApi, emailSvc, "Sends confirmations", "HTTPS")
    Rel(orderApi, shippingSvc, "Creates shipments", "HTTPS")
```

**Rule:** If you can't fit this on one page, you have too many external dependencies — that's the real problem to solve.

---

## Level 2 — Container Diagram

Shows deployable units. Use when you have more than one process running.

```mermaid
C4Container
    title Container Diagram — Order Platform

    Person(customer, "Customer")

    Container(webApp, "Web App", "React SPA", "Browser-side UI")
    Container(apiGateway, "API Gateway", "nginx", "TLS termination, routing")
    Container(orderApi, "Order API", "Go/Gin", "Core business logic")
    Container(workerService, "Background Worker", "Go", "Async jobs: emails, shipping")
    ContainerDb(postgres, "PostgreSQL", "Database", "Orders, users, catalog")
    ContainerDb(redis, "Redis", "Cache + Queue", "Sessions, job queue")

    Rel(customer, webApp, "Uses", "HTTPS")
    Rel(webApp, apiGateway, "Calls", "HTTPS/JSON")
    Rel(apiGateway, orderApi, "Routes to", "HTTP")
    Rel(orderApi, postgres, "Reads/writes", "TCP/5432")
    Rel(orderApi, redis, "Caches, enqueues", "TCP/6379")
    Rel(workerService, redis, "Dequeues from", "TCP/6379")
    Rel(workerService, postgres, "Updates state", "TCP/5432")
```

---

## Level 3 — Component Diagram

Shows internal structure of one container. Use only when a container has 5+ distinct responsibilities and onboarding new devs takes more than a day.

```mermaid
C4Component
    title Component Diagram — Order API (internal)

    Component(userCmp, "User Module", "internal/user", "Registration, auth, profile")
    Component(orderCmp, "Order Module", "internal/order", "Cart, checkout, order lifecycle")
    Component(catalogCmp, "Catalog Module", "internal/catalog", "Products, categories, search")
    Component(notifCmp, "Notification Module", "internal/notification", "Email, push triggers")
    Component(platform, "Platform", "internal/platform", "DB pool, auth middleware, logger")

    Rel(orderCmp, userCmp, "Looks up user", "interface UserLookup")
    Rel(orderCmp, catalogCmp, "Validates items", "interface ProductLookup")
    Rel(orderCmp, notifCmp, "Emits events", "interface EventPublisher")
    Rel(userCmp, platform, "Uses DB, auth")
    Rel(orderCmp, platform, "Uses DB")
    Rel(catalogCmp, platform, "Uses DB")
```

**Go-specific mapping:** Each C4 component maps to a Go package under `internal/`. The arrows between components must be satisfied by Go interfaces — this prevents import cycles and enforces boundaries.

## Level 4 — Code Level

Skip it. Go package documentation, `go doc`, and readable package structure do this better with zero maintenance cost.

---

## Cross-Skill References

- For bounded context analysis: see **[system-design-bounded-contexts.md](system-design-bounded-contexts.md)**
- For project structure by scale: see **[system-design-project-structure.md](system-design-project-structure.md)**
- For complexity budget: see **[complexity-assessment.md](complexity-assessment.md)**
