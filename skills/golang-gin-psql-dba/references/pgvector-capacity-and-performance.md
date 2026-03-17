# pgvector — Capacity Planning and Performance Tips

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

HNSW builds the entire graph in memory during index creation.

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

In Go — set per-connection using SET LOCAL:

```go
func (r *documentRepository) SearchHighRecall(
    ctx context.Context, tenantID string, queryEmbedding []float32, topK int,
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
SELECT id, title, 1 - (embedding <=> $1) AS similarity
FROM documents
WHERE tenant_id = $2          -- highly selective
  AND model = 'text-embedding-3-small'
ORDER BY embedding <=> $1
LIMIT 10;
```

### Batch inserts for bulk loading

```go
func (r *documentRepository) BulkStore(ctx context.Context, docs []domain.Document) error {
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

```sql
VACUUM ANALYZE documents;

-- Reindex without locking (PostgreSQL 12+)
REINDEX INDEX CONCURRENTLY idx_documents_embedding;
```

### Performance checklist

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
- *Hybrid search with BM25: [paradedb-advanced-search-and-hybrid.md](paradedb-advanced-search-and-hybrid.md)*
