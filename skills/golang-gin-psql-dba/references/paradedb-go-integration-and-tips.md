# ParadeDB — Go Integration Pattern, pg_analytics, and Performance Tips

## pg_analytics (Brief)

```sql
CREATE EXTENSION IF NOT EXISTS pg_analytics;

CREATE TABLE order_events (
    id         BIGINT GENERATED ALWAYS AS IDENTITY,
    user_id    UUID NOT NULL,
    event_type TEXT NOT NULL,
    revenue    NUMERIC(10,2),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
) USING parquet;  -- columnar storage
```

| Use Case | Recommendation |
|---|---|
| Aggregation, GROUP BY, reporting | pg_analytics (columnar scan is faster) |
| OLTP inserts, point lookups by PK | Standard heap table |
| Mixed OLTP + analytics on same data | Partition: heap for recent rows, pg_analytics for cold history |

**Typical pattern:** OLTP data lands in a heap table. A scheduled job copies completed records into a pg_analytics table for dashboards.

---

## Full Go Integration Pattern

```go
// internal/domain/search.go
package domain

type SearchRequest struct {
    Q        string  `form:"q"         binding:"required,min=2,max=200"`
    Category string  `form:"category"`
    MinPrice float64 `form:"min_price"`
    MaxPrice float64 `form:"max_price"`
    Page     int     `form:"page"`
    Limit    int     `form:"limit"`
}

type ProductResult struct {
    ID       int64   `json:"id"`
    Name     string  `json:"name"`
    Category string  `json:"category"`
    Price    float64 `json:"price"`
    Score    float64 `json:"score,omitempty"`
    Excerpt  string  `json:"excerpt,omitempty"`
}

type SearchRepository interface {
    Search(ctx context.Context, req SearchRequest) ([]ProductResult, int, error)
    Autocomplete(ctx context.Context, prefix string, limit int) ([]string, error)
}
```

```go
// internal/repository/search_repository.go
func (r *searchRepository) Search(ctx context.Context, req domain.SearchRequest) ([]domain.ProductResult, int, error) {
    limit := req.Limit
    if limit <= 0 || limit > 50 { limit = 20 }
    page := req.Page
    if page <= 0 { page = 1 }
    offset := (page - 1) * limit

    filters := []string{}
    args := []any{req.Q}
    idx := 2

    if req.Category != "" {
        filters = append(filters, fmt.Sprintf("paradedb.term('category', $%d)", idx))
        args = append(args, req.Category); idx++
    }
    if req.MaxPrice > 0 {
        filters = append(filters, fmt.Sprintf("paradedb.range('price', upper => $%d::real, upper_inclusive => true)", idx))
        args = append(args, req.MaxPrice); idx++
    }

    whereClause := "products @@@ paradedb.parse($1)"
    if len(filters) > 0 {
        whereClause = fmt.Sprintf(
            "products @@@ paradedb.boolean(must => ARRAY[paradedb.parse($1), %s])",
            strings.Join(filters, ", "))
    }

    args = append(args, limit, offset)
    sql := fmt.Sprintf(`
        SELECT id, name, category, price,
            paradedb.score(id) AS score,
            paradedb.snippet(id, field => 'description', max_num_chars => 150) AS excerpt,
            COUNT(*) OVER () AS total
        FROM products
        WHERE %s
        ORDER BY score DESC, id DESC
        LIMIT $%d OFFSET $%d`, whereClause, idx, idx+1)

    var rows []searchResultRow
    if err := r.db.SelectContext(ctx, &rows, sql, args...); err != nil {
        return nil, 0, fmt.Errorf("search query: %w", err)
    }

    results := make([]domain.ProductResult, len(rows))
    var total int
    for i, row := range rows {
        results[i] = domain.ProductResult{ID: row.ID, Name: row.Name, Category: row.Category, Price: row.Price, Score: row.Score, Excerpt: row.Excerpt}
        if i == 0 { total = row.Total }
    }
    return results, total, nil }
```

Routes: `GET /search` → `searchHandler.Search` (ShouldBindQuery → search.Search → JSON), `GET /search/autocomplete` → Autocomplete.

---

## Performance Tips

### Index Size

```sql
SELECT indexrelname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE indexrelname = 'products_search_idx';
```

BM25 indexes are typically 1–3x the size of the indexed text columns.

### Reindex After Bulk Loads

```sql
CALL paradedb.drop_bm25_index('products_search_idx');
CALL paradedb.create_bm25_index( /* same config */ );
```

ParadeDB keeps the index in sync via triggers. Manual reindex only needed after bulk operations bypassing triggers or field config changes.

### ParadeDB vs External Search

| Criteria | ParadeDB | Elasticsearch / Meilisearch |
|---|---|---|
| Existing PostgreSQL stack | Preferred — no new infra | Adds operational complexity |
| Sub-100ms search on <100M rows | Yes | Yes |
| Distributed search across billions of rows | Consider external | Designed for this |
| Transactional consistency (search + write in same tx) | Yes — native ACID | No |
| Operational cost | Low (PostgreSQL only) | High (separate cluster) |

**Decision shortcut:** One PostgreSQL instance + need better search than `tsvector` → ParadeDB. Move to external only at PostgreSQL's scale ceiling or when distributed cross-region search is required.
