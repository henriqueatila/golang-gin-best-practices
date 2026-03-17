# Tech Debt — Identification, Categorization, and Prioritization

Practical framework for managing technical debt in Go Gin API projects.

---

## What Is Tech Debt (And What Isn't)

**Tech debt IS:** code shortcuts taken with awareness for speed; outdated patterns that slow new feature development; missing tests for critical paths; hardcoded values; tight coupling.

**Tech debt is NOT:** code you don't like the style of; a different approach than you'd have chosen; code written by someone less experienced; features not built yet.

**Rule:** If it doesn't slow down current development or create real risk, it's not debt worth tracking.

---

## The Debt Quadrant

```
                    Deliberate                    Inadvertent
              ┌─────────────────────┬─────────────────────┐
  Reckless    │ "We know this is    │ "What's a           │
              │  wrong but ship"    │  repository pattern?"│
              │ → High priority fix │ → Training + refactor│
              ├─────────────────────┼─────────────────────┤
  Prudent     │ "Simple now,        │ "Now we know how    │
              │  refactor later"    │  it should've been" │
              │ → Schedule when     │ → Refactor when     │
              │   pain is real      │   touching the code │
              └─────────────────────┴─────────────────────┘
```

Prudent deliberate debt is OK — document it (ADR) and move on. Reckless deliberate debt compounds — fix it.

---

## Identifying Debt

### Code Signals

| Signal | Debt Type | Severity |
|---|---|---|
| `// TODO`, `// HACK`, `// FIXME` | Explicit | Check each one |
| Duplicated code blocks | DRY violation | Medium |
| Functions > 50 lines | Complexity | Low-Medium |
| No tests for business logic | Safety | High |
| Hardcoded URLs, secrets, magic numbers | Configuration | Medium-High |
| Error ignored: `_ = someFunc()` | Reliability | High |
| `fmt.Println` instead of `slog` | Observability | Low |

### Architecture Signals

| Signal | Debt Type | Severity |
|---|---|---|
| Handler contains SQL queries | Layer violation | High |
| Circular package dependencies | Coupling | High |
| No database migrations (manual DDL) | Process | High |
| No health check endpoint | Operations | Medium |
| Secrets in code or git | Security | Critical |
| No rate limiting on public endpoints | Security | High |

### Process Signals

| Signal | What It Means |
|---|---|
| Same bug appears twice | Missing test coverage |
| New devs take > 2 weeks to be productive | Poor docs or complex code |
| "Don't touch that, it works" | Technical tombstone — debt is severe |
| Deploy takes > 30 minutes | Build/CI debt |

---

## Categorizing Debt

| Category | Examples | Priority |
|---|---|---|
| **Security** | Hardcoded secrets, SQL injection, missing auth | Fix immediately |
| **Reliability** | Missing error handling, ignored errors | Sprint priority |
| **Testing** | No tests for critical paths, flaky tests | Sprint priority |
| **Architecture** | Layer violations, circular deps, coupling | Plan & schedule |
| **Code Quality** | Duplicated code, long functions, naming | Boy scout rule |
| **Documentation** | Missing API docs, no ADRs | Ongoing |

---

## Measuring Debt

### Lightweight Debt Inventory

```markdown
| ID | Description | Category | Severity | Effort | Files Affected |
|---|---|---|---|---|---|
| TD-001 | No tests for payment flow | Testing | High | 2d | internal/payment/* |
| TD-002 | User handler has SQL queries | Architecture | High | 1d | internal/handler/user.go |
| TD-003 | Duplicated validation logic | Code Quality | Medium | 0.5d | internal/handler/*.go |
```

### Effort Scoring

| Score | Meaning |
|---|---|
| XS (< 1h) | Remove dead code, fix TODO, add missing error check |
| S (1h-4h) | Extract function, add input validation, add test |
| M (1d-2d) | Split large file, add test suite, implement middleware |
| L (3d-5d) | Restructure package, implement missing layer |
| XL (1w+) | Change architecture pattern, migrate database |

---

## Prioritizing Debt

```
                High Impact
                    │
        ┌───────────┼───────────┐
        │  SCHEDULE  │  FIX NOW  │
  Low ──┼───────────┼───────────┼── High Effort
  Effort│  BOY SCOUT│  EVALUATE │
        └───────────┼───────────┘
                    │
                Low Impact
```

**Priority score = (Frequency + Severity + Spread) - Effort** — higher = fix sooner.

Score each item 1-5 on: Frequency (how often it slows someone down), Severity (how bad when it bites), Effort (how hard to fix), Spread (how many files/features affected).

---

## See Also

- `tech-debt-refactoring-communication.md` — refactoring strategies (boy scout, strangler fig, parallel change), stakeholder communication, prevention
