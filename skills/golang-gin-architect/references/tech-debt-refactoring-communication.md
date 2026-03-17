# Tech Debt — Refactoring Strategies and Stakeholder Communication

Companion to `tech-debt-identification-prioritization.md` (identification, categorization, measuring, prioritizing).

---

## Refactoring Strategies

### The Boy Scout Rule

> Leave the code better than you found it.

When touching a file for a feature, fix small debt items in the same PR. Don't make a separate "cleanup PR" for trivial changes.

**Applies to:** Code quality debt (naming, dead code, small duplications).

### The Strangler Fig Pattern

Gradually replace old code with new code, routing traffic between them.

```go
// Phase 1: New handler alongside old one
r.POST("/api/v1/orders", oldHandler.CreateOrder)
r.POST("/api/v2/orders", newHandler.CreateOrder)

// Phase 2: Route percentage of traffic to new
r.POST("/api/v1/orders", func(c *gin.Context) {
    if shouldUseNew(c) { // feature flag or percentage
        newHandler.CreateOrder(c)
        return
    }
    oldHandler.CreateOrder(c)
})

// Phase 3: All traffic to new, remove old
r.POST("/api/v1/orders", newHandler.CreateOrder)
```

**Applies to:** Large rewrites where you can't do a big-bang switch.

### The Parallel Change Pattern

1. Add new code alongside old code
2. Migrate callers one by one
3. Remove old code when no callers remain

```go
// Step 1: Add new method, keep old
// Deprecated: use GetByIDV2 which returns proper errors
func (r *UserRepo) GetByID(id string) *User { ... }

func (r *UserRepo) GetByIDV2(ctx context.Context, id string) (*User, error) { ... }

// Step 2: Migrate callers to GetByIDV2
// Step 3: Delete GetByID when all callers migrated
```

**Applies to:** Interface changes, function signature changes, data model migrations.

### Dedicated Debt Sprints

Allocate 10-20% of sprint capacity to tech debt embedded in every sprint — not a separate sprint.

---

## Communicating to Stakeholders

| Tech Debt Term | Business Translation |
|---|---|
| "We have tech debt" | "New features will take longer to build" |
| "We need to refactor" | "We're investing in speed for the next 6 months" |
| "Missing tests" | "We can't guarantee changes won't break existing features" |
| "Architecture debt" | "Adding the next 3 features will cost 2x what they should" |
| "Security debt" | "We have known vulnerabilities that could be exploited" |

### Quarterly Debt Report Template

```markdown
# Tech Debt Report — Q1 2026

## Summary
- Total debt items: 23 (down from 28 last quarter)
- Critical items: 2 (security: 0, reliability: 2)
- Estimated total effort: 15 dev-days
- Items resolved this quarter: 8

## Top 5 Items (by priority score)

| # | Description | Impact | Effort | Plan |
|---|---|---|---|---|
| 1 | No tests for payment flow | High (revenue risk) | 2d | Sprint 4 |
| 2 | Manual deployment process | High (slow releases) | 3d | Sprint 5 |
| 3 | No rate limiting | High (abuse risk) | 1d | Sprint 4 |

## Recommendation
Allocate 15% of Sprint 4-5 capacity to address items #1-3.
Expected outcome: faster feature delivery and reduced incident risk.
```

---

## Prevention

### At Code Review Time

- Does this PR add complexity that isn't needed now?
- Are there tests for the happy path AND error cases?
- Is the new code consistent with existing patterns?
- Are secrets hardcoded? (auto-check with `gitleaks`)
- Does this touch a file already in the debt inventory?

### At Design Time

- Use complexity assessment before building
- Write an ADR for non-obvious decisions
- Start with the simplest pattern that works
- Set up CI/CD and tests from day one

### Automated Guards

```yaml
# .golangci.yml — catches common debt patterns
linters:
  enable:
    - errcheck      # unchecked errors
    - govet         # suspicious constructs
    - staticcheck   # bugs, simplifications
    - unused        # unused code
    - gocognit      # cognitive complexity (threshold: 30)
    - funlen        # function length (default: 60 lines)
```

**Rule:** If a linter catches it automatically, you don't need to track it as debt. Fix it or configure the linter to accept it.
