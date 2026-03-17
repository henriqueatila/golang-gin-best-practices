# PostGIS — Go Integration and Performance Tips

## Go Integration with sqlx

PostGIS returns spatial data as WKB or WKT. The simplest approach is to read coordinates via `ST_X`/`ST_Y` rather than parsing raw WKB in Go.

### Domain Type

```go
// internal/domain/location.go
package domain

type Point struct {
    Lat float64
    Lng float64
}

type Store struct {
    ID       int64   `db:"id"`
    Name     string  `db:"name"`
    Lat      float64 `db:"lat"`
    Lng      float64 `db:"lng"`
    Distance float64 `db:"distance_metres"`
}
```

### Inserting Spatial Data

```go
// internal/repository/store_repository.go
package repository

import (
    "context"
    "fmt"
    "log/slog"

    "github.com/jmoiron/sqlx"
    "yourmodule/internal/domain"
)

type StoreRepository struct {
    db     *sqlx.DB
    logger *slog.Logger
}

func NewStoreRepository(db *sqlx.DB, logger *slog.Logger) *StoreRepository {
    return &StoreRepository{db: db, logger: logger}
}

func (r *StoreRepository) Create(ctx context.Context, name string, p domain.Point) (int64, error) {
    const q = `
        INSERT INTO stores (name, location)
        VALUES ($1, ST_MakePoint($2, $3)::geography)
        RETURNING id`

    var id int64
    // Note: ST_MakePoint(lng, lat) — longitude first
    if err := r.db.QueryRowContext(ctx, q, name, p.Lng, p.Lat).Scan(&id); err != nil {
        return 0, fmt.Errorf("StoreRepository.Create: %w", err)
    }
    r.logger.InfoContext(ctx, "store created", "id", id, "name", name)
    return id, nil
}
```

### Scanning Results — extract coordinates in SELECT

```go
func (r *StoreRepository) FindNearby(
    ctx context.Context, center domain.Point, radiusMeters float64, limit int,
) ([]domain.Store, error) {
    const q = `
        SELECT
            id, name,
            ST_Y(location::geometry)                               AS lat,
            ST_X(location::geometry)                               AS lng,
            ST_Distance(location, ST_MakePoint($2, $1)::geography) AS distance_metres
        FROM stores
        WHERE ST_DWithin(location, ST_MakePoint($2, $1)::geography, $3)
        ORDER BY distance_metres
        LIMIT $4`

    // $1=lat, $2=lng — ST_MakePoint(lng, lat)
    var stores []domain.Store
    if err := r.db.SelectContext(ctx, &stores, q,
        center.Lat, center.Lng, radiusMeters, limit,
    ); err != nil {
        return nil, fmt.Errorf("StoreRepository.FindNearby: %w", err)
    }
    return stores, nil
}
```

---

---

## Performance Tips

| Tip | Why |
|---|---|
| Use `ST_DWithin` — never `ST_Distance < X` | `ST_DWithin` is index-aware; `ST_Distance` computes for every row |
| Create GiST index before loading data | Bulk insert then index is faster than incremental updates |
| Use `CREATE INDEX CONCURRENTLY` on live tables | Avoids `AccessExclusiveLock` |
| Cast to `geometry` only for bounding-box ops | `geography` spherical math is slower |
| Cluster table by spatial index | `CLUSTER stores USING idx_stores_location` — co-locates nearby rows |
| Simplify complex polygons before indexing | `ST_Simplify(boundary, 0.0001)` reduces vertex count |
| Partition large tables by grid cell | Combine with bounding-box filters for >50M points |
| Set `work_mem` for sort-heavy spatial joins | `SET work_mem = '64MB'` per session |

**Diagnostic query — check index usage:**

```sql
SELECT schemaname, tablename, indexname, idx_scan, idx_tup_read
FROM pg_stat_user_indexes
WHERE indexname LIKE '%location%'
ORDER BY idx_scan DESC;
```

`idx_scan = 0` after normal traffic means the planner is ignoring the index — run `ANALYZE stores;` then re-check.

*Cross-skill references:*
- *Docker/Kubernetes deployment: **golang-gin-deploy** skill*
- *sqlx connection setup: **golang-gin-database** skill → [sqlx-patterns.md](../../golang-gin-database/references/sqlx-patterns.md)*
- *Testing spatial queries with testcontainers-go: **golang-gin-testing** skill*
