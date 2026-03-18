# Defensive Query Patterns — Error Handling, LIKE Escape, Table Aliases

See also: `sqlx-patterns-queries.md`, `setup-repositories.md`

## Rule #1: NEVER Ignore Database Errors

Every `db.Get()`, `db.Select()`, `db.Exec()`, and `db.QueryRow().Scan()` call can fail. Ignoring the error returns zero-values, corrupts responses, and masks outages.

```go
// ❌ DANGEROUS — silently returns zeros if DB is down
r.db.Get(&stats.Total, "SELECT COUNT(*) FROM users WHERE active = true")
r.db.QueryRow("SELECT COUNT(*) ...").Scan(&stats.Active, &stats.Blocked)

// ✅ SAFE — propagates error to handler, which returns 500
if err := r.db.Get(&stats.Total, "SELECT COUNT(*) FROM users WHERE active = true"); err != nil {
    return nil, fmt.Errorf("active users count: %w", err)
}
if err := r.db.QueryRow("SELECT COUNT(*) ...").Scan(&stats.Active, &stats.Blocked); err != nil {
    return nil, fmt.Errorf("user counts: %w", err)
}
```

**Rule:** If a function returns `error`, you MUST check it. The only exception is fire-and-forget audit logging — and even then, log the error:

```go
// Audit: fire-and-forget but still log failures
if _, err := r.db.Exec("INSERT INTO audit_logs ...", ...); err != nil {
    slog.Error("audit log failed", "action", entry.Action, "error", err)
}
```

## Rule #2: Escape LIKE Metacharacters in User Search Input

The `%` and `_` characters are LIKE wildcards. If user input contains them, the query matches unintended patterns. This can cause performance degradation (full table scans) or information leaks.

```go
// ❌ DANGEROUS — user input "100%" becomes "ILIKE '%100%%'" (matches everything with "100")
// But user input "%" becomes "ILIKE '%%%'" (matches EVERYTHING)
fb.add("data::text ILIKE $%d", "%"+params.Search+"%")

// ✅ SAFE — escape metacharacters before wrapping with wildcards
func escapeLike(s string) string {
    s = strings.ReplaceAll(s, `\`, `\\`)
    s = strings.ReplaceAll(s, `%`, `\%`)
    s = strings.ReplaceAll(s, `_`, `\_`)
    return s
}

fb.add("data::text ILIKE $%d", "%"+escapeLike(params.Search)+"%")
```

**Rule:** ALWAYS escape `\`, `%`, and `_` in user input before using in LIKE/ILIKE clauses. This applies to both raw SQL and query builders.

## Rule #3: Always Use Table Alias in JOINed Queries

When a query has JOINs, unqualified column names can be ambiguous (both tables have `id`, `created_at`, etc.). Always use the table alias prefix.

```go
// ❌ AMBIGUOUS — "id" exists in both cobrancas and users tables
dataQuery := fmt.Sprintf(`SELECT c.* FROM cobrancas c JOIN users u ON c.user_id = u.id %s %s`,
    baseWhere, paginate(params, "", "ORDER BY id DESC"))
//                                  ^^               ^^ ambiguous!

// ✅ EXPLICIT — table alias used everywhere
dataQuery := fmt.Sprintf(`SELECT c.* FROM cobrancas c JOIN users u ON c.user_id = u.id %s %s`,
    baseWhere, paginate(params, "c", "ORDER BY c.id DESC"))
//                                  ^^^              ^^^^ clear!
```

**Rule:** In any query with JOINs:
- Pass the main table alias to helper functions (`paginate`, `orderBy`, etc.)
- Use the alias in ORDER BY, WHERE, and SELECT clauses
- Be consistent — if ListByEmpresa uses alias `"v"`, ListByHolding must too

## Rule #4: Validate Sort Fields Against a Whitelist

Dynamic ORDER BY from user input is an SQL injection vector. Always validate against a whitelist of allowed fields.

```go
// ✅ SAFE — only whitelisted fields allowed in ORDER BY
var sortableFields = map[string]bool{
    "id": true, "created_at": true, "updated_at": true,
    "status": true, "name": true, "email": true,
}

func orderByClause(sort, tableAlias, defaultOrder string) string {
    if sort == "" {
        return defaultOrder
    }
    field := strings.TrimPrefix(sort, "-")
    if !sortableFields[field] {
        return defaultOrder // reject unknown fields
    }
    dir := "ASC"
    if strings.HasPrefix(sort, "-") {
        dir = "DESC"
    }
    prefix := ""
    if tableAlias != "" {
        prefix = tableAlias + "."
    }
    return fmt.Sprintf("ORDER BY %s%s %s", prefix, field, dir)
}
```

## Summary Checklist

Before marking a repository method "done," verify:

- [ ] Every `db.Get`, `db.Select`, `db.Exec`, `db.QueryRow().Scan()` error is checked
- [ ] LIKE/ILIKE user input has `%`, `_`, `\` escaped
- [ ] JOINed queries use explicit table aliases in ORDER BY, WHERE, paginate
- [ ] Dynamic ORDER BY validated against a field whitelist
- [ ] Parameterized queries used everywhere (no `fmt.Sprintf` with user input in SQL)
- [ ] Fire-and-forget operations (audit, analytics) log errors via `slog`
