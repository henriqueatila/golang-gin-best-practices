# PostGIS Geospatial Reference

PostGIS extends PostgreSQL with ISO-standard spatial types, GiST-indexed spatial operations, and geodetic distance functions — making it the production choice for location-aware Gin APIs. This reference covers Docker setup, the `geometry` vs `geography` decision, spatial types, GiST indexes, the most common query patterns (distance, KNN, bounding box, point-in-polygon, line intersection), and complete Go integration with sqlx using a store-finder example. All SQL is PostgreSQL-specific; all Go uses `log/slog`, `context.Context`, and `fmt.Errorf`.

> **Rule: Use `geography` for global lat/lng data. Use `geometry` for local/projected data (city-level, custom SRID).**

## Table of Contents

1. [Docker Setup](#docker-setup)
2. [Enabling PostGIS](#enabling-postgis)
3. [geometry vs geography](#geometry-vs-geography)
4. [Spatial Types](#spatial-types)
5. [GiST Indexes for Spatial Data](#gist-indexes-for-spatial-data)
6. [Common Spatial Queries](#common-spatial-queries)
   - [Distance — find within radius](#distance--find-within-radius)
   - [K-Nearest Neighbor (KNN)](#k-nearest-neighbor-knn)
   - [Bounding Box — points within rectangle](#bounding-box--points-within-rectangle)
   - [Point-in-Polygon](#point-in-polygon)
   - [Line Intersection](#line-intersection)
7. [Go Integration with sqlx](#go-integration-with-sqlx)
8. [Complete Store Finder Example](#complete-store-finder-example)
9. [Performance Tips](#performance-tips)

---

## Docker Setup

Use the official `postgis/postgis` image — it layers PostGIS on top of the official `postgres` image.

```yaml
# docker-compose.yml
services:
  postgres:
    image: postgis/postgis:16-3.4  # PostgreSQL 16 + PostGIS 3.4
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: appdb
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d appdb"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  pgdata:
```

For production containers and Kubernetes StatefulSet patterns, cross-reference the **gin-deploy** skill.

---

## Enabling PostGIS

PostGIS must be enabled per database. Run once after the database is created — safe to include in your first migration:

```sql
-- migrations/000001_init.up.sql
CREATE EXTENSION IF NOT EXISTS postgis;
```

Verify the installation:

```sql
SELECT PostGIS_Full_Version();
-- POSTGIS="3.4.x" PGSQL="160" ...
```

---

## geometry vs geography

Choosing the wrong type is the most common PostGIS mistake. The difference is how distance and area are computed.

| Property | `geometry` | `geography` |
|---|---|---|
| Coordinate system | Flat / Euclidean (projected) | Spherical (real Earth surface) |
| Accuracy at global scale | Low — distorts with distance | High — geodetic accuracy anywhere |
| Calculation speed | Faster | ~10–20% slower |
| Default SRID | Any (must specify) | 4326 (WGS84 — GPS coordinates) |
| Distance unit | Projection units (metres, feet…) | Always **metres** |
| Index support | GiST, SP-GiST | GiST only |
| Cast between types | `geom::geography` / `geog::geometry` | Same |
| Best for | Local/city-level data, custom projections | Global lat/lng, cross-continent distances |

**Rule:** If the coordinates come from GPS devices, mobile apps, or any WGS84 source — use `geography`. Use `geometry` only when working with a specific projected coordinate system (e.g., EPSG:3857 for web tiles, or local survey data).

**SRID 4326** is the standard for GPS/WGS84. PostGIS `geography` columns always use 4326.

---

## Spatial Types

PostGIS provides standard OGC geometry/geography types. All take an optional SRID parameter.

```sql
-- Columns
location   geography(POINT, 4326)       -- lat/lng point (global, GPS)
boundary   geometry(POLYGON, 4326)      -- area in WGS84 geometry
route      geometry(LINESTRING, 4326)   -- path/route
region     geography(POLYGON, 4326)     -- area with geodetic accuracy
```

**Common types:**

| Type | Description | Use Case |
|---|---|---|
| `POINT` | Single coordinate | Store location, user position |
| `LINESTRING` | Ordered sequence of points | Route, road segment, path |
| `POLYGON` | Closed ring of points | Area boundary, delivery zone |
| `MULTIPOINT` | Collection of points | Multiple pickup locations |
| `MULTIPOLYGON` | Collection of polygons | Country with islands |

**Creating a point — longitude comes first:**

```sql
-- ST_MakePoint(longitude, latitude) — NOT (lat, lng)
ST_MakePoint(-73.9857, 40.7484)::geography   -- Times Square, NYC

-- From WKT
ST_GeographyFromText('POINT(-73.9857 40.7484)')

-- From GeoJSON
ST_GeomFromGeoJSON('{"type":"Point","coordinates":[-73.9857,40.7484]}')::geography
```

The longitude-first convention matches GeoJSON and WKT standards. Swapping lat/lng is the second most common PostGIS mistake.

---

## GiST Indexes for Spatial Data

PostGIS spatial queries **require** a GiST index to avoid sequential scans. Without it, `ST_DWithin`, `ST_Intersects`, and `<->` operator degrade to O(n).

```sql
-- geography column — use geography operator class
CREATE INDEX idx_stores_location ON stores USING gist (location);

-- geometry column — default GiST works; specify ops for nD data
CREATE INDEX idx_zones_boundary ON zones USING gist (boundary);

-- Create concurrently on live tables (avoids write lock)
CREATE INDEX CONCURRENTLY idx_stores_location ON stores USING gist (location);
```

**Index operator classes:**

| Column Type | Operator Class | Notes |
|---|---|---|
| `geography` | default (gist_geography_ops) | Handles spherical calculations |
| `geometry` 2D | default (gist_geometry_ops_2d) | Bounding-box based |
| `geometry` nD | `gist_geometry_ops_nd` | For 3D/4D data — rare |

**Verify index is used:**

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM stores
WHERE ST_DWithin(location, ST_MakePoint(-73.9857, 40.7484)::geography, 5000);
-- Look for: "Index Scan using idx_stores_location"
```

---

## Common Spatial Queries

### Distance — find within radius

`ST_DWithin` is index-aware. `ST_Distance < X` is **not** — it always triggers a full scan.

```sql
-- Find all stores within 5km of a point (radius in metres for geography)
SELECT
    id,
    name,
    ST_Distance(location, ST_MakePoint(-73.9857, 40.7484)::geography) AS distance_metres
FROM stores
WHERE ST_DWithin(
    location,
    ST_MakePoint(-73.9857, 40.7484)::geography,
    5000  -- metres
)
ORDER BY distance_metres;
```

### K-Nearest Neighbor (KNN)

The `<->` operator enables index-driven KNN — PostgreSQL uses a GiST index scan that stops early once K results are found.

```sql
-- Closest 10 stores, no radius constraint
SELECT
    id,
    name,
    location <-> ST_MakePoint(-73.9857, 40.7484)::geography AS distance_metres
FROM stores
ORDER BY location <-> ST_MakePoint(-73.9857, 40.7484)::geography
LIMIT 10;
```

**KNN vs DWithin:**
- Use `<->` + `LIMIT` when you want the nearest N results regardless of distance.
- Use `ST_DWithin` when you have a hard radius requirement.
- Combine both when you want "nearest N within X metres":

```sql
SELECT id, name,
    location <-> ST_MakePoint(-73.9857, 40.7484)::geography AS distance_metres
FROM stores
WHERE ST_DWithin(location, ST_MakePoint(-73.9857, 40.7484)::geography, 5000)
ORDER BY distance_metres
LIMIT 5;
```

### Bounding Box — points within rectangle

Useful for map viewport queries (SW corner to NE corner).

```sql
-- Points inside a bounding box defined by SW and NE corners
SELECT id, name
FROM stores
WHERE ST_Within(
    location::geometry,
    ST_MakeEnvelope(
        -74.0500, 40.6900,  -- SW: lng, lat
        -73.9000, 40.8000,  -- NE: lng, lat
        4326
    )
);
```

Note: `ST_MakeEnvelope` returns `geometry`, so cast `location` to `geometry` for the comparison. For large viewports, `ST_DWithin` with a geography centre-point is often more natural.

### Point-in-Polygon

Determine if a user's location falls inside a defined zone (delivery area, geofence, etc.).

```sql
-- Is point (-73.98, 40.75) inside the Manhattan polygon?
SELECT name
FROM zones
WHERE ST_Within(
    ST_MakePoint(-73.9800, 40.7500)::geometry,
    boundary::geometry
);

-- Alternative: ST_Contains(polygon, point) — semantically equivalent
SELECT name
FROM zones
WHERE ST_Contains(boundary::geometry, ST_MakePoint(-73.9800, 40.7500)::geometry);
```

### Line Intersection

Check if two routes or paths cross. Useful for ride-sharing, logistics, or geofence breach detection.

```sql
-- Do two delivery routes intersect?
SELECT
    r1.id AS route_a,
    r2.id AS route_b,
    ST_Intersection(r1.path::geometry, r2.path::geometry) AS intersection_point
FROM routes r1
JOIN routes r2 ON r1.id < r2.id
WHERE ST_Intersects(r1.path::geometry, r2.path::geometry);
```

---

## Go Integration with sqlx

### Dependency

```bash
go get github.com/jmoiron/sqlx
go get github.com/lib/pq   # or github.com/jackc/pgx/v5/stdlib
```

PostGIS returns spatial data as WKB (Well-Known Binary) or WKT (Well-Known Text). The simplest approach is to read coordinates back via `ST_X`/`ST_Y` or `ST_AsText` rather than parsing raw WKB in Go.

### Domain Type

```go
// internal/domain/location.go
package domain

// Point holds a WGS84 coordinate pair.
type Point struct {
    Lat float64
    Lng float64
}

// Store is the domain model for a retail location.
type Store struct {
    ID       int64   `db:"id"`
    Name     string  `db:"name"`
    Lat      float64 `db:"lat"`
    Lng      float64 `db:"lng"`
    Distance float64 `db:"distance_metres"` // populated by spatial queries
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

// Create inserts a new store with a geography POINT column.
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

### Scanning Results with Coordinates

Return coordinates by extracting them in the SELECT — avoids WKB parsing entirely:

```go
// FindNearby returns stores within radiusMeters, ordered by distance ascending.
func (r *StoreRepository) FindNearby(
    ctx context.Context,
    center domain.Point,
    radiusMeters float64,
    limit int,
) ([]domain.Store, error) {
    const q = `
        SELECT
            id,
            name,
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

## Complete Store Finder Example

### DDL

```sql
-- migrations/000002_stores.up.sql
CREATE TABLE stores (
    id         BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name       TEXT        NOT NULL,
    address    TEXT        NOT NULL DEFAULT '',
    location   geography(POINT, 4326) NOT NULL,
    active     BOOLEAN     NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- GiST index — required for spatial query performance
CREATE INDEX idx_stores_location ON stores USING gist (location);
```

### Repository — FindInBoundingBox

```go
// BoundingBox defines a SW/NE viewport rectangle.
type BoundingBox struct {
    SWLat, SWLng float64
    NELat, NELng float64
}

// FindInBoundingBox returns all active stores inside the given viewport.
func (r *StoreRepository) FindInBoundingBox(
    ctx context.Context,
    bb BoundingBox,
) ([]domain.Store, error) {
    const q = `
        SELECT
            id,
            name,
            ST_Y(location::geometry) AS lat,
            ST_X(location::geometry) AS lng,
            0.0                      AS distance_metres
        FROM stores
        WHERE active = true
          AND ST_Within(
              location::geometry,
              ST_MakeEnvelope($1, $2, $3, $4, 4326)
          )`

    // $1=SWLng $2=SWLat $3=NELng $4=NELat
    var stores []domain.Store
    if err := r.db.SelectContext(ctx, &stores, q,
        bb.SWLng, bb.SWLat, bb.NELng, bb.NELat,
    ); err != nil {
        return nil, fmt.Errorf("StoreRepository.FindInBoundingBox: %w", err)
    }
    return stores, nil
}
```

### Handler

```go
// internal/handler/store_handler.go
package handler

import (
    "log/slog"
    "net/http"

    "github.com/gin-gonic/gin"
    "yourmodule/internal/domain"
    "yourmodule/internal/repository"
)

type StoreHandler struct {
    repo   *repository.StoreRepository
    logger *slog.Logger
}

func NewStoreHandler(repo *repository.StoreRepository, logger *slog.Logger) *StoreHandler {
    return &StoreHandler{repo: repo, logger: logger}
}

type nearbyQuery struct {
    Lat    float64 `form:"lat"    binding:"required"`
    Lng    float64 `form:"lng"    binding:"required"`
    Radius float64 `form:"radius" binding:"required,min=1,max=50000"`
    Limit  int     `form:"limit"  binding:"omitempty,min=1,max=100"`
}

// GET /stores/nearby?lat=40.7484&lng=-73.9857&radius=5000&limit=10
func (h *StoreHandler) Nearby(c *gin.Context) {
    var q nearbyQuery
    if err := c.ShouldBindQuery(&q); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    if q.Limit == 0 {
        q.Limit = 20
    }

    stores, err := h.repo.FindNearby(
        c.Request.Context(),
        domain.Point{Lat: q.Lat, Lng: q.Lng},
        q.Radius,
        q.Limit,
    )
    if err != nil {
        h.logger.ErrorContext(c.Request.Context(), "FindNearby failed", "error", err)
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
        return
    }

    type storeResponse struct {
        ID       int64   `json:"id"`
        Name     string  `json:"name"`
        Lat      float64 `json:"lat"`
        Lng      float64 `json:"lng"`
        Distance float64 `json:"distance_metres"`
    }
    resp := make([]storeResponse, len(stores))
    for i, s := range stores {
        resp[i] = storeResponse{
            ID:       s.ID,
            Name:     s.Name,
            Lat:      s.Lat,
            Lng:      s.Lng,
            Distance: s.Distance,
        }
    }
    c.JSON(http.StatusOK, gin.H{"stores": resp, "count": len(resp)})
}

// Register routes — call from your router setup
func (h *StoreHandler) RegisterRoutes(rg *gin.RouterGroup) {
    rg.GET("/nearby", h.Nearby)
}
```

**Example response:**

```json
{
  "count": 3,
  "stores": [
    { "id": 42, "name": "Times Square Store", "lat": 40.7580, "lng": -73.9855, "distance_metres": 107.3 },
    { "id": 17, "name": "Bryant Park Store",  "lat": 40.7536, "lng": -73.9832, "distance_metres": 624.8 }
  ]
}
```

---

## Performance Tips

| Tip | Why |
|---|---|
| Use `ST_DWithin` — never `ST_Distance < X` | `ST_DWithin` is index-aware; `ST_Distance` computes for every row |
| Create GiST index before loading data | Bulk insert then index is faster than incremental index updates |
| Use `CREATE INDEX CONCURRENTLY` on live tables | Avoids `AccessExclusiveLock` that blocks all writes |
| Cast to `geometry` only for bounding-box ops | `geography` spherical math is slower; cast selectively when projection is needed |
| Cluster table by spatial index | `CLUSTER stores USING idx_stores_location` — co-locates nearby rows for I/O efficiency |
| Simplify complex polygons before indexing | `ST_Simplify(boundary, 0.0001)` reduces vertex count; speeds containment tests |
| Partition large tables by grid cell | Combine with PostGIS bounding-box filters for very large datasets (>50M points) |
| Set `work_mem` for sort-heavy spatial joins | `SET work_mem = '64MB'` per session when sorting large spatial result sets |

**Diagnostic query — check index usage for spatial queries:**

```sql
SELECT schemaname, tablename, indexname, idx_scan, idx_tup_read
FROM pg_stat_user_indexes
WHERE indexname LIKE '%location%'
ORDER BY idx_scan DESC;
```

A `idx_scan` count of 0 after normal traffic means the planner is ignoring the index — run `ANALYZE stores;` to refresh statistics, then re-check.

**Cross-skill references:**
- Docker/Kubernetes deployment of PostGIS containers: **gin-deploy** skill
- sqlx connection setup and repository wiring: **gin-database** skill → [sqlx-patterns.md](../../gin-database/references/sqlx-patterns.md)
- Testing spatial queries with testcontainers-go: **gin-testing** skill
