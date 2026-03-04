# ParadeDB Full-Text Search Reference

ParadeDB is a PostgreSQL extension suite that brings Lucene-quality full-text search directly into your existing PostgreSQL database. The primary extension, `pg_search`, implements BM25 ranking with fuzzy matching, phrase search, boolean queries, field boosting, and search result highlighting — all through standard SQL. No external search cluster, no data synchronization, no operational overhead beyond the PostgreSQL instance you already run. ParadeDB is production-ready, backed by Y Combinator, and increasingly adopted by teams that need real search quality without adding Elasticsearch to their stack.

> **Architectural note:** ParadeDB is growing fast. Core search features are stable and production-ready. Some advanced analytics features are newer — check the release notes for your version.

## Table of Contents

1. [ParadeDB Overview](#paradedb-overview)
2. [Docker Setup](#docker-setup)
3. [pg_search Setup](#pg_search-setup)
4. [Basic BM25 Queries](#basic-bm25-queries)
5. [Advanced Search Features](#advanced-search-features)
6. [Autocomplete and Typeahead](#autocomplete-and-typeahead)
7. [Hybrid Search (BM25 + pgvector)](#hybrid-search-bm25--pgvector)
8. [pg_analytics (Brief)](#pg_analytics-brief)
9. [Go Integration Pattern](#go-integration-pattern)
10. [Performance Tips](#performance-tips)

---

## ParadeDB Overview

### What It Is

ParadeDB ships two PostgreSQL extensions:

| Extension | Purpose |
|---|---|
| **pg_search** | BM25 full-text search — the main focus of this file |
| **pg_analytics** | Column-oriented analytical storage (DuckDB integration) |

**pg_search vs PostgreSQL built-in `tsvector`:**

| Feature | `tsvector` + GIN | ParadeDB `pg_search` |
|---|---|---|
| Ranking algorithm | TF-IDF (basic) | **BM25** (Lucene-quality) |
| Fuzzy matching | No (pg_trgm workaround) | Native, configurable edit distance |
| Field boosting | No | Yes — boost title over body |
| Faceted search | No | Yes — counts per category |
| Snippet highlighting | No | Yes — extract match context |
| Phrase proximity | Limited | Yes |
| Hybrid BM25 + vector | No | Yes (with pgvector) |
| Setup complexity | Zero (built-in) | Requires ParadeDB image/extension |

**Rule:** Start with `tsvector` + GIN for simple keyword search on a single language. Switch to ParadeDB when you need BM25 relevance ranking, fuzzy tolerance, highlighting, or hybrid semantic search.

---

## Docker Setup

ParadeDB ships a Docker image based on the official PostgreSQL image with both extensions pre-installed.

```yaml
# docker-compose.yml (development)
services:
  db:
    image: paradedb/paradedb:latest  # Based on postgres:16
    restart: unless-stopped
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: appdb
      PARADEDB_TELEMETRY: false      # Opt out of anonymous telemetry
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./migrations:/docker-entrypoint-initdb.d  # Auto-run SQL on first start
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d appdb"]
      interval: 10s
      timeout: 5s
      retries: 5

  app:
    build: .
    depends_on:
      db:
        condition: service_healthy
    environment:
      DATABASE_URL: postgres://app:secret@db:5432/appdb?sslmode=disable

volumes:
  pgdata:
```

> For full orchestration with PgBouncer, replicas, and production hardening: see the **gin-deploy** skill.

---

## pg_search Setup

### Enable the Extension

```sql
-- Run once per database, typically in your first migration
CREATE EXTENSION IF NOT EXISTS pg_search;
```

### Schema: Products Table Example

All examples in this file use a `products` table. Create it before indexing:

```sql
CREATE TABLE products (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name        TEXT        NOT NULL,
    description TEXT,
    category    TEXT        NOT NULL,
    price       NUMERIC(10, 2) NOT NULL,
    in_stock    BOOLEAN     NOT NULL DEFAULT true,
    rating      REAL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Creating a BM25 Index

`paradedb.create_bm25_index()` replaces the manual `CREATE INDEX ... USING bm25` syntax. It creates the index and registers field configurations for tokenization and boosting.

```sql
CALL paradedb.create_bm25_index(
    index_name      => 'products_search_idx',
    table_name      => 'products',
    key_field       => 'id',
    text_fields     => paradedb.field('name',        tokenizer => paradedb.tokenizer('en_stem'), boost => 3.0)
                    || paradedb.field('description', tokenizer => paradedb.tokenizer('en_stem'), boost => 1.0)
                    || paradedb.field('category',    tokenizer => paradedb.tokenizer('keyword')),
    numeric_fields  => paradedb.field('price') || paradedb.field('rating'),
    boolean_fields  => paradedb.field('in_stock')
);
```

**Field configuration options:**

| Parameter | Values | Notes |
|---|---|---|
| `tokenizer` | `en_stem`, `lowercase`, `keyword`, `whitespace`, `raw` | `en_stem` for natural language; `keyword` for exact category values |
| `boost` | `REAL` (default `1.0`) | Higher = more weight in BM25 score |
| `stored` | `true` / `false` | `true` enables snippet highlighting for that field |

**Drop and recreate:**

```sql
-- Drop before recreating (e.g. schema changes)
CALL paradedb.drop_bm25_index('products_search_idx');
```

---

## Basic BM25 Queries

ParadeDB extends SQL with `@@@` (the BM25 search operator) and `paradedb.score()`.

### Simple Text Search

```sql
SELECT id, name, category, price
FROM products
WHERE name @@@ 'wireless headphones'
ORDER BY paradedb.score(id) DESC
LIMIT 20;
```

### Multi-Field Search with Score

```sql
-- Search across name and description; score reflects BM25 relevance
SELECT
    id,
    name,
    category,
    price,
    paradedb.score(id) AS relevance_score
FROM products
WHERE products @@@ paradedb.parse('name:headphones OR description:headphones')
ORDER BY relevance_score DESC
LIMIT 20;
```

### Combining BM25 with Filters

```sql
-- Full-text search scoped to in-stock items under $200
SELECT id, name, price, paradedb.score(id) AS score
FROM products
WHERE products @@@ paradedb.boolean(
    must   => ARRAY[paradedb.parse('name:headphones OR description:headphones')],
    filter => ARRAY[
        paradedb.term('in_stock', true),
        paradedb.range('price', upper => 200.0, upper_inclusive => true)
    ]
)
ORDER BY score DESC
LIMIT 20;
```

### Pagination with Score Cursor

```sql
-- Offset-based pagination (simple; use keyset for large result sets)
SELECT id, name, price, paradedb.score(id) AS score
FROM products
WHERE name @@@ 'headphones'
ORDER BY score DESC, id DESC
LIMIT 20 OFFSET 40;
```

### Go Repository Function

```go
// internal/repository/product_search_repository.go
package repository

import (
    "context"
    "fmt"

    "github.com/jmoiron/sqlx"
    "myapp/internal/domain"
)

type productSearchRow struct {
    ID       int64   `db:"id"`
    Name     string  `db:"name"`
    Category string  `db:"category"`
    Price    float64 `db:"price"`
    Score    float64 `db:"score"`
}

func (r *searchRepository) BasicSearch(
    ctx context.Context,
    query string,
    limit, offset int,
) ([]domain.ProductResult, error) {
    const sql = `
        SELECT id, name, category, price, paradedb.score(id) AS score
        FROM products
        WHERE name @@@ $1 OR description @@@ $1
        ORDER BY score DESC, id DESC
        LIMIT $2 OFFSET $3
    `
    var rows []productSearchRow
    if err := r.db.SelectContext(ctx, &rows, sql, query, limit, offset); err != nil {
        return nil, fmt.Errorf("search: %w", err)
    }
    results := make([]domain.ProductResult, len(rows))
    for i, row := range rows {
        results[i] = domain.ProductResult{
            ID: row.ID, Name: row.Name,
            Category: row.Category, Price: row.Price,
            Score: row.Score,
        }
    }
    return results, nil
}
```

---

## Advanced Search Features

### Fuzzy Matching (Typo Tolerance)

```sql
-- Match "headphons", "headphone", "headphones" within edit distance 1
SELECT id, name, paradedb.score(id) AS score
FROM products
WHERE name @@@ paradedb.fuzzy_term('name', 'headphons', distance => 1)
ORDER BY score DESC
LIMIT 20;
```

### Phrase Search

```sql
-- Match "noise cancelling" as an exact phrase (word order matters)
SELECT id, name, paradedb.score(id) AS score
FROM products
WHERE description @@@ paradedb.phrase('description', ARRAY['noise', 'cancelling'])
ORDER BY score DESC;
```

### Boolean Queries (must / should / must_not)

```sql
-- must: all terms required; should: boosts score; must_not: excludes
SELECT id, name, paradedb.score(id) AS score
FROM products
WHERE products @@@ paradedb.boolean(
    must     => ARRAY[paradedb.term('category', 'electronics')],
    should   => ARRAY[paradedb.parse('name:wireless'), paradedb.parse('name:bluetooth')],
    must_not => ARRAY[paradedb.term('name', 'refurbished')]
)
ORDER BY score DESC;
```

### Range Queries on Numeric Fields

```sql
-- Products rated 4.0–5.0, priced under $150
SELECT id, name, rating, price, paradedb.score(id) AS score
FROM products
WHERE products @@@ paradedb.boolean(
    must => ARRAY[
        paradedb.parse('name:headphones'),
        paradedb.range('rating', lower => 4.0, lower_inclusive => true),
        paradedb.range('price', upper => 150.0, upper_inclusive => true)
    ]
)
ORDER BY score DESC;
```

### Regex Search

```sql
-- Match product names starting with "ultra" (case-insensitive by tokenizer)
SELECT id, name, paradedb.score(id) AS score
FROM products
WHERE name @@@ paradedb.regex('name', 'ultra.*')
ORDER BY score DESC;
```

### Snippet Highlighting

```sql
-- Returns HTML-highlighted excerpts around matched terms
SELECT
    id,
    name,
    paradedb.snippet(id, field => 'description', max_num_chars => 200) AS excerpt
FROM products
WHERE products @@@ paradedb.parse('description:noise cancelling')
ORDER BY paradedb.score(id) DESC
LIMIT 10;
-- excerpt example: "...premium <b>noise</b> <b>cancelling</b> technology with..."
```

**Go — fetch snippet in result row:**

```go
type productSnippetRow struct {
    ID      int64  `db:"id"`
    Name    string `db:"name"`
    Excerpt string `db:"excerpt"`
    Score   float64 `db:"score"`
}

func (r *searchRepository) SearchWithHighlight(
    ctx context.Context,
    query string,
    limit int,
) ([]productSnippetRow, error) {
    const sql = `
        SELECT
            id,
            name,
            paradedb.snippet(id, field => 'description', max_num_chars => 150) AS excerpt,
            paradedb.score(id) AS score
        FROM products
        WHERE products @@@ paradedb.parse($1)
        ORDER BY score DESC
        LIMIT $2
    `
    var rows []productSnippetRow
    if err := r.db.SelectContext(ctx, &rows, sql, query, limit); err != nil {
        return nil, fmt.Errorf("search with highlight: %w", err)
    }
    return rows, nil
}
```

---

## Autocomplete and Typeahead

### Prefix Matching

```sql
-- Match any product name starting with the typed prefix
SELECT DISTINCT name
FROM products
WHERE name @@@ paradedb.prefix('name', 'wire')
ORDER BY name
LIMIT 10;
```

### Fuzzy Prefix (Typo-Tolerant Typeahead)

```sql
-- Handles "wirelss" → "wireless" while still doing prefix matching
SELECT DISTINCT name
FROM products
WHERE name @@@ paradedb.fuzzy_phrase('name', 'wirelss head', distance => 1, prefix => true)
ORDER BY name
LIMIT 10;
```

### Go Handler + Repository

```go
// internal/domain/search.go
package domain

type AutocompleteRequest struct {
    Q     string `form:"q" binding:"required,min=2,max=100"`
    Limit int    `form:"limit"`
}

type AutocompleteResponse struct {
    Suggestions []string `json:"suggestions"`
}

// internal/repository/autocomplete_repository.go
package repository

import (
    "context"
    "fmt"

    "github.com/jmoiron/sqlx"
)

func (r *searchRepository) Autocomplete(
    ctx context.Context,
    prefix string,
    limit int,
) ([]string, error) {
    if limit <= 0 || limit > 20 {
        limit = 10
    }
    const sql = `
        SELECT DISTINCT name
        FROM products
        WHERE name @@@ paradedb.prefix('name', $1)
        ORDER BY name
        LIMIT $2
    `
    var suggestions []string
    if err := r.db.SelectContext(ctx, &suggestions, sql, prefix, limit); err != nil {
        return nil, fmt.Errorf("autocomplete: %w", err)
    }
    return suggestions, nil
}

// internal/handler/search_handler.go
package handler

import (
    "log/slog"
    "net/http"

    "github.com/gin-gonic/gin"
    "myapp/internal/domain"
)

type SearchHandler struct {
    search domain.SearchRepository
    logger *slog.Logger
}

func (h *SearchHandler) Autocomplete(c *gin.Context) {
    var req domain.AutocompleteRequest
    if err := c.ShouldBindQuery(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    suggestions, err := h.search.Autocomplete(c.Request.Context(), req.Q, req.Limit)
    if err != nil {
        h.logger.ErrorContext(c.Request.Context(), "autocomplete failed", "error", err)
        c.JSON(http.StatusInternalServerError, gin.H{"error": "search unavailable"})
        return
    }

    c.JSON(http.StatusOK, domain.AutocompleteResponse{Suggestions: suggestions})
}
```

---

## Hybrid Search (BM25 + pgvector)

Hybrid search combines BM25 keyword relevance with vector semantic similarity. It handles both "find documents containing these words" (BM25) and "find documents *about* this concept" (embeddings). Reciprocal Rank Fusion (RRF) merges the two ranked lists without needing to normalize scores across different scales.

**RRF formula:** `rrf_score = 1 / (k + rank_bm25) + 1 / (k + rank_vector)` where `k = 60` (standard constant).

### Schema Requirements

```sql
-- Add embedding column (requires pgvector extension)
ALTER TABLE products
    ADD COLUMN embedding VECTOR(1536);  -- Dimension matches your model

-- Vector index for similarity search
CREATE INDEX products_embedding_hnsw_idx
    ON products USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);
```

### Hybrid SQL Query with RRF

```sql
WITH bm25_results AS (
    SELECT
        id,
        ROW_NUMBER() OVER (ORDER BY paradedb.score(id) DESC) AS bm25_rank
    FROM products
    WHERE products @@@ paradedb.parse($1)   -- $1: text query
    LIMIT 60
),
vector_results AS (
    SELECT
        id,
        ROW_NUMBER() OVER (ORDER BY embedding <=> $2) AS vector_rank
    FROM products
    WHERE embedding IS NOT NULL
    ORDER BY embedding <=> $2              -- $2: query embedding ([]float32)
    LIMIT 60
),
rrf AS (
    SELECT
        COALESCE(b.id, v.id) AS id,
        (1.0 / (60 + COALESCE(b.bm25_rank, 61)))   +
        (1.0 / (60 + COALESCE(v.vector_rank, 61))) AS rrf_score
    FROM bm25_results b
    FULL OUTER JOIN vector_results v USING (id)
)
SELECT p.id, p.name, p.category, p.price, r.rrf_score
FROM rrf r
JOIN products p USING (id)
ORDER BY r.rrf_score DESC
LIMIT $3;                                  -- $3: result limit
```

### Go Function for Hybrid Search

```go
// internal/repository/hybrid_search_repository.go
package repository

import (
    "context"
    "fmt"

    "github.com/jmoiron/sqlx"
    "github.com/pgvector/pgvector-go"
    "myapp/internal/domain"
)

const hybridSearchSQL = `
    WITH bm25_results AS (
        SELECT id, ROW_NUMBER() OVER (ORDER BY paradedb.score(id) DESC) AS bm25_rank
        FROM products
        WHERE products @@@ paradedb.parse($1)
        LIMIT 60
    ),
    vector_results AS (
        SELECT id, ROW_NUMBER() OVER (ORDER BY embedding <=> $2) AS vector_rank
        FROM products
        WHERE embedding IS NOT NULL
        ORDER BY embedding <=> $2
        LIMIT 60
    ),
    rrf AS (
        SELECT
            COALESCE(b.id, v.id) AS id,
            (1.0 / (60 + COALESCE(b.bm25_rank, 61))) +
            (1.0 / (60 + COALESCE(v.vector_rank, 61))) AS rrf_score
        FROM bm25_results b
        FULL OUTER JOIN vector_results v USING (id)
    )
    SELECT p.id, p.name, p.category, p.price, r.rrf_score
    FROM rrf r
    JOIN products p USING (id)
    ORDER BY r.rrf_score DESC
    LIMIT $3
`

type hybridResultRow struct {
    ID       int64   `db:"id"`
    Name     string  `db:"name"`
    Category string  `db:"category"`
    Price    float64 `db:"price"`
    RRFScore float64 `db:"rrf_score"`
}

func (r *searchRepository) HybridSearch(
    ctx context.Context,
    textQuery string,
    embedding []float32,
    limit int,
) ([]domain.ProductResult, error) {
    vec := pgvector.NewVector(embedding)
    var rows []hybridResultRow
    if err := r.db.SelectContext(ctx, &rows, hybridSearchSQL, textQuery, vec, limit); err != nil {
        return nil, fmt.Errorf("hybrid search: %w", err)
    }
    results := make([]domain.ProductResult, len(rows))
    for i, row := range rows {
        results[i] = domain.ProductResult{
            ID: row.ID, Name: row.Name,
            Category: row.Category, Price: row.Price,
            Score: row.RRFScore,
        }
    }
    return results, nil
}
```

> For generating embeddings and pgvector setup (HNSW index, distance functions, capacity planning): see [pgvector-embeddings.md](pgvector-embeddings.md).

---

## pg_analytics (Brief)

`pg_analytics` embeds DuckDB's columnar execution engine into PostgreSQL. It stores data in column-oriented Parquet files optimized for analytical aggregation.

```sql
CREATE EXTENSION IF NOT EXISTS pg_analytics;

-- Designate a table as a columnar table
CREATE TABLE order_events (
    id         BIGINT GENERATED ALWAYS AS IDENTITY,
    user_id    UUID NOT NULL,
    event_type TEXT NOT NULL,
    revenue    NUMERIC(10,2),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
) USING parquet;  -- columnar storage
```

**When to use pg_analytics:**

| Use Case | Recommendation |
|---|---|
| Aggregation, GROUP BY, reporting | pg_analytics (columnar scan is faster) |
| OLTP inserts, point lookups by PK | Standard heap table |
| Mixed OLTP + analytics on same data | Partition: heap for recent rows, pg_analytics for cold history |
| Joining search results with analytics | Both extensions can coexist in the same query |

**Typical pattern:** OLTP data lands in a standard PostgreSQL table. A scheduled job (pg_cron or application) copies completed/settled records into a pg_analytics table for dashboards and reporting queries.

---

## Go Integration Pattern

Complete, wired-up search stack: repository interface, structs, handler, and routing.

```go
// internal/domain/search.go
package domain

type SearchRequest struct {
    Q        string  `form:"q"        binding:"required,min=2,max=200"`
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

type SearchResponse struct {
    Results []ProductResult `json:"results"`
    Total   int             `json:"total"`
    Page    int             `json:"page"`
    Limit   int             `json:"limit"`
}

type SearchRepository interface {
    Search(ctx context.Context, req SearchRequest) ([]ProductResult, int, error)
    Autocomplete(ctx context.Context, prefix string, limit int) ([]string, error)
}

// internal/repository/search_repository.go
package repository

import (
    "context"
    "fmt"
    "strings"

    "github.com/jmoiron/sqlx"
    "myapp/internal/domain"
)

type searchRepository struct {
    db *sqlx.DB
}

func NewSearchRepository(db *sqlx.DB) domain.SearchRepository {
    return &searchRepository{db: db}
}

type searchResultRow struct {
    ID       int64   `db:"id"`
    Name     string  `db:"name"`
    Category string  `db:"category"`
    Price    float64 `db:"price"`
    Score    float64 `db:"score"`
    Excerpt  string  `db:"excerpt"`
    Total    int     `db:"total"`
}

func (r *searchRepository) Search(
    ctx context.Context,
    req domain.SearchRequest,
) ([]domain.ProductResult, int, error) {
    limit := req.Limit
    if limit <= 0 || limit > 50 {
        limit = 20
    }
    page := req.Page
    if page <= 0 {
        page = 1
    }
    offset := (page - 1) * limit

    // Build BM25 filter chain
    filters := []string{}
    args := []any{req.Q}
    idx := 2

    if req.Category != "" {
        filters = append(filters, fmt.Sprintf("paradedb.term('category', $%d)", idx))
        args = append(args, req.Category)
        idx++
    }
    if req.MaxPrice > 0 {
        filters = append(filters, fmt.Sprintf(
            "paradedb.range('price', upper => $%d::real, upper_inclusive => true)", idx))
        args = append(args, req.MaxPrice)
        idx++
    }

    whereClause := "products @@@ paradedb.parse($1)"
    if len(filters) > 0 {
        whereClause = fmt.Sprintf(
            "products @@@ paradedb.boolean(must => ARRAY[paradedb.parse($1), %s])",
            strings.Join(filters, ", "))
    }

    args = append(args, limit, offset)
    sql := fmt.Sprintf(`
        SELECT
            id, name, category, price,
            paradedb.score(id)                                          AS score,
            paradedb.snippet(id, field => 'description',
                             max_num_chars => 150)                      AS excerpt,
            COUNT(*) OVER ()                                            AS total
        FROM products
        WHERE %s
        ORDER BY score DESC, id DESC
        LIMIT $%d OFFSET $%d
    `, whereClause, idx, idx+1)

    var rows []searchResultRow
    if err := r.db.SelectContext(ctx, &rows, sql, args...); err != nil {
        return nil, 0, fmt.Errorf("search query: %w", err)
    }

    results := make([]domain.ProductResult, len(rows))
    total := 0
    for i, row := range rows {
        results[i] = domain.ProductResult{
            ID: row.ID, Name: row.Name, Category: row.Category,
            Price: row.Price, Score: row.Score, Excerpt: row.Excerpt,
        }
        if i == 0 {
            total = row.Total
        }
    }
    return results, total, nil
}
```

**Handler (thin — binds request, calls repo, formats response):**

```go
// internal/handler/search_handler.go
package handler

import (
    "log/slog"
    "net/http"

    "github.com/gin-gonic/gin"
    "myapp/internal/domain"
)

func (h *SearchHandler) Search(c *gin.Context) {
    var req domain.SearchRequest
    if err := c.ShouldBindQuery(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    limit := req.Limit
    if limit <= 0 {
        limit = 20
    }

    results, total, err := h.search.Search(c.Request.Context(), req)
    if err != nil {
        h.logger.ErrorContext(c.Request.Context(), "search failed", "error", err, "query", req.Q)
        c.JSON(http.StatusInternalServerError, gin.H{"error": "search unavailable"})
        return
    }

    c.JSON(http.StatusOK, domain.SearchResponse{
        Results: results,
        Total:   total,
        Page:    req.Page,
        Limit:   limit,
    })
}
```

**Route registration:**

```go
// cmd/api/routes.go
searchHandler := handler.NewSearchHandler(searchRepo, logger)
api := r.Group("/api/v1")
{
    api.GET("/search",          searchHandler.Search)
    api.GET("/search/autocomplete", searchHandler.Autocomplete)
}
```

---

## Performance Tips

### Index Size Estimation

```sql
-- Check BM25 index size on disk
SELECT
    indexrelname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE indexrelname = 'products_search_idx';
```

BM25 indexes are typically 1–3x the size of the text columns they index. Factor this into storage planning before indexing very large text columns.

### Reindex Strategy

```sql
-- Rebuild after heavy bulk inserts or schema changes
CALL paradedb.drop_bm25_index('products_search_idx');
CALL paradedb.create_bm25_index( /* same config */ );
```

ParadeDB automatically keeps the index in sync with DML (INSERT/UPDATE/DELETE) via triggers. Manual reindex is only needed after bulk operations that bypass triggers or after changing field configuration.

### When to Use ParadeDB vs External Search

| Criteria | ParadeDB | External (Elasticsearch / Meilisearch) |
|---|---|---|
| Existing PostgreSQL stack | Preferred — no new infra | Adds operational complexity |
| Sub-100ms search on <100M rows | Yes | Yes |
| Distributed search across billions of rows | Consider external | Designed for this |
| Faceted navigation with 50+ facet values | Yes | Yes (better UI tooling) |
| ML-trained ranking (Learning to Rank) | Not yet | Elasticsearch supports |
| Team already operates Elastic/Meilisearch | Keep existing | No migration needed |
| Transactional consistency (search + write in same tx) | Yes — native ACID | No |
| Operational cost | Low (PostgreSQL only) | High (separate cluster) |

**Decision shortcut:** If your team manages one PostgreSQL instance and needs search better than `tsvector`, ParadeDB is the lowest-friction choice. Move to an external search cluster only when you hit PostgreSQL's scale ceiling or need distributed replication of the search index across regions.
