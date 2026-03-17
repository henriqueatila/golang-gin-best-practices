# ParadeDB — Advanced Search, Autocomplete, and Hybrid Search

## Advanced Search Features

### Fuzzy, Phrase, Boolean, Range Queries

```sql
-- Fuzzy: edit-distance matching
WHERE name @@@ paradedb.fuzzy_term('name', 'headphons', distance => 1)

-- Phrase: exact phrase
WHERE description @@@ paradedb.phrase('description', ARRAY['noise', 'cancelling'])

-- Boolean: must/should/must_not
WHERE products @@@ paradedb.boolean(
    must     => ARRAY[paradedb.term('category', 'electronics')],
    should   => ARRAY[paradedb.parse('name:wireless'), paradedb.parse('name:bluetooth')],
    must_not => ARRAY[paradedb.term('name', 'refurbished')])

-- Range: combine with boolean
WHERE products @@@ paradedb.boolean(must => ARRAY[
    paradedb.parse('name:headphones'),
    paradedb.range('rating', lower => 4.0, lower_inclusive => true),
    paradedb.range('price', upper => 150.0, upper_inclusive => true)])
```

### Snippet Highlighting

```sql
SELECT id, name,
    paradedb.snippet(id, field => 'description', max_num_chars => 200) AS excerpt
FROM products
WHERE products @@@ paradedb.parse('description:noise cancelling')
ORDER BY paradedb.score(id) DESC LIMIT 10;
-- excerpt: "...premium <b>noise</b> <b>cancelling</b> technology with..."
```

```go
func (r *searchRepository) SearchWithHighlight(ctx context.Context, query string, limit int) ([]productSnippetRow, error) {
    const sql = `SELECT id, name, paradedb.snippet(id, field => 'description', max_num_chars => 150) AS excerpt,
        paradedb.score(id) AS score FROM products WHERE products @@@ paradedb.parse($1) ORDER BY score DESC LIMIT $2`
    var rows []productSnippetRow
    if err := r.db.SelectContext(ctx, &rows, sql, query, limit); err != nil {
        return nil, fmt.Errorf("search with highlight: %w", err)
    }
    return rows, nil
}
```

---

## Autocomplete and Typeahead

```sql
-- Prefix matching
SELECT DISTINCT name FROM products
WHERE name @@@ paradedb.prefix('name', 'wire')
ORDER BY name LIMIT 10;

-- Fuzzy prefix (typo-tolerant typeahead)
SELECT DISTINCT name FROM products
WHERE name @@@ paradedb.fuzzy_phrase('name', 'wirelss head', distance => 1, prefix => true)
ORDER BY name LIMIT 10;
```

```go
func (r *searchRepository) Autocomplete(ctx context.Context, prefix string, limit int) ([]string, error) {
    if limit <= 0 || limit > 20 { limit = 10 }
    const sql = `SELECT DISTINCT name FROM products WHERE name @@@ paradedb.prefix('name', $1) ORDER BY name LIMIT $2`
    var suggestions []string
    if err := r.db.SelectContext(ctx, &suggestions, sql, prefix, limit); err != nil {
        return nil, fmt.Errorf("autocomplete: %w", err)
    }
    return suggestions, nil
}
```

---

## Hybrid Search (BM25 + pgvector)

Combines BM25 keyword relevance with vector semantic similarity using Reciprocal Rank Fusion (RRF).

**RRF formula:** `rrf_score = 1/(k + rank_bm25) + 1/(k + rank_vector)` where `k = 60`.

### Schema Requirements

```sql
ALTER TABLE products ADD COLUMN embedding VECTOR(1536);
CREATE INDEX products_embedding_hnsw_idx
    ON products USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);
```

### Hybrid SQL with RRF

```sql
WITH bm25_results AS (
    SELECT id, ROW_NUMBER() OVER (ORDER BY paradedb.score(id) DESC) AS bm25_rank
    FROM products WHERE products @@@ paradedb.parse($1) LIMIT 60
),
vector_results AS (
    SELECT id, ROW_NUMBER() OVER (ORDER BY embedding <=> $2) AS vector_rank
    FROM products WHERE embedding IS NOT NULL
    ORDER BY embedding <=> $2 LIMIT 60
),
rrf AS (
    SELECT COALESCE(b.id, v.id) AS id,
        (1.0 / (60 + COALESCE(b.bm25_rank, 61))) +
        (1.0 / (60 + COALESCE(v.vector_rank, 61))) AS rrf_score
    FROM bm25_results b FULL OUTER JOIN vector_results v USING (id)
)
SELECT p.id, p.name, p.category, p.price, r.rrf_score
FROM rrf r JOIN products p USING (id)
ORDER BY r.rrf_score DESC LIMIT $3;
```

### Go Implementation

```go
func (r *searchRepository) HybridSearch(ctx context.Context, textQuery string, embedding []float32, limit int) ([]domain.ProductResult, error) {
    // SQL: use the WITH bm25_results / vector_results / rrf CTE shown above, params: $1=textQuery $2=vec $3=limit
    vec := pgvector.NewVector(embedding)
    var rows []hybridResultRow
    if err := r.db.SelectContext(ctx, &rows, hybridSearchSQL, textQuery, vec, limit); err != nil {
        return nil, fmt.Errorf("hybrid search: %w", err)
    }
    results := make([]domain.ProductResult, len(rows))
    for i, row := range rows {
        results[i] = domain.ProductResult{ID: row.ID, Name: row.Name, Category: row.Category, Price: row.Price, Score: row.RRFScore}
    }
    return results, nil
}
```

> For embedding generation and pgvector setup: see [pgvector-embeddings.md](pgvector-embeddings.md).
