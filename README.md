# School Site Selection using PostGIS

<p align="center">
  <img src="docs/model-builder.png" alt="Model Builder Workflow" width="800"/>
</p>

## Assignment

> **Task:** Find new areas suitable for building a new school.
>
> **Selection Criteria:**
> 1. Land use types should be **unused**, **agricultural land**, or **commercial land**
> 2. Area should be **≥ 5,000 m²**
> 3. **No buildings** should be on the land use parcels
> 4. Areas should be within **25 meters** from the nearest road
>
> **Tools:** SQL in PostGIS + QGIS for visualization

---

## Project Overview

This project uses **PostGIS spatial analysis** to identify suitable locations for building a new school based on multiple spatial and attribute criteria. The solution automates the site selection process using SQL queries and spatial functions.

### Objective

Find land parcels that meet all of the following criteria:
- Appropriate land use types (Un-Used, Agricultural, Commercial)
- Sufficient area for school construction (≥ 5,000 m²)
- No existing buildings on the parcel
- Proximity to transportation infrastructure (within 25m of roads)

---

## Database Information

### Database Details
| Property | Value |
|----------|-------|
| **Database Name** | `school_site_db` (or your chosen name) |
| **Schema** | `school_site` |
| **PostGIS Version** | 3.x |
| **SRID** | 0 (already in projected meters - Palestine Grid) |
| **Units** | Meters |
| **Transformation** | Not needed (already projected) |

### Source Data Files
Located in the `data/shapefiles/` directory:

| File | Description |
|------|-------------|
| `Landuse.shp` | Land use parcels with classification |
| `Buildings.shp` | Existing building footprints |
| `Roads.shp` | Road network (as polygons) |
| `Cistern.shp` | Water infrastructure (not used in analysis) |
| `Sewage.shp` | Sewage infrastructure (not used in analysis) |

---

## Database Schema

### Table 1: `school_site."school_site.landuse"`

| Property | Value |
|----------|-------|
| **Geometry Type** | MULTIPOLYGON |
| **SRID** | 0 (projected meters) |
| **Row Count** | **277** |

#### Columns

| Column Name | Data Type | Description | Notes |
|-------------|-----------|-------------|-------|
| `id` | integer | Primary key (auto-increment) | 1, 2, 3... |
| `geom` | geometry | Parcel geometry | MULTIPOLYGON |
| `ID` | integer | Land use identifier | - |
| `CODE` | bigint | Land use code | - |
| `OWNER` | character varying | Land owner | - |
| `TYPE` | character varying | Land use type | **Key filter column** |
| `SALE` | character varying | Sale status | - |
| `NO_OF_APPA` | bigint | Number of apartments | - |
| `AREA` | double precision | Parcel area (attribute) | May differ from ST_Area() |
| `IMAGE` | character varying | Image reference | - |

#### TYPE Values (Land Use Classification)

| TYPE | Count | Used in Analysis |
|------|-------|------------------|
| Agricultural Areas | 26 | ✅ **Yes** |
| Building "Sakan" | 122 | ❌ No |
| Building with Agricultral Area | 14 | ❌ No |
| Commercial Lands | 36 | ✅ **Yes** |
| School | 1 | ❌ No |
| Un-Used | 78 | ✅ **Yes** |
| **Total** | **277** | **140 suitable** |

#### Area Statistics
| Statistic | Value (m²) |
|-----------|------------|
| Minimum | 346.28 |
| Maximum | 13,437.04 |
| Average | 2,299.75 |

---

### Table 2: `school_site."school_site.buildings"`

| Property | Value |
|----------|-------|
| **Geometry Type** | MULTIPOLYGON |
| **SRID** | 0 (projected meters) |
| **Row Count** | **251** |

#### Columns

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| `id` | integer | Primary key (auto-increment) |
| `geom` | geometry | Building footprint geometry |
| `ID` | integer | Building identifier |
| `AREA` | double precision | Building area |
| `OWNER` | character varying | Building owner |
| `TYPE` | character varying | Building type |
| `CLASS` | character varying | Building class |
| `ADDRESS` | character varying | Building address |
| `APPART` | bigint | Number of apartments |
| `APP_RENT` | bigint | Apartment rent |
| `PRICE` | bigint | Building price |
| `MALE` | bigint | Male population |
| `FEMALE` | bigint | Female population |
| `BL_CODE` | character varying | Block code |
| `BLK_CODE` | character varying | Block code (alt) |
| `PRCL_CODE` | character varying | Parcel code |
| `NOPOP` | bigint | Population count |
| `Hyper` | character varying | Hyper attribute |
| `Test` | character varying | Test attribute |

#### TYPE Values (Building Types)

| TYPE | Count |
|------|-------|
| Agricultural | 13 |
| Commercial | 109 |
| Residential | 128 |
| School | 1 |
| **Total** | **251** |

#### CLASS Values (Building Classification)

| CLASS | Count |
|-------|-------|
| Agricultural | 13 |
| Building | 79 |
| Commercial | 49 |
| House | 109 |
| School | 1 |
| **Total** | **251** |

