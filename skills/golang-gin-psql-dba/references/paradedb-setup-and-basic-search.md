# ParadeDB — Setup, BM25 Index, and Basic Queries

ParadeDB brings Lucene-quality full-text search into PostgreSQL via the `pg_search` extension. BM25 ranking, fuzzy matching, phrase search, field boosting, and highlighting — all through standard SQL. No external search cluster required.

## ParadeDB vs tsvector

| Feature | `tsvector` + GIN | ParadeDB `pg_search` |
|---|---|---|
| Ranking algorithm | TF-IDF (basic) | **BM25** (Lucene-quality) |
| Fuzzy matching | No (pg_trgm workaround) | Native, configurable edit distance |
| Field boosting | No | Yes |
| Snippet highlighting | No | Yes |
| Hybrid BM25 + vector | No | Yes (with pgvector) |
| Setup complexity | Zero (built-in) | Requires ParadeDB image/extension |

**Rule:** Start with `tsvector` + GIN for simple keyword search. Switch to ParadeDB when you need BM25 relevance, fuzzy tolerance, highlighting, or hybrid semantic search.

## Docker Setup

```yaml
# docker-compose.yml (development)
services:
  db:
    image: paradedb/paradedb:latest
    restart: unless-stopped
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: appdb
      PARADEDB_TELEMETRY: false
    ports: ["5432:5432"]
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./migrations:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d appdb"]
      interval: 10s
      timeout: 5s
      retries: 5
volumes:
  pgdata:
```

## pg_search Setup

```sql
-- Run once per database
CREATE EXTENSION IF NOT EXISTS pg_search;
```

### Products Table Schema

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

```sql
CALL paradedb.create_bm25_index(
    index_name     => 'products_search_idx',
    table_name     => 'products',
    key_field      => 'id',
    text_fields    => paradedb.field('name',        tokenizer => paradedb.tokenizer('en_stem'), boost => 3.0)
                   || paradedb.field('description', tokenizer => paradedb.tokenizer('en_stem'), boost => 1.0)
                   || paradedb.field('category',    tokenizer => paradedb.tokenizer('keyword')),
    numeric_fields => paradedb.field('price') || paradedb.field('rating'),
    boolean_fields => paradedb.field('in_stock')
);

-- Drop before recreating after schema changes
CALL paradedb.drop_bm25_index('products_search_idx');
```

| Parameter | Values | Notes |
|---|---|---|
| `tokenizer` | `en_stem`, `lowercase`, `keyword`, `whitespace`, `raw` | `en_stem` for natural language; `keyword` for exact category values |
| `boost` | `REAL` (default `1.0`) | Higher = more weight in BM25 score |
| `stored` | `true` / `false` | `true` enables snippet highlighting |

## Basic BM25 Queries

```sql
-- Simple text search
SELECT id, name, category, price
FROM products
WHERE name @@@ 'wireless headphones'
ORDER BY paradedb.score(id) DESC LIMIT 20;

-- Multi-field search with score
SELECT id, name, category, price, paradedb.score(id) AS relevance_score
FROM products
WHERE products @@@ paradedb.parse('name:headphones OR description:headphones')
ORDER BY relevance_score DESC LIMIT 20;

-- Combine BM25 with filters
SELECT id, name, price, paradedb.score(id) AS score
FROM products
WHERE products @@@ paradedb.boolean(
    must   => ARRAY[paradedb.parse('name:headphones OR description:headphones')],
    filter => ARRAY[
        paradedb.term('in_stock', true),
        paradedb.range('price', upper => 200.0, upper_inclusive => true)
    ]
)
ORDER BY score DESC LIMIT 20;
```

### Go Repository

```go
func (r *searchRepository) BasicSearch(ctx context.Context, query string, limit, offset int) ([]domain.ProductResult, error) {
    const sql = `
        SELECT id, name, category, price, paradedb.score(id) AS score
        FROM products
        WHERE name @@@ $1 OR description @@@ $1
        ORDER BY score DESC, id DESC
        LIMIT $2 OFFSET $3`
    var rows []productSearchRow
    if err := r.db.SelectContext(ctx, &rows, sql, query, limit, offset); err != nil {
        return nil, fmt.Errorf("search: %w", err)
    }
    results := make([]domain.ProductResult, len(rows))
    for i, row := range rows {
        results[i] = domain.ProductResult{ID: row.ID, Name: row.Name, Category: row.Category, Price: row.Price, Score: row.Score}
    }
    return results, nil
}
```
