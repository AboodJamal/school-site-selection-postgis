# School Site Selection - Complete SQL Solution

## Overview
This document contains the complete SQL solution for automating the school site selection process using PostGIS spatial analysis.

---

## Database Setup

### STEP 0: Enable PostGIS Extensions
```sql
CREATE EXTENSION IF NOT EXISTS postgis;
CREATE EXTENSION IF NOT EXISTS postgis_topology;
CREATE SCHEMA IF NOT EXISTS school_site;
```

---

## Data Verification

### STEP 1: Verify Data Structure

#### 1.1 Check All Tables Exist
```sql
SELECT table_name 
FROM information_schema.tables 
WHERE table_schema = 'school_site' AND table_type = 'BASE TABLE';
```

**Expected Result:**
| table_name |
|------------|
| school_site.landuse |
| school_site.buildings |
| school_site.roads |
| school_site.cistern |
| school_site.sewage |

#### 1.2 Check Row Counts
```sql
SELECT 'landuse' as table_name, COUNT(*) as row_count FROM school_site."school_site.landuse"
UNION ALL
SELECT 'buildings', COUNT(*) FROM school_site."school_site.buildings"
UNION ALL
SELECT 'roads', COUNT(*) FROM school_site."school_site.roads"
UNION ALL
SELECT 'cistern', COUNT(*) FROM school_site."school_site.cistern"
UNION ALL
SELECT 'sewage', COUNT(*) FROM school_site."school_site.sewage";
```

**Expected Result:**
| table_name | row_count |
|------------|-----------|
| landuse | 277 |
| buildings | 251 |
| roads | 13 |
| cistern | 64 |
| sewage | 68 |

#### 1.3 Check Geometry Types and SRID
```sql
SELECT 'landuse' as table_name, GeometryType(geom) as geom_type, ST_SRID(geom) as srid
FROM school_site."school_site.landuse" LIMIT 1;

SELECT 'buildings' as table_name, GeometryType(geom) as geom_type, ST_SRID(geom) as srid
FROM school_site."school_site.buildings" LIMIT 1;

SELECT 'roads' as table_name, GeometryType(geom) as geom_type, ST_SRID(geom) as srid
FROM school_site."school_site.roads" LIMIT 1;
```

**Expected Results:**
| table_name | geom_type | srid |
|------------|-----------|------|
| landuse | MULTIPOLYGON | 0 |
| buildings | MULTIPOLYGON | 0 |
| roads | MULTIPOLYGON | 0 |

**Note:** SRID is 0 but coordinates are already in projected meters (Palestine Grid), so no transformation is needed.

#### 1.4 Verify Coordinates Are in Meters
```sql
SELECT ST_XMin(geom) as x_min, ST_YMin(geom) as y_min 
FROM school_site."school_site.landuse" LIMIT 1;
```

**Expected:** Values like `153699, 114097` (meters, not degrees)

#### 1.5 Check Land Use TYPE Values
```sql
SELECT "TYPE", COUNT(*) as count 
FROM school_site."school_site.landuse" 
GROUP BY "TYPE" 
ORDER BY "TYPE";
```

**Expected Result:**
| TYPE | count |
|------|-------|
| Agricultural Areas | 26 |
| Building "Sakan" | 122 |
| Building with Agricultral Area | 14 |
| Commercial Lands | 36 |
| School | 1 |
| Un-Used | 78 |

#### 1.6 Check Buildings TYPE Values
```sql
SELECT "TYPE", COUNT(*) as count 
FROM school_site."school_site.buildings" 
GROUP BY "TYPE" 
ORDER BY "TYPE";
```

**Expected Result:**
| TYPE | count |
|------|-------|
| Agricultural | 13 |
| Commercial | 109 |
| Residential | 128 |
| School | 1 |

#### 1.7 Check Roads TYPE Values
```sql
SELECT "TYPE", COUNT(*) as count 
FROM school_site."school_site.roads" 
GROUP BY "TYPE" 
ORDER BY "TYPE";
```

**Expected Result:**
| TYPE | count |
|------|-------|
| Karkar | 2 |
| Local | 6 |
| Main | 4 |
| Turabi | 1 |

#### 1.8 Check Area Statistics
```sql
SELECT 
    ROUND(MIN(ST_Area(geom))::numeric, 2) as min_area,
    ROUND(MAX(ST_Area(geom))::numeric, 2) as max_area,
    ROUND(AVG(ST_Area(geom))::numeric, 2) as avg_area
FROM school_site."school_site.landuse";
```