---

### Table 3: `school_site."school_site.roads"`

| Property | Value |
|----------|-------|
| **Geometry Type** | MULTIPOLYGON |
| **SRID** | 0 (projected meters) |
| **Row Count** | **13** |

#### Columns

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| `id` | integer | Primary key (auto-increment) |
| `geom` | geometry | Road geometry (polygon) |
| `ID` | integer | Road identifier |
| `TYPE` | character varying | Road type |
| `NO_PATHS` | character varying | Number of paths |
| `AVG_WIDTH` | double precision | Average width |
| `CONDITION` | character varying | Road condition |
| `LENGTH` | double precision | Road length |
| `NAME` | character varying | Road name |
| `JAZEERAH` | character varying | Jazeerah attribute |
| `DIRECTION` | bigint | Road direction |
| `OWNER` | character varying | Road owner |

#### TYPE Values (Road Types)

| TYPE | Count |
|------|-------|
| Karkar | 2 |
| Local | 6 |
| Main | 4 |
| Turabi | 1 |
| **Total** | **13** |

#### OWNER Values

| OWNER |
|-------|
| Private |
| Public |

---

### Table 4: `school_site."school_site.cistern"` (Not Used)

| Property | Value |
|----------|-------|
| **Row Count** | **64** |
| **Purpose** | Water infrastructure (not used in main analysis) |

---

### Table 5: `school_site."school_site.sewage"` (Not Used)

| Property | Value |
|----------|-------|
| **Row Count** | **68** |
| **Purpose** | Sewage infrastructure (not used in main analysis) |

---

## Selection Criteria

### Criterion 1: Land Use Type

| Criterion | SQL Condition |
|-----------|---------------|
| Acceptable land use types | `TYPE IN ('Un-Used', 'Agricultural Areas', 'Commercial Lands')` |

**Rationale:** These land use types are suitable for school construction (no existing residential buildings).

### Criterion 2: Minimum Area

| Criterion | SQL Condition |
|-----------|---------------|
| Minimum parcel area | `ST_Area(geom) >= 5000` |

**Rationale:** A school requires at least 5,000 m² for building, playground, and parking.

### Criterion 3: No Existing Buildings

| Criterion | SQL Condition |
|-----------|---------------|
| No buildings on parcel | `NOT EXISTS (SELECT 1 FROM buildings WHERE ST_Intersects(...))` |

**Rationale:** Selected parcels must be vacant land without existing structures.

### Criterion 4: Road Proximity

| Criterion | SQL Condition |
|-----------|---------------|
| Within 25 meters of road | `ST_DWithin(parcel.geom, road.geom, 25)` |

**Rationale:** School must have good accessibility via road network.

---

## Results Summary

### Filtering Progression

| Step | View Name | Count | Criteria Applied |
|------|-----------|-------|------------------|
| 1 | `suitable_landuse_types` | **140** | TYPE IN (Un-Used, Agricultural Areas, Commercial Lands) |
| 2 | `suitable_area_parcels` | **23** | ST_Area(geom) ≥ 5,000 m² |
| 3 | `parcels_without_buildings` | **17** | NOT ST_Intersects with buildings |
| 4 | `final_school_sites` | **7** | ST_DWithin roads 25m |

### 🏆 Final School Sites (7 Parcels)

| ID | Land Use Type | Area (m²) | Distance to Road (m) |
|----|---------------|-----------|---------------------|
| 20 | Agricultural Areas | 9,935.86 | 0.00 |
| 131 | Un-Used | 8,247.02 | 0.00 |
| 215 | Un-Used | 7,981.13 | 0.00 |
| 8 | Un-Used | 7,283.46 | 0.00 |
| 145 | Un-Used | 7,214.55 | 0.00 |
| 11 | Un-Used | 6,255.88 | 14.22 |
| 17 | Agricultural Areas | 5,861.30 | 0.00 |

### Summary by Land Use Type

| Land Use Type | Count | Total Area (m²) |
|---------------|-------|-----------------|
| Un-Used | 5 | 36,982.04 |
| Agricultural Areas | 2 | 15,797.16 |
| **Total** | **7** | **52,779.20** |

### 🗺️ Spatial Output (QGIS Visualization)

<p align="center">
  <img src="visual-outputs/canvas-layers-visual.png" alt="Final School Sites - QGIS Visualization" width="800"/>
</p>

---

## Technical Implementation

### Coordinate System

| Property | Value |
|----------|-------|
| **SRID** | 0 (undefined) |
| **Actual System** | Palestine Grid (projected) |
| **Units** | Meters |
| **Sample Coordinates** | ~153,699, ~114,097 |
| **Transformation** | Not needed |

**Note:** Although SRID is 0, the coordinates are clearly in a projected coordinate system (meters), so no transformation is required.

### Spatial Functions Used

