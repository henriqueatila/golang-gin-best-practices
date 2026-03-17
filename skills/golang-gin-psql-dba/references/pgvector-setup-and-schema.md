# pgvector — Setup, Schema Design, and Index Types

pgvector is an open-source PostgreSQL extension that adds a native `vector` column type and similarity-search operators, turning a standard PostgreSQL database into a vector store. Powers semantic search, RAG, recommendation engines, and image similarity.

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
| Image embeddings | CLIP ViT-B/32 | 512 |
| Multilingual | `paraphrase-multilingual-mpnet-base-v2` | 768 |

**Key principle:** Every vector in a column must have the same number of dimensions.

---

## Setup

### Docker — quickest start

Use image `pgvector/pgvector:pg16` — pgvector is pre-installed.

### Install on existing PostgreSQL image

```dockerfile
FROM postgres:16-bookworm
RUN apt-get update && apt-get install -y postgresql-16-pgvector && rm -rf /var/lib/apt/lists/*
```

### Enable the extension

```sql
-- 0001_enable_pgvector.up.sql
CREATE EXTENSION IF NOT EXISTS vector;
```

---

## Table Design

### When to normalize vectors

| Distance metric | Normalization needed? |
|---|---|
| Cosine (`<=>`) | No — operator normalizes implicitly |
| Inner product (`<#>`) | **Yes** — vectors must be pre-normalized |
| L2 (`<->`) | No |

### Complete DDL — documents table

```sql
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

CREATE INDEX idx_documents_tenant   ON documents (tenant_id);
CREATE INDEX idx_documents_metadata ON documents USING gin (metadata jsonb_path_ops);

-- Vector index (HNSW for production)
CREATE INDEX idx_documents_embedding ON documents
    USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);
```

**Design rules:**
- Store the model name alongside the embedding — re-embed when switching models
- Store `token_count` for cost tracking or chunking logic
- Keep raw `content` in the same table; avoids joins on retrieval
- Use `JSONB metadata` with gin index for fast containment queries

---

## Index Types Comparison

### IVFFlat

```sql
CREATE INDEX idx_documents_embedding_ivf ON documents
    USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);
-- Rule of thumb: lists = sqrt(row_count). For 1M rows: lists = 1000
```

**IVFFlat requires data before indexing.** Build after loading at least `lists * 16` rows.

### HNSW

```sql
CREATE INDEX idx_documents_embedding_hnsw ON documents
    USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);
-- m: connections per node (higher = better recall, more memory)
-- ef_construction: search depth during build (higher = better quality, slower)
```

### Comparison Table

| Attribute | IVFFlat | HNSW |
|---|---|---|
| Build time | Fast | Slow (2-5x IVFFlat) |
| Query time | Moderate | Fast |
| Recall quality | Good (80-95%) | Excellent (95-99%) |
| Memory usage | Low | High (~2-3x vector data) |
| Requires pre-loaded data | Yes | No |
| Tuning parameter | `probes` (query) | `ef_search` (query) |
| Best for | Batch-built, large datasets | Production, incremental inserts |

**HNSW**: default for production (incremental inserts). **IVFFlat**: fixed large dataset, tight memory. **No index**: exact KNN under ~100K rows.

---

## Distance Functions

| Operator | Distance Type | SQL Example |
|---|---|---|
| `<=>` | Cosine distance | `embedding <=> $1` |
| `<->` | L2 (Euclidean) distance | `embedding <-> $1` |
| `<#>` | Negative inner product | `embedding <#> $1` |

Operators return **distance** (lower = more similar). For similarity score: `1 - (embedding <=> $1)`.

| Embedding type | Recommended operator | Reason |
|---|---|---|
| Text (OpenAI, sentence-transformers) | `<=>` cosine | Magnitude varies; direction encodes meaning |
| Pre-normalized vectors | `<#>` inner product | Equivalent to cosine but ~15% faster |
| Image / spatial | `<->` L2 | Magnitude carries information |

The index operator class **must match** the query operator.