**Expected Result:**
| min_area | max_area | avg_area |
|----------|----------|----------|
| 346.28 | 13437.04 | 2299.75 |

---

## Performance Optimization

### STEP 2: Create Spatial Indexes

```sql
CREATE INDEX IF NOT EXISTS idx_landuse_geom 
    ON school_site."school_site.landuse" USING GIST(geom);

CREATE INDEX IF NOT EXISTS idx_buildings_geom 
    ON school_site."school_site.buildings" USING GIST(geom);

CREATE INDEX IF NOT EXISTS idx_roads_geom 
    ON school_site."school_site.roads" USING GIST(geom);
```

---

## VIEW 1: Filter by Land Use Type

### Criteria:
- TYPE must be 'Un-Used', 'Agricultural Areas', or 'Commercial Lands'

**Expected Result:** 140 parcels (26 + 36 + 78)

```sql
CREATE OR REPLACE VIEW school_site.suitable_landuse_types AS
SELECT 
    id,
    "TYPE",
    "AREA",
    ST_Area(geom) AS area_sqm,
    geom
FROM school_site."school_site.landuse"
WHERE "TYPE" IN ('Un-Used', 'Agricultural Areas', 'Commercial Lands');
```

### Test Queries:

```sql
-- Count suitable land use types
SELECT COUNT(*) AS count_suitable_types 
FROM school_site.suitable_landuse_types;
-- Expected: 140

-- Summary by type
SELECT "TYPE", COUNT(*) as count, ROUND(SUM(area_sqm)::numeric, 2) AS total_area_sqm
FROM school_site.suitable_landuse_types
GROUP BY "TYPE" 
ORDER BY "TYPE";
```

**Expected Result:**
| TYPE | count | total_area_sqm |
|------|-------|----------------|
| Agricultural Areas | 26 | (varies) |
| Commercial Lands | 36 | (varies) |
| Un-Used | 78 | (varies) |

---

## VIEW 2: Filter by Area (≥ 5,000 m²)

### Criteria:
- Parcel area must be ≥ 5,000 square meters

**Expected Result:** 23 parcels

```sql
CREATE OR REPLACE VIEW school_site.suitable_area_parcels AS
SELECT 
    id,
    "TYPE",
    "AREA",
    ST_Area(geom) AS area_sqm,
    geom
FROM school_site.suitable_landuse_types
WHERE ST_Area(geom) >= 5000;
```

### Test Queries:

```sql
-- Count parcels with sufficient area
SELECT COUNT(*) AS count_suitable_area 
FROM school_site.suitable_area_parcels;
-- Expected: 23

-- Area statistics
SELECT 
    COUNT(*) AS total_parcels,
    ROUND(MIN(area_sqm)::numeric, 2) AS min_area,
    ROUND(MAX(area_sqm)::numeric, 2) AS max_area,
    ROUND(AVG(area_sqm)::numeric, 2) AS avg_area
FROM school_site.suitable_area_parcels;

-- Distribution by type
SELECT "TYPE", COUNT(*) as count
FROM school_site.suitable_area_parcels
GROUP BY "TYPE" ORDER BY count DESC;
```

---

## VIEW 3: Exclude Parcels with Buildings

### Criteria:
- Selected parcels must have NO existing buildings on them

**Expected Result:** 17 parcels

```sql
CREATE OR REPLACE VIEW school_site.parcels_without_buildings AS
SELECT DISTINCT
    l.id,
    l."TYPE",
    l."AREA",
    l.area_sqm,
    l.geom
FROM school_site.suitable_area_parcels AS l
WHERE NOT EXISTS (
    SELECT 1
    FROM school_site."school_site.buildings" AS b
    WHERE ST_Intersects(l.geom, b.geom)
);
```

### Explanation:
- `ST_Intersects()`: Returns TRUE if building touches or overlaps with parcel
- `NOT EXISTS`: Keeps only parcels with NO building intersection
- This ensures selected parcels are completely free of buildings

### Test Queries:

```sql
-- Count parcels without buildings
SELECT COUNT(*) AS parcels_without_buildings 
FROM school_site.parcels_without_buildings;
-- Expected: 17

-- Summary
SELECT 
    COUNT(*) AS parcels_without_buildings,
    ROUND(SUM(area_sqm)::numeric, 2) AS total_area_sqm
FROM school_site.parcels_without_buildings;

-- Distribution by type
SELECT "TYPE", COUNT(*) as count
FROM school_site.parcels_without_buildings
GROUP BY "TYPE" ORDER BY count DESC;
```