| Function | Purpose | Example |
|----------|---------|---------|
| `ST_Area()` | Calculate parcel area | `ST_Area(geom) >= 5000` |
| `ST_Intersects()` | Check building overlap | `ST_Intersects(parcel, building)` |
| `ST_DWithin()` | Distance-based filter | `ST_DWithin(parcel, road, 25)` |
| `ST_Distance()` | Calculate exact distance | `ST_Distance(parcel, road)` |

### Performance Optimization

- **GIST Spatial Indexes** on all geometry columns
- **Views** for progressive filtering (debuggable pipeline)
- **NOT EXISTS** subquery for efficient exclusion

---

## Project Structure

```
school-site-selection-postgis/
├── data/
│   └── shapefiles/
│       ├── Buildings.shp/.dbf/.shx
│       ├── Landuse.shp/.dbf/.shx
│       ├── Roads.shp/.dbf/.shx
│       ├── Cistern.shp/.dbf/.shx
│       └── Sewage.shp/.dbf/.shx
├── docs/
│   ├── assignment-question.png
│   ├── model-builder.html
│   └── model-builder.png
├── visual-outputs/              # QGIS visualization exports
├── school-site-selection.qgz   # QGIS project file
├── SOLUTION.md                  # Complete SQL solution
└── README.md                    # This file
```

---

## Getting Started

### Prerequisites
- PostgreSQL 12+ with PostGIS 3.x extension
- QGIS 3.x (for data import and visualization)
- pgAdmin 4 (optional, for database management)

### Setup Steps

1. **Create Database**
   ```sql
   CREATE DATABASE school_site_db;
   \c school_site_db
   CREATE EXTENSION postgis;
   CREATE EXTENSION postgis_topology;
   CREATE SCHEMA school_site;
   ```

2. **Import Shapefiles via QGIS**
   - Open QGIS → Database → DB Manager
   - Connect to PostgreSQL database
   - Import each shapefile to `school_site` schema
   - Leave SRID as detected (0 or auto)
   - Check "Create spatial index"

3. **Run SQL Solution**
   - Open `SOLUTION.md` in pgAdmin
   - Execute statements sequentially
   - Verify results at each step

4. **Visualize in QGIS**
   - Add views as layers
   - Style appropriately
   - Analyze spatial patterns

---

## Solution File

For the complete SQL solution with detailed comments and test queries, see **[SOLUTION.md](./SOLUTION.md)**.

---

## Workflow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        SCHOOL SITE SELECTION                                 │
└─────────────────────────────────────────────────────────────────────────────┘

    ┌──────────────┐         ┌──────────────┐         ┌──────────────┐
    │   LANDUSE    │         │  BUILDINGS   │         │    ROADS     │
    │  (277 rows)  │         │  (251 rows)  │         │  (13 rows)   │
    └──────┬───────┘         └──────┬───────┘         └──────┬───────┘
           │                        │                        │
           ▼                        │                        │
    ┌──────────────┐                │                        │
    │   WHERE:     │                │                        │
    │ • Un-Used    │                │                        │
    │ • Agri Areas │                │                        │
    │ • Commercial │                │                        │
    └──────┬───────┘                │                        │
           │                        │                        │
           ▼                        │                        │
    ┌──────────────┐                │                        │
    │   SUITABLE   │                │                        │
    │  LAND TYPES  │                │                        │
    │  (140 rows)  │                │                        │
    └──────┬───────┘                │                        │
           │                        │                        │
           ▼                        │                        │
    ┌──────────────┐                │                        │
    │   WHERE:     │                │                        │
    │ Area ≥ 5000  │                │                        │
    └──────┬───────┘                │                        │
           │                        │                        │
           ▼                        │                        │
    ┌──────────────┐                │                        │
    │   SUITABLE   │                │                        │
    │ AREA PARCELS │◄───────────────┤                        │
    │  (23 rows)   │  NOT EXISTS    │                        │
    └──────┬───────┘  ST_Intersects │                        │
           │                        │                        │
           ▼                        │                        │
    ┌──────────────┐                │                        │
    │   PARCELS    │                │                        │
    │   WITHOUT    │                │                        │
    │  BUILDINGS   │◄────────────────────────────────────────┤
    │  (17 rows)   │                    ST_DWithin 25m       │
    └──────┬───────┘                                         │
           │                                                 │
           ▼                                                 │
    ┌──────────────┐                                         │
    │    FINAL     │                                         │
    │   SCHOOL     │                                         │
    │    SITES     │                                         │
    │   (7 rows)   │                                         │
    │  ⭐ RESULT   │                                         │
    └──────────────┘
```

---

## Key Learnings

1. **Coordinate System Verification** - Always check if data is geographic or projected before analysis
2. **Progressive Filtering** - Build complex queries through cascading views
3. **Spatial Exclusion** - Use `NOT EXISTS` with `ST_Intersects()` to exclude overlapping features
4. **Distance Analysis** - Use `ST_DWithin()` for efficient proximity filtering
5. **Data Validation** - Verify results at each step to ensure correctness

---

## Authors

**Abdallah Alharrem** & **Hossam Shehadeh**  
Spatial Data Analysis Course  
School Site Selection using PostGIS

## License

This project is for educational purposes.
