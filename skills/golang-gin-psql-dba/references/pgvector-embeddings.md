# pgvector Embeddings Reference

pgvector is an open-source PostgreSQL extension that adds a native `vector` column type and similarity-search operators, turning a standard PostgreSQL database into a vector store. It powers semantic search, retrieval-augmented generation (RAG), recommendation engines, and image similarity — all without a separate vector database. This reference covers extension setup, schema design, index selection, distance functions, similarity queries, Go integration via `pgvector-go` and `sqlx`, embedding generation patterns, capacity planning, and performance tuning.

> **Architectural note:** These are DBA/architecture patterns. For repository wiring (sqlx connection setup, struct scanning): see the **golang-gin-database** skill. For Docker/Kubernetes setup: see the **golang-gin-deploy** skill.

## Table of Contents

1. [pgvector Overview](#pgvector-overview)
2. [Setup](#setup)
3. [Table Design](#table-design)
4. [Index Types Comparison](#index-types-comparison)
5. [Distance Functions](#distance-functions)
6. [Similarity Queries](#similarity-queries)
7. [Go Integration with pgvector-go](#go-integration-with-pgvector-go)
8. [Generating Embeddings in Go](#generating-embeddings-in-go)
9. [Capacity Planning](#capacity-planning)
10. [Performance Tips](#performance-tips)

---

## pgvector Overview

**Extension:** `pgvector` — https://github.com/pgvector/pgvector

**What it provides:**
- `vector(N)` column type storing dense float32 vectors of N dimensions
- Distance operators: `<=>` (cosine), `<->` (L2), `<#>` (inner product)
- Approximate nearest-neighbor indexes: HNSW and IVFFlat
- Exact KNN with sequential scan (no index, small datasets)

**Common use cases:**

| Use Case | Model Example | Dimensions |
|---|---|---|
| Semantic text search / RAG | OpenAI `text-embedding-ada-002` | 1536 |
| Sentence similarity | `sentence-transformers/all-MiniLM-L6-v2` | 384 |
| General purpose | `text-embedding-3-small` | 1536 |
| General purpose (large) | `text-embedding-3-large` | 3072 |
| Image embeddings | CLIP ViT-B/32 | 512 |
| Multilingual | `paraphrase-multilingual-mpnet-base-v2` | 768 |

**Key principle:** Every vector in a column must have the same number of dimensions. Mixing dimension counts in one column is a type error.

---

## Setup

### Docker — quickest start

Use the official `pgvector/pgvector` image (pgvector pre-installed):

```yaml
# docker-compose.yml
services:
  db:
    image: pgvector/pgvector:pg16   # or pg15, pg17
    environment:
      POSTGRES_DB:       myapp
      POSTGRES_USER:     myapp
      POSTGRES_PASSWORD: secret
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./migrations:/docker-entrypoint-initdb.d  # runs *.sql on first start

volumes:
  pgdata:
```

### Install on existing PostgreSQL image

```dockerfile
# Dockerfile (custom image extending standard postgres)
FROM postgres:16-bookworm

RUN apt-get update && apt-get install -y \
    postgresql-16-pgvector \
    && rm -rf /var/lib/apt/lists/*
```

### Enable the extension

Run once per database (include in your first migration):

```sql
-- 0001_enable_pgvector.up.sql
CREATE EXTENSION IF NOT EXISTS vector;
```

Cross-reference: for migration tooling (golang-migrate), see [golang-gin-database/references/migrations.md](../../golang-gin-database/references/migrations.md). For full Docker setup: see [golang-gin-deploy/references/docker-compose.md](../../golang-gin-deploy/references/docker-compose.md).

---

## Table Design

### Column definition

```sql
-- embedding vector(1536) — OpenAI ada-002 / text-embedding-3-small
-- embedding vector(768)  — sentence-transformers
-- embedding vector(384)  — MiniLM
embedding vector(1536) NOT NULL
```

### When to normalize vectors

| Distance metric | Normalization needed? | Notes |
|---|---|---|
| Cosine (`<=>`) | No — operator normalizes implicitly | Most common for text |
| Inner product (`<#>`) | **Yes** — vectors must be pre-normalized | Fastest when normalized |
| L2 (`<->`) | No | Use for spatial/geometric similarity |

To store pre-normalized vectors (enables inner-product optimization):

```sql
-- Enforce unit vectors at the database level
ALTER TABLE documents
    ADD CONSTRAINT embedding_is_unit_vector
    CHECK (abs(1 - vector_norm(embedding)) < 1e-5);
```

### Complete DDL — documents table

```sql
-- migrations/0002_create_documents.up.sql
CREATE TABLE documents (
    id           UUID        DEFAULT gen_random_uuid() PRIMARY KEY,
    tenant_id    UUID        NOT NULL,
    source_url   TEXT,
    title        TEXT        NOT NULL,
    content      TEXT        NOT NULL,
    metadata     JSONB       NOT NULL DEFAULT '{}',
    embedding    vector(1536) NOT NULL,
    model        TEXT        NOT NULL DEFAULT 'text-embedding-3-small',
    token_count  INT,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Metadata index for filtered searches
CREATE INDEX idx_documents_tenant ON documents (tenant_id);
CREATE INDEX idx_documents_metadata ON documents USING gin (metadata jsonb_path_ops);

-- Vector index (HNSW for production)
CREATE INDEX idx_documents_embedding ON documents
    USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);
```

**Design rules:**
- Store the model name alongside the embedding — you will need to re-embed when switching models
- Store `token_count` if you need cost tracking or chunking logic
- Keep raw `content` in the same table; avoids joins on retrieval
- Use `JSONB metadata` for arbitrary filters (tags, categories, language) — gin index enables fast containment queries

---

## Index Types Comparison

### IVFFlat

Divides the vector space into `lists` clusters (Voronoi cells). At query time, searches `probes` nearest clusters.

```sql
CREATE INDEX idx_documents_embedding_ivf ON documents
    USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);

-- Rule of thumb: lists = sqrt(row_count)
-- For 1M rows: lists = 1000
```

**IVFFlat requires data before indexing.** Build the index after loading at least `lists * 16` rows.

### HNSW

Hierarchical Navigable Small World graph. Builds a multi-layer proximity graph at insert time.

```sql
CREATE INDEX idx_documents_embedding_hnsw ON documents
    USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);

-- m: connections per node (higher = better recall, more memory)
-- ef_construction: search depth during build (higher = better quality, slower build)
```

### Comparison table

| Attribute | IVFFlat | HNSW |
|---|---|---|
| Build time | Fast | Slow (2-5x IVFFlat) |
| Query time | Moderate | Fast |
| Recall quality | Good (80-95%) | Excellent (95-99%) |
| Memory usage | Low | High (~2-3x vector data) |
| Requires pre-loaded data | Yes | No |
| Tuning parameter | `probes` (query) | `ef_search` (query) |
| Best for | Batch-built, large datasets | Production, incremental inserts |
| pgvector version | Original | Added in 0.5.0 |

### When to use each

- **HNSW**: default choice for production. Handles incremental inserts without rebuilding. Better recall.
- **IVFFlat**: use when you have a fixed large dataset, tight memory budget, and can tolerate a batch rebuild.
- **No index (exact KNN)**: acceptable under ~100K rows or when recall must be 100%.

---

## Distance Functions

### Operators

| Operator | Distance Type | SQL Example |
|---|---|---|
| `<=>` | Cosine distance | `embedding <=> $1` |
| `<->` | L2 (Euclidean) distance | `embedding <-> $1` |
| `<#>` | Negative inner product | `embedding <#> $1` |

Operators return **distance** (lower = more similar). For similarity score: `1 - (embedding <=> $1)`.

### Operator classes for indexes

| Operator class | Distance | Use with |
|---|---|---|
| `vector_cosine_ops` | Cosine | `<=>` |
| `vector_l2_ops` | L2 | `<->` |
| `vector_ip_ops` | Inner product | `<#>` |

The index operator class **must match** the query operator. A cosine-ops index is not used by an L2-distance query.

### Decision table

| Embedding type | Recommended operator | Reason |
|---|---|---|
| Text (OpenAI, sentence-transformers) | `<=>` cosine | Magnitude varies; direction encodes meaning |
| Pre-normalized vectors | `<#>` inner product | Equivalent to cosine but ~15% faster |
| Image / spatial | `<->` L2 | Magnitude carries information |
| Binary vectors | `<->` L2 or Hamming | Bit-level distance |

---

## Similarity Queries

### K-nearest neighbors (KNN)

```sql
SELECT id, title, 1 - (embedding <=> $1) AS similarity
FROM documents
ORDER BY embedding <=> $1
LIMIT $2;
```

### Filtered similarity — WHERE + ORDER BY

PostgreSQL runs the filter first when a selective WHERE clause is present:

```sql
-- Tenant-scoped search (most common pattern in multi-tenant apps)
SELECT id, title, content, 1 - (embedding <=> $1) AS similarity
FROM documents
WHERE tenant_id = $2
ORDER BY embedding <=> $1
LIMIT 10;
```

**Important:** If the WHERE clause is highly selective, the planner may choose a sequential scan over the vector index. This is often correct. Use `EXPLAIN ANALYZE` to verify.

### Partial index for filtered search

When you always filter by the same condition, a partial index reduces index size and improves performance:

```sql
-- Partial HNSW index — only indexes active documents
CREATE INDEX idx_docs_active_embedding ON documents
    USING hnsw (embedding vector_cosine_ops)
    WHERE deleted_at IS NULL;
```

### Similarity threshold

```sql
-- Only return results above a similarity threshold (e.g., 0.75)
SELECT id, title, 1 - (embedding <=> $1) AS similarity
FROM documents
WHERE tenant_id = $2
  AND 1 - (embedding <=> $1) > 0.75
ORDER BY embedding <=> $1
LIMIT 20;
```

### Pagination with vector search

Standard OFFSET pagination works but is not efficient for deep pages. For most RAG use cases, you only need the top-K results — pagination is rare.

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

### pgvector.Vector type

`pgvector.Vector` implements `driver.Valuer` and `sql.Scanner` — it serializes/deserializes the PostgreSQL `vector` wire format automatically with `sqlx`.

```go
import "github.com/pgvector/pgvector-go"

// Construct from float32 slice
v := pgvector.NewVector([]float32{0.1, 0.2, 0.3, /* ... 1536 values */})

// Access underlying floats
floats := v.Slice() // []float32
```

### Row struct with embedding

```go
// internal/repository/document_row.go
package repository

import (
    "time"

    "github.com/pgvector/pgvector-go"
    "myapp/internal/domain"
)

type documentRow struct {
    ID         string           `db:"id"`
    TenantID   string           `db:"tenant_id"`
    Title      string           `db:"title"`
    Content    string           `db:"content"`
    Metadata   []byte           `db:"metadata"` // JSONB → scan as []byte, unmarshal manually
    Embedding  pgvector.Vector  `db:"embedding"`
    Model      string           `db:"model"`
    TokenCount *int             `db:"token_count"`
    CreatedAt  time.Time        `db:"created_at"`
    UpdatedAt  time.Time        `db:"updated_at"`
}
```

### Complete repository: store, search, delete

```go
// internal/repository/document_repository.go
package repository

import (
    "context"
    "database/sql"
    "errors"
    "fmt"

    "github.com/jmoiron/sqlx"
    "github.com/pgvector/pgvector-go"
    "myapp/internal/domain"
)

const (
    insertDocumentSQL = `
        INSERT INTO documents (id, tenant_id, title, content, metadata, embedding, model, token_count)
        VALUES (:id, :tenant_id, :title, :content, :metadata, :embedding, :model, :token_count)
    `
    searchDocumentsSQL = `
        SELECT id, title, content, metadata, 1 - (embedding <=> $1) AS similarity
        FROM documents
        WHERE tenant_id = $2
        ORDER BY embedding <=> $1
        LIMIT $3
    `
    deleteDocumentSQL = `
        DELETE FROM documents WHERE id = $1 AND tenant_id = $2
    `
)

type documentRepository struct {
    db *sqlx.DB
}

func NewDocumentRepository(db *sqlx.DB) domain.DocumentRepository {
    return &documentRepository{db: db}
}

// Store inserts a document with its embedding.
func (r *documentRepository) Store(ctx context.Context, doc *domain.Document) error {
    row := documentInsertRow{
        ID:         doc.ID,
        TenantID:   doc.TenantID,
        Title:      doc.Title,
        Content:    doc.Content,
        Metadata:   doc.MetadataJSON, // pre-marshaled []byte
        Embedding:  pgvector.NewVector(doc.Embedding),
        Model:      doc.Model,
        TokenCount: doc.TokenCount,
    }
    if _, err := r.db.NamedExecContext(ctx, insertDocumentSQL, row); err != nil {
        return fmt.Errorf("document.Store: %w", err)
    }
    return nil
}

// Search returns top-K documents ordered by cosine similarity.
func (r *documentRepository) Search(
    ctx context.Context,
    tenantID string,
    queryEmbedding []float32,
    topK int,
) ([]domain.SearchResult, error) {
    qv := pgvector.NewVector(queryEmbedding)

    type resultRow struct {
        ID         string  `db:"id"`
        Title      string  `db:"title"`
        Content    string  `db:"content"`
        Metadata   []byte  `db:"metadata"`
        Similarity float64 `db:"similarity"`
    }

    var rows []resultRow
    if err := r.db.SelectContext(ctx, &rows, searchDocumentsSQL, qv, tenantID, topK); err != nil {
        return nil, fmt.Errorf("document.Search: %w", err)
    }

    results := make([]domain.SearchResult, len(rows))
    for i, row := range rows {
        results[i] = domain.SearchResult{
            ID:         row.ID,
            Title:      row.Title,
            Content:    row.Content,
            Similarity: row.Similarity,
        }
    }
    return results, nil
}

// Delete removes a document by ID, scoped to tenant.
func (r *documentRepository) Delete(ctx context.Context, id, tenantID string) error {
    result, err := r.db.ExecContext(ctx, deleteDocumentSQL, id, tenantID)
    if err != nil {
        return fmt.Errorf("document.Delete: %w", err)
    }
    n, err := result.RowsAffected()
    if err != nil {
        return fmt.Errorf("document.Delete rows: %w", err)
    }
    if n == 0 {
        return domain.ErrNotFound
    }
    return nil
}
```

---

## Generating Embeddings in Go

### OpenAI embeddings

```go
// internal/embeddings/openai_embedder.go
package embeddings

import (
    "bytes"
    "context"
    "encoding/json"
    "fmt"
    "log/slog"
    "net/http"
    "time"
)

const openAIEmbedURL = "https://api.openai.com/v1/embeddings"

type OpenAIEmbedder struct {
    apiKey     string
    model      string
    httpClient *http.Client
    logger     *slog.Logger
}

func NewOpenAIEmbedder(apiKey, model string, logger *slog.Logger) *OpenAIEmbedder {
    return &OpenAIEmbedder{
        apiKey: apiKey,
        model:  model,
        httpClient: &http.Client{Timeout: 30 * time.Second},
        logger: logger,
    }
}

type embedRequest struct {
    Input []string `json:"input"`
    Model string   `json:"model"`
}

type embedResponse struct {
    Data []struct {
        Embedding []float32 `json:"embedding"`
        Index     int       `json:"index"`
    } `json:"data"`
}

// Embed returns embeddings for a batch of texts.
// Retries once on transient HTTP 5xx errors.
func (e *OpenAIEmbedder) Embed(ctx context.Context, texts []string) ([][]float32, error) {
    body, err := json.Marshal(embedRequest{Input: texts, Model: e.model})
    if err != nil {
        return nil, fmt.Errorf("embed marshal: %w", err)
    }

    req, err := http.NewRequestWithContext(ctx, http.MethodPost, openAIEmbedURL, bytes.NewReader(body))
    if err != nil {
        return nil, fmt.Errorf("embed request: %w", err)
    }
    req.Header.Set("Authorization", "Bearer "+e.apiKey)
    req.Header.Set("Content-Type", "application/json")

    resp, err := e.httpClient.Do(req)
    if err != nil {
        return nil, fmt.Errorf("embed http: %w", err)
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusOK {
        return nil, fmt.Errorf("embed status %d", resp.StatusCode)
    }

    var result embedResponse
    if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
        return nil, fmt.Errorf("embed decode: %w", err)
    }

    embeddings := make([][]float32, len(result.Data))
    for _, d := range result.Data {
        embeddings[d.Index] = d.Embedding
    }

    e.logger.InfoContext(ctx, "embeddings generated", "count", len(texts), "model", e.model)
    return embeddings, nil
}
```

### Batch processing pattern

```go
// Embed documents in batches of batchSize to respect API rate limits.
func embedDocumentsBatch(
    ctx context.Context,
    embedder *embeddings.OpenAIEmbedder,
    texts []string,
    batchSize int,
    logger *slog.Logger,
) ([][]float32, error) {
    var all [][]float32

    for i := 0; i < len(texts); i += batchSize {
        end := i + batchSize
        if end > len(texts) {
            end = len(texts)
        }
        batch := texts[i:end]

        vecs, err := embedder.Embed(ctx, batch)
        if err != nil {
            return nil, fmt.Errorf("batch %d-%d: %w", i, end, err)
        }
        all = append(all, vecs...)

        logger.InfoContext(ctx, "embedding batch complete",
            "processed", end, "total", len(texts))
    }

    return all, nil
}
```

**OpenAI batch limit:** 2048 inputs per request for `text-embedding-3-small`. Use `batchSize = 100` as a conservative default.

---

## Capacity Planning

### Storage estimates

```
vector storage = dimensions × 4 bytes per vector
```

| Rows | Dimensions | Vector data | HNSW index (~2.5×) | Total |
|---|---|---|---|---|
| 100K | 1536 | ~590 MB | ~1.5 GB | ~2.1 GB |
| 1M | 1536 | ~5.9 GB | ~14.8 GB | ~20.7 GB |
| 10M | 1536 | ~59 GB | ~148 GB | ~207 GB |
| 1M | 384 | ~1.5 GB | ~3.7 GB | ~5.2 GB |

**Row overhead:** add ~100 bytes per row for heap tuple header, NULL bitmap, and other metadata.

### HNSW memory requirements

HNSW builds the entire graph in memory during index creation. Ensure PostgreSQL `maintenance_work_mem` is large enough:

```sql
-- Session-level before CREATE INDEX
SET maintenance_work_mem = '4GB';
CREATE INDEX idx_documents_embedding ON documents
    USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);
```

Rule of thumb: allocate `~1.5 × estimated index size` for `maintenance_work_mem` during build.

### Exact vs approximate search

| Dataset size | Recommendation |
|---|---|
| < 50K rows | Exact KNN (no index) — fast enough, 100% recall |
| 50K – 500K rows | HNSW with default params |
| > 500K rows | HNSW, tune `m` and `ef_search`; consider IVFFlat for memory-constrained environments |
| > 50M rows | Consider partitioning vectors by tenant/category + per-partition indexes |

---

## Performance Tips

### Query-time recall tuning

```sql
-- HNSW: increase ef_search for higher recall (default = 40)
SET hnsw.ef_search = 100;

-- IVFFlat: increase probes for higher recall (default = 1)
SET ivfflat.probes = 10;

-- These are session-local — set per request when needed
```

In Go:

```go
// Set per-connection before executing vector query
func (r *documentRepository) SearchHighRecall(
    ctx context.Context,
    tenantID string,
    queryEmbedding []float32,
    topK int,
) ([]domain.SearchResult, error) {
    tx, err := r.db.BeginTxx(ctx, nil)
    if err != nil {
        return nil, fmt.Errorf("begin: %w", err)
    }
    defer tx.Rollback()

    if _, err := tx.ExecContext(ctx, "SET LOCAL hnsw.ef_search = 100"); err != nil {
        return nil, fmt.Errorf("set ef_search: %w", err)
    }

    // ... run query on tx ...
    return results, tx.Commit()
}
```

### Filter before vector search

When your WHERE clause is highly selective (e.g., `tenant_id` narrows to < 1% of rows), PostgreSQL often does a seq scan on the filtered subset — this is faster than an index scan on all vectors. Trust the planner; verify with `EXPLAIN ANALYZE`.

```sql
-- Good: selective filter + vector distance
SELECT id, title, 1 - (embedding <=> $1) AS similarity
FROM documents
WHERE tenant_id = $2          -- highly selective
  AND model = 'text-embedding-3-small'
ORDER BY embedding <=> $1
LIMIT 10;
```

### Batch inserts for bulk loading

Use `COPY` or multi-row `INSERT` for bulk loads; single-row inserts are slow for large datasets.

```go
// Bulk insert using pgx CopyFrom (fastest for large batches)
// Or use multi-row VALUES with sqlx:
func (r *documentRepository) BulkStore(
    ctx context.Context,
    docs []domain.Document,
) error {
    // Build multi-row INSERT — batches of 500
    const batchSize = 500
    for i := 0; i < len(docs); i += batchSize {
        end := i + batchSize
        if end > len(docs) {
            end = len(docs)
        }
        if err := r.insertBatch(ctx, docs[i:end]); err != nil {
            return fmt.Errorf("bulk store batch %d: %w", i/batchSize, err)
        }
    }
    return nil
}
```

### VACUUM after large inserts

HNSW index quality can degrade after large inserts or deletes. Run VACUUM to clean up:

```sql
VACUUM ANALYZE documents;

-- For index-only: reindex without locking (PostgreSQL 12+)
REINDEX INDEX CONCURRENTLY idx_documents_embedding;
```

### Summary: performance checklist

| Setting / Pattern | Default | Recommended |
|---|---|---|
| `hnsw.ef_search` | 40 | 80–200 for high-recall RAG |
| `ivfflat.probes` | 1 | 5–20 depending on `lists` |
| `maintenance_work_mem` | 64MB | 2–8GB during index build |
| `shared_buffers` | 128MB | 25% of RAM |
| Index type | — | HNSW for production |
| Batch insert size | 1 | 100–500 rows per statement |
| Filter placement | — | WHERE before ORDER BY vector |
| Post-bulk VACUUM | Manual | Always run after large loads |

---

*Cross-references:*
- *Extension setup and Docker: [golang-gin-deploy/references/docker-compose.md](../../golang-gin-deploy/references/docker-compose.md)*
- *Migration tooling: [golang-gin-database/references/migrations.md](../../golang-gin-database/references/migrations.md)*
- *sqlx patterns: [golang-gin-database/references/sqlx-patterns.md](../../golang-gin-database/references/sqlx-patterns.md)*
- *Hybrid search with BM25: [golang-gin-psql-dba/references/paradedb-full-text-search.md](paradedb-full-text-search.md)*