---

## VIEW 4: Filter by Road Proximity (≤ 25 meters)

### Criteria:
- Selected areas must be within 25 meters of the nearest road

**Expected Result:** 7 parcels

```sql
CREATE OR REPLACE VIEW school_site.final_school_sites AS
SELECT DISTINCT
    p.id,
    p."TYPE",
    p."AREA",
    p.area_sqm,
    MIN(ST_Distance(p.geom, r.geom)) AS distance_to_road_m,
    p.geom
FROM school_site.parcels_without_buildings AS p
JOIN school_site."school_site.roads" AS r
    ON ST_DWithin(p.geom, r.geom, 25)
GROUP BY p.id, p."TYPE", p."AREA", p.area_sqm, p.geom;
```

### Explanation:
- `ST_DWithin(geom1, geom2, 25)`: Returns TRUE if within 25 meters
- `JOIN ... ON ST_DWithin`: Keeps only parcels near roads
- `MIN(ST_Distance(...))`: Calculates distance to nearest road

### Test Queries:

```sql
-- Count final sites
SELECT COUNT(*) AS total_suitable_sites 
FROM school_site.final_school_sites;
-- Expected: 7

-- Area and distance statistics
SELECT 
    COUNT(*) AS total_suitable_sites,
    ROUND(MIN(area_sqm)::numeric, 2) AS min_area,
    ROUND(MAX(area_sqm)::numeric, 2) AS max_area,
    ROUND(MIN(distance_to_road_m)::numeric, 2) AS min_distance,
    ROUND(MAX(distance_to_road_m)::numeric, 2) AS max_distance
FROM school_site.final_school_sites;
```

---

## Final Results

### View All Final School Sites

```sql
SELECT 
    id,
    "TYPE" AS land_use_type,
    ROUND(area_sqm::numeric, 2) AS area_sqm,
    ROUND(distance_to_road_m::numeric, 2) AS distance_to_road_m,
    CASE WHEN area_sqm >= 5000 THEN '✓' ELSE '✗' END AS "Area≥5000",
    CASE WHEN distance_to_road_m <= 25 THEN '✓' ELSE '✗' END AS "Road≤25m"
FROM school_site.final_school_sites
ORDER BY area_sqm DESC;
```

**Expected Result (7 parcels):**
| id | land_use_type | area_sqm | distance_to_road_m | Area≥5000 | Road≤25m |
|----|---------------|----------|-------------------|-----------|----------|
| 20 | Agricultural Areas | 9935.86 | 0.00 | ✓ | ✓ |
| 131 | Un-Used | 8247.02 | 0.00 | ✓ | ✓ |
| 215 | Un-Used | 7981.13 | 0.00 | ✓ | ✓ |
| 8 | Un-Used | 7283.46 | 0.00 | ✓ | ✓ |
| 145 | Un-Used | 7214.55 | 0.00 | ✓ | ✓ |
| 11 | Un-Used | 6255.88 | 14.22 | ✓ | ✓ |
| 17 | Agricultural Areas | 5861.30 | 0.00 | ✓ | ✓ |

### Summary by Land Use Type

```sql
SELECT 
    "TYPE" AS land_use_type,
    COUNT(*) AS site_count,
    ROUND(SUM(area_sqm)::numeric, 2) AS total_area_sqm,
    ROUND(AVG(area_sqm)::numeric, 2) AS avg_area_sqm
FROM school_site.final_school_sites
GROUP BY "TYPE"
ORDER BY site_count DESC;
```

**Expected Result:**
| land_use_type | site_count | total_area_sqm | avg_area_sqm |
|---------------|------------|----------------|--------------|
| Un-Used | 5 | 36982.04 | 7396.41 |
| Agricultural Areas | 2 | 15797.16 | 7898.58 |

### Filtering Progression Summary

```sql
SELECT 'suitable_landuse_types' as view_name, COUNT(*) as count FROM school_site.suitable_landuse_types
UNION ALL
SELECT 'suitable_area_parcels', COUNT(*) FROM school_site.suitable_area_parcels
UNION ALL
SELECT 'parcels_without_buildings', COUNT(*) FROM school_site.parcels_without_buildings
UNION ALL
SELECT 'final_school_sites', COUNT(*) FROM school_site.final_school_sites;
```

