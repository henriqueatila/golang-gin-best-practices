# pgvector — Similarity Queries and Go Integration

## Similarity Queries

### K-nearest neighbors (KNN)

```sql
SELECT id, title, 1 - (embedding <=> $1) AS similarity
FROM documents
ORDER BY embedding <=> $1
LIMIT $2;
```

### Filtered similarity — tenant-scoped search

```sql
SELECT id, title, content, 1 - (embedding <=> $1) AS similarity
FROM documents
WHERE tenant_id = $2
ORDER BY embedding <=> $1
LIMIT 10;
```

**Important:** If the WHERE clause is highly selective, the planner may choose a sequential scan over the vector index. This is often correct. Use `EXPLAIN ANALYZE` to verify.

### Partial index for filtered search

```sql
-- Only indexes active documents
CREATE INDEX idx_docs_active_embedding ON documents
    USING hnsw (embedding vector_cosine_ops)
    WHERE deleted_at IS NULL;
```

### Similarity threshold

```sql
SELECT id, title, 1 - (embedding <=> $1) AS similarity
FROM documents
WHERE tenant_id = $2
  AND 1 - (embedding <=> $1) > 0.75
ORDER BY embedding <=> $1
LIMIT 20;
```

### Pagination with vector search

Standard OFFSET pagination works but is not efficient for deep pages. For most RAG use cases, only the top-K results are needed.

```sql
-- Keyset pagination: track last seen distance
SELECT id, title, embedding <=> $1 AS distance
FROM documents
WHERE tenant_id = $2
  AND embedding <=> $1 > $3  -- $3 = last seen distance from previous page
ORDER BY embedding <=> $1
LIMIT 10;
```

---

## Go Integration with pgvector-go

### Install

```bash
go get github.com/pgvector/pgvector-go
```

`pgvector.Vector` implements `driver.Valuer` and `sql.Scanner` — serializes/deserializes the PostgreSQL `vector` wire format automatically with `sqlx`.

### Row struct with embedding

```go
// internal/repository/document_row.go
package repository

import (
    "time"
    "github.com/pgvector/pgvector-go"
)

type documentRow struct {
    ID         string          `db:"id"`
    TenantID   string          `db:"tenant_id"`
    Title      string          `db:"title"`
    Content    string          `db:"content"`
    Metadata   []byte          `db:"metadata"`
    Embedding  pgvector.Vector `db:"embedding"`
    Model      string          `db:"model"`
    TokenCount *int            `db:"token_count"`
    CreatedAt  time.Time       `db:"created_at"`
    UpdatedAt  time.Time       `db:"updated_at"`
}
```

### Repository: store, search, delete

```go
type documentRepository struct{ db *sqlx.DB }

func (r *documentRepository) Store(ctx context.Context, doc *domain.Document) error {
    const q = `INSERT INTO documents (id, tenant_id, title, content, metadata, embedding, model, token_count)
        VALUES (:id, :tenant_id, :title, :content, :metadata, :embedding, :model, :token_count)`
    row := documentInsertRow{
        ID: doc.ID, TenantID: doc.TenantID, Title: doc.Title,
        Content: doc.Content, Metadata: doc.MetadataJSON,
        Embedding: pgvector.NewVector(doc.Embedding), Model: doc.Model, TokenCount: doc.TokenCount,
    }
    if _, err := r.db.NamedExecContext(ctx, q, row); err != nil {
        return fmt.Errorf("document.Store: %w", err)
    }
    return nil
}

func (r *documentRepository) Search(ctx context.Context, tenantID string, queryEmbedding []float32, topK int) ([]domain.SearchResult, error) {
    const q = `SELECT id, title, content, 1 - (embedding <=> $1) AS similarity
        FROM documents WHERE tenant_id = $2 ORDER BY embedding <=> $1 LIMIT $3`
    var rows []struct {
        ID string `db:"id"`; Title string `db:"title"`; Content string `db:"content"`; Similarity float64 `db:"similarity"`
    }
    if err := r.db.SelectContext(ctx, &rows, q, pgvector.NewVector(queryEmbedding), tenantID, topK); err != nil {
        return nil, fmt.Errorf("document.Search: %w", err)
    }
    results := make([]domain.SearchResult, len(rows))
    for i, row := range rows {
        results[i] = domain.SearchResult{ID: row.ID, Title: row.Title, Content: row.Content, Similarity: row.Similarity}
    }
    return results, nil
}

func (r *documentRepository) Delete(ctx context.Context, id, tenantID string) error {
    result, err := r.db.ExecContext(ctx, `DELETE FROM documents WHERE id = $1 AND tenant_id = $2`, id, tenantID)
    if err != nil { return fmt.Errorf("document.Delete: %w", err) }
    if n, _ := result.RowsAffected(); n == 0 { return domain.ErrNotFound }
    return nil
}
```


> For embedding generation with OpenAI and batch processing patterns, see [pgvector-capacity-and-performance.md](pgvector-capacity-and-performance.md).
