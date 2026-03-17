# PostGIS — Setup, Types, and GiST Indexes

PostGIS extends PostgreSQL with ISO-standard spatial types, GiST-indexed spatial operations, and geodetic distance functions.

> **Rule: Use `geography` for global lat/lng data. Use `geometry` for local/projected data (city-level, custom SRID).**

## Docker Setup

Use image `postgis/postgis:16-3.4`. Include a healthcheck: `pg_isready -U app -d appdb`.

## Enabling PostGIS

```sql
-- migrations/000001_init.up.sql
CREATE EXTENSION IF NOT EXISTS postgis;

-- Verify
SELECT PostGIS_Full_Version();
```

---

## geometry vs geography

| Property | `geometry` | `geography` |
|---|---|---|
| Coordinate system | Flat / Euclidean (projected) | Spherical (real Earth surface) |
| Accuracy at global scale | Low — distorts with distance | High — geodetic accuracy anywhere |
| Calculation speed | Faster | ~10–20% slower |
| Default SRID | Any (must specify) | 4326 (WGS84 — GPS coordinates) |
| Distance unit | Projection units (metres, feet…) | Always **metres** |
| Index support | GiST, SP-GiST | GiST only |
| Best for | Local/city-level data, custom projections | Global lat/lng, cross-continent distances |

**SRID 4326** is the standard for GPS/WGS84. PostGIS `geography` columns always use 4326.

---

## Spatial Types

```sql
location   geography(POINT, 4326)       -- lat/lng point (global, GPS)
boundary   geometry(POLYGON, 4326)      -- area in WGS84 geometry
route      geometry(LINESTRING, 4326)   -- path/route
```

| Type | Description | Use Case |
|---|---|---|
| `POINT` | Single coordinate | Store location, user position |
| `LINESTRING` | Ordered sequence of points | Route, road segment, path |
| `POLYGON` | Closed ring of points | Area boundary, delivery zone |
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

Swapping lat/lng is the most common PostGIS mistake after choosing the wrong type.

---

## GiST Indexes for Spatial Data

PostGIS spatial queries **require** a GiST index to avoid sequential scans.

```sql
-- geography column
CREATE INDEX idx_stores_location ON stores USING gist (location);

-- geometry column
CREATE INDEX idx_zones_boundary ON zones USING gist (boundary);

-- Create concurrently on live tables
CREATE INDEX CONCURRENTLY idx_stores_location ON stores USING gist (location);
```

| Column Type | Operator Class | Notes |
|---|---|---|
| `geography` | default (gist_geography_ops) | Handles spherical calculations |
| `geometry` 2D | default (gist_geometry_ops_2d) | Bounding-box based |

**Verify index is used:**

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM stores
WHERE ST_DWithin(location, ST_MakePoint(-73.9857, 40.7484)::geography, 5000);
-- Look for: "Index Scan using idx_stores_location"
```

---

## Common Spatial Queries

### Distance — find within radius

`ST_DWithin` is index-aware. `ST_Distance < X` is **not** — always triggers a full scan.

```sql
SELECT id, name,
    ST_Distance(location, ST_MakePoint(-73.9857, 40.7484)::geography) AS distance_metres
FROM stores
WHERE ST_DWithin(location, ST_MakePoint(-73.9857, 40.7484)::geography, 5000)
ORDER BY distance_metres;
```

### K-Nearest Neighbor (KNN)

The `<->` operator enables index-driven KNN — stops early once K results are found.

```sql
SELECT id, name,
    location <-> ST_MakePoint(-73.9857, 40.7484)::geography AS distance_metres
FROM stores
ORDER BY location <-> ST_MakePoint(-73.9857, 40.7484)::geography
LIMIT 10;
```

Combine KNN + radius: add `WHERE ST_DWithin(...)` before `ORDER BY distance_metres LIMIT N`.

### Bounding Box — map viewport queries

```sql
SELECT id, name FROM stores
WHERE ST_Within(
    location::geometry,
    ST_MakeEnvelope(-74.0500, 40.6900, -73.9000, 40.8000, 4326)
);
```

### Point-in-Polygon

```sql
SELECT name FROM zones
WHERE ST_Contains(boundary::geometry, ST_MakePoint(-73.9800, 40.7500)::geometry);
```

### Line Intersection

`ST_Intersects(r1.path::geometry, r2.path::geometry)` filters route pairs, `ST_Intersection(...)` returns the intersection point. Combine with a GiST index on the geometry column.