**Expected Result:**
| view_name | count |
|-----------|-------|
| suitable_landuse_types | 140 |
| suitable_area_parcels | 23 |
| parcels_without_buildings | 17 |
| final_school_sites | 7 |

---

## Complete SQL Script (Copy-Paste Ready)

```sql
-- ============================================================================
-- HW2: SCHOOL SITE SELECTION - COMPLETE SOLUTION
-- Database: school_site_db
-- Schema: school_site
-- Tables: "school_site.landuse", "school_site.buildings", "school_site.roads"
-- ============================================================================

-- ===================
-- STEP 0: Setup
-- ===================
CREATE EXTENSION IF NOT EXISTS postgis;
CREATE EXTENSION IF NOT EXISTS postgis_topology;
CREATE SCHEMA IF NOT EXISTS school_site;

-- ===================
-- STEP 1: Verify Data
-- ===================
-- Check row counts
SELECT 'landuse' as table_name, COUNT(*) as row_count FROM school_site."school_site.landuse"
UNION ALL SELECT 'buildings', COUNT(*) FROM school_site."school_site.buildings"
UNION ALL SELECT 'roads', COUNT(*) FROM school_site."school_site.roads";

-- Check land use types
SELECT "TYPE", COUNT(*) FROM school_site."school_site.landuse" GROUP BY "TYPE" ORDER BY "TYPE";

-- Check geometry info
SELECT 'landuse' as tbl, GeometryType(geom) as geom_type, ST_SRID(geom) as srid
FROM school_site."school_site.landuse" LIMIT 1;

-- Verify coordinates are in meters
SELECT ST_XMin(geom), ST_YMin(geom) FROM school_site."school_site.landuse" LIMIT 1;

-- ===================
-- STEP 2: Create Spatial Indexes
-- ===================
CREATE INDEX IF NOT EXISTS idx_landuse_geom ON school_site."school_site.landuse" USING GIST(geom);
CREATE INDEX IF NOT EXISTS idx_buildings_geom ON school_site."school_site.buildings" USING GIST(geom);
CREATE INDEX IF NOT EXISTS idx_roads_geom ON school_site."school_site.roads" USING GIST(geom);

-- ===================
-- VIEW 1: Filter by Land Use Type
-- Criteria: TYPE IN ('Un-Used', 'Agricultural Areas', 'Commercial Lands')
-- ===================
CREATE OR REPLACE VIEW school_site.suitable_landuse_types AS
SELECT id, "TYPE", "AREA", ST_Area(geom) AS area_sqm, geom
FROM school_site."school_site.landuse"
WHERE "TYPE" IN ('Un-Used', 'Agricultural Areas', 'Commercial Lands');

-- Test VIEW 1
SELECT "TYPE", COUNT(*), ROUND(SUM(ST_Area(geom))::numeric, 2) AS total_area 
FROM school_site.suitable_landuse_types GROUP BY "TYPE";
-- Expected: 140 total (26 + 36 + 78)

-- ===================
-- VIEW 2: Filter by Area (≥ 5000 m²)
-- ===================
CREATE OR REPLACE VIEW school_site.suitable_area_parcels AS
SELECT id, "TYPE", "AREA", ST_Area(geom) AS area_sqm, geom
FROM school_site.suitable_landuse_types
WHERE ST_Area(geom) >= 5000;

-- Test VIEW 2
SELECT COUNT(*) AS count, ROUND(MIN(area_sqm)::numeric, 2) AS min_area,
       ROUND(MAX(area_sqm)::numeric, 2) AS max_area 
FROM school_site.suitable_area_parcels;
-- Expected: 23 parcels

-- ===================
-- VIEW 3: Exclude Parcels with Buildings
-- ===================
CREATE OR REPLACE VIEW school_site.parcels_without_buildings AS
SELECT DISTINCT l.id, l."TYPE", l."AREA", l.area_sqm, l.geom
FROM school_site.suitable_area_parcels AS l
WHERE NOT EXISTS (
    SELECT 1 FROM school_site."school_site.buildings" AS b
    WHERE ST_Intersects(l.geom, b.geom)
);

-- Test VIEW 3
SELECT COUNT(*) AS parcels_without_buildings,
       ROUND(SUM(area_sqm)::numeric, 2) AS total_area 
FROM school_site.parcels_without_buildings;
-- Expected: 17 parcels

-- ===================
-- VIEW 4: Filter by Road Proximity (≤ 25m)
-- ===================
CREATE OR REPLACE VIEW school_site.final_school_sites AS
SELECT DISTINCT p.id, p."TYPE", p."AREA", p.area_sqm,
       MIN(ST_Distance(p.geom, r.geom)) AS distance_to_road_m, p.geom
FROM school_site.parcels_without_buildings AS p
JOIN school_site."school_site.roads" AS r ON ST_DWithin(p.geom, r.geom, 25)
GROUP BY p.id, p."TYPE", p."AREA", p.area_sqm, p.geom;

-- Test VIEW 4
SELECT COUNT(*) AS final_sites FROM school_site.final_school_sites;
-- Expected: 7 parcels

-- ===================
-- FINAL RESULTS
-- ===================
-- All final sites with details
SELECT id, "TYPE" AS land_use_type,
       ROUND(area_sqm::numeric, 2) AS area_sqm,
       ROUND(distance_to_road_m::numeric, 2) AS distance_to_road_m,
       CASE WHEN area_sqm >= 5000 THEN '✓' ELSE '✗' END AS "Area≥5000",
       CASE WHEN distance_to_road_m <= 25 THEN '✓' ELSE '✗' END AS "Road≤25m"
FROM school_site.final_school_sites
ORDER BY area_sqm DESC;

-- Summary by land use type
SELECT "TYPE" AS land_use_type, COUNT(*) AS site_count,
       ROUND(SUM(area_sqm)::numeric, 2) AS total_area_sqm,
       ROUND(AVG(area_sqm)::numeric, 2) AS avg_area_sqm
FROM school_site.final_school_sites
GROUP BY "TYPE" ORDER BY site_count DESC;

-- Filtering progression
SELECT 'suitable_landuse_types' as step, COUNT(*) as count FROM school_site.suitable_landuse_types
UNION ALL SELECT 'suitable_area_parcels', COUNT(*) FROM school_site.suitable_area_parcels
UNION ALL SELECT 'parcels_without_buildings', COUNT(*) FROM school_site.parcels_without_buildings
UNION ALL SELECT 'final_school_sites', COUNT(*) FROM school_site.final_school_sites;
```

