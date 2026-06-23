# Geometry Column Detection & Common Problems

## How Teams Get Geometry Wrong

Teams frequently mishandle spatial data in schema dictionaries. This reference helps the skill detect and flag these issues.

---

## Problem 1: String Column Storing Spatial Data

**Symptom:** Column is `STRING` / `AlphaNumeric` but stores WKT, WKB, or GeoJSON.

**Detection patterns:**
- Column name contains: `geometry`, `geom`, `shape`, `polygon`, `boundary`, `geography`, `wkt`, `wkb`
- Description mentions: "Well-Known Text", "WKT", "WKB", "GeoJSON", "geometry string"
- Sample values start with: `POINT(`, `POLYGON((`, `MULTIPOLYGON`, `{"type":`, `0x01`

**Correct annotation:**
```json
{"data_type": "STRING", "geo_role": "geometry", "srid": 4326, "encoding": "wkt"}
```
or
```json
{"data_type": "STRING", "geo_role": "geometry", "srid": 4326, "encoding": "wkb"}
```

**LLM instruction to add:**
> Stored as WKT/WKB string. For spatial operations, cast with `ST_GEOGFROMTEXT(col)` (WKT) or `ST_GEOGFROMWKB(col)` (WKB). For CARTO tile rendering, the engine handles conversion.

---

## Problem 2: Misnamed WKT/WKB Columns

**Symptom:** Column named `*_wkt` but actually contains WKB hex bytes, or vice versa.

**Detection:** Usually requires sample values. Flag for user confirmation:
- WKT looks like: `POINT(-73.98 40.75)`, `POLYGON((-73 40, -74 41, ...))`
- WKB looks like: `0101000020E6100000...` (hex string starting with `01`)
- GeoJSON looks like: `{"type": "Point", "coordinates": [-73.98, 40.75]}`

**Review output:**
```
🟡 WARNING: Column 'location_wkt' — name implies WKT but verify actual encoding.
   WKT = human-readable (POINT(-73 40)), WKB = hex binary (0101000020...).
   If misnamed, set encoding: "wkb" and add description note about the naming inconsistency.
```

---

## Problem 3: GeoJSON in String Columns

**Symptom:** Column stores GeoJSON as a JSON string.

**Detection:**
- Description mentions "GeoJSON"
- Sample values show `{"type": "Point", "coordinates": [...]}`
- Column named `geojson`, `geo_json`, `*_geojson`

**Correct annotation:**
```json
{"data_type": "STRING", "geo_role": "geometry", "srid": 4326, "encoding": "geojson"}
```

**LLM instruction to add:**
> Stored as GeoJSON string. For spatial operations: `ST_GEOGFROMGEOJSON(col)`. Note: GeoJSON uses [lon, lat] order.

---

## Problem 4: Lat/Lon Pair Issues

**Symptom:** Latitude and longitude columns exist but are not properly paired.

**Common mistakes:**
| Issue | Detection |
|-------|-----------|
| Only latitude, no longitude | Unpaired — cannot render |
| Lat/lon in wrong columns | Description says "latitude" but column is named `*_longitude` |
| Multiple lat/lon pairs | e.g., `centroid_lat/lon` AND `location_lat/lon` — which is primary? |
| Lat/lon stored as integers | Multiplied by 1,000,000 (millionths of degrees) — needs division |
| Swapped lat/lon | Latitude values > 90 or Longitude values between -90 and 90 |

**Pairing rules:**
1. Match by prefix: `location_latitude` ↔ `location_longitude`
2. Match by proximity: adjacent columns with lat/lon in name
3. If multiple pairs: flag for user to designate primary

**Correct annotation (both columns):**
```json
// On latitude column:
{"data_type": "FLOAT64", "geo_role": "lat_lon_pair", "paired_with": "location_longitude", "srid": 4326}

// On longitude column:
{"data_type": "FLOAT64", "geo_role": "lat_lon_pair", "paired_with": "location_latitude", "srid": 4326}
```

---

## Problem 5: Geohash Confused as Geometry

**Symptom:** Column named `geohash` or `*_geohash` incorrectly annotated as a geometry column.

**Rule:** Geohash is an INFORMATIONAL encoding of lat/lon. It is NOT a renderable geometry.
- Do NOT set `geo_role` on geohash columns
- Do NOT include in CARTO tile SELECT
- Geohash is useful for approximate bucketing/clustering only

---

## Problem 6: Missing SRID

**Symptom:** Geometry column with no SRID specified.

**Rule:** Default to `srid: 4326` (WGS84, standard for web maps). Flag if:
- Description mentions "state plane", "UTM", "NAD83" → not 4326
- Coordinates in meters instead of degrees → projected CRS, not 4326
- Values > 180 for longitude or > 90 for latitude → likely projected

---

## Problem 7: Multiple Geometry Columns

**Symptom:** Table has both point and polygon geometry.

**Examples:**
- `clip` table: `geometry` (point) + could JOIN to `parcels.geometry` (polygon)
- `address` table: `geometry` (geocoded point) — no polygon in same table

**Rules:**
1. If table has ONE geometry column: it's primary, annotate normally
2. If table has point + polygon: add `preferred_display_geometry` on the point
3. If geometry is in ANOTHER table via JOIN: document in `ai_context.instructions`

---

## Problem 8: Centroid Columns

**Symptom:** Columns named `centroid_latitude`, `centroid_longitude`, `parcel_centroid_*`.

**Rules:**
- These are still lat/lon pairs — annotate with `geo_role: "lat_lon_pair"`
- If the table ALSO has a native geometry column, note which to prefer
- Add in description: "Centroid point — may not fall inside irregular parcels"
- If table has both `centroid_lat/lon` AND `location_lat/lon`, document the difference

---

## Decision Tree for Geometry Detection

```
Column name/description suggests spatial?
├── NO → Skip (not geometry)
└── YES → What's the data_type?
    ├── GEOGRAPHY (native BQ) → geo_role: "geometry", encoding: "native"
    ├── STRING with "wkt" → geo_role: "geometry", encoding: "wkt" (verify not WKB)
    ├── STRING with "wkb" → geo_role: "geometry", encoding: "wkb"
    ├── STRING with "geojson" → geo_role: "geometry", encoding: "geojson"
    ├── STRING with "geometry/shape" → FLAG: ask user for encoding
    ├── FLOAT64 with "lat/latitude" → geo_role: "lat_lon_pair", find pair
    ├── FLOAT64 with "lon/longitude/lng" → geo_role: "lat_lon_pair", find pair
    └── STRING with "geohash" → NOT geometry, skip
```

---

## Review Template for Geometry Issues

When generating the adversarial review, format geometry findings as:

```markdown
### 🔴 Geometry Issues

**Columns detected as spatial: {count}**

| Column | Inferred Type | Confidence | Issue |
|--------|--------------|------------|-------|
| location_wkt | WKB string | ⚠️ Medium | Name says WKT but verify encoding |
| parcel_centroid_lat | lat/lon pair | ✅ High | Paired with parcel_centroid_lon |
| geohash | NOT geometry | ✅ High | Informational only, not renderable |

**Questions for the data owner:**
1. Column `location_wkt`: Is this actually WKT (POINT(-73 40)) or WKB (hex: 0101...)?
2. Are coordinates in WGS84 (EPSG:4326) or a projected CRS?
3. Multiple lat/lon pairs detected — which is the primary for map display?
```