---

## Visualization in QGIS

### Recommended Layer Order (Bottom to Top)

| Order | Layer | Source | Style Suggestion |
|-------|-------|--------|------------------|
| 1 | All Land Parcels | `school_site.landuse` | Light gray fill, transparent |
| 2 | Roads | `school_site.roads` | Black fill |
| 3 | Buildings | `school_site.buildings` | Red fill |
| 4 | Suitable Land Types | `suitable_landuse_types` | Light yellow fill |
| 5 | Suitable Area | `suitable_area_parcels` | Orange fill |
| 6 | Without Buildings | `parcels_without_buildings` | Light green fill |
| 7 | Final Sites | `final_school_sites` | Bright green ⭐ |

---

## Summary

### Final Results: 7 Suitable School Sites

| ID | Land Use Type | Area (m²) | Distance to Road (m) |
|----|---------------|-----------|---------------------|
| 20 | Agricultural Areas | 9,935.86 | 0.00 ✓ |
| 131 | Un-Used | 8,247.02 | 0.00 ✓ |
| 215 | Un-Used | 7,981.13 | 0.00 ✓ |
| 8 | Un-Used | 7,283.46 | 0.00 ✓ |
| 145 | Un-Used | 7,214.55 | 0.00 ✓ |
| 11 | Un-Used | 6,255.88 | 14.22 ✓ |
| 17 | Agricultural Areas | 5,861.30 | 0.00 ✓ |

### Filtering Funnel

```
Land Use (277) → Suitable Types (140) → Filter by Un-Used/Agri/Commercial
            → Suitable Area (23) → Filter by ≥ 5000 m²
            → Without Buildings (17) → Exclude parcels with buildings
            → Final Sites (7) → Filter by ≤ 25m from roads
```

### Key Observations

1. **6 of 7 sites** are directly adjacent to roads (distance = 0)
2. **5 of 7 sites** are "Un-Used" land, ideal for new construction
3. **Largest site** (ID 20) is 9,935.86 m² - almost 1 hectare
4. **All sites** exceed the 5,000 m² minimum requirement

