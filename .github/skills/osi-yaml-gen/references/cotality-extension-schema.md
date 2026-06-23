# COTALITY Extension Quick Reference

## Extension JSON Fields (inside `custom_extensions[].data`)

### Required on Every Field
| Field | Values |
|-------|--------|
| `data_type` | `STRING`, `INT64`, `FLOAT64`, `GEOGRAPHY`, `DATE`, `TIMESTAMP`, `BOOL`, `ARRAY<STRING>` |

### Role Annotations
| Field | Type | When |
|-------|------|------|
| `is_id` | bool | Primary key / unique identifier |
| `is_metric` | bool | Aggregatable numeric (SUM/AVG/COUNT) |
| `filterable` | bool | Commonly used in WHERE |
| `is_flag` | bool | 0/1 or Y/N boolean stored as number/string |
| `internal` | bool | ETL/pipeline column — exclude from text2SQL |

### Geometry
| Field | Type | Values |
|-------|------|--------|
| `geo_role` | string | `"geometry"` or `"lat_lon_pair"` |
| `srid` | int | Usually `4326` |
| `encoding` | string | `"native"` (BQ GEOGRAPHY), `"wkt"`, `"wkb"`, `"geojson"` |
| `paired_with` | string | Name of the paired lat/lon column |
| `geometry_type` | string | `"point"`, `"polygon"`, `"line"` |
| `preferred_display_geometry` | object | `{table, column, join_path, geometry_type, reason}` |

### Coded Columns
| Field | Type | When |
|-------|------|------|
| `high_cardinality` | bool | 50+ distinct values |
| `has_lookup` | bool | Only when served via MCP/text2SQL |
| `related_description_column` | string | Points to human-readable description column |
| `value_groups` | object | Semantic groupings (see below) |
| `sample_values` | array | 3-5 representative values |
| `values_hint` | string | Short text hint about valid values |

### Metrics
| Field | Type | When |
|-------|------|------|
| `unit` | string | `"USD"`, `"sqft"`, `"acres"`, `"year"`, `"percent"` |

### Dedup / Relationships
| Field | Type | Values |
|-------|------|--------|
| `dedup_role` | string | `"group_key"` or `"sort_key"` |

## value_groups Structure

```json
{
  "group_name": {
    "description": "Plain English meaning",
    "codes": ["10", "11"],          // Low cardinality: exhaustive
    "sample_codes": ["1000", "1001"], // High cardinality: representative
    "note": "Cross-reference hint"
  }
}
```

## Dataset-Level Extension
Note: query_role, lookup_columns, and spatial_filter_note only apply to spatial-reference tables served via MCP — omit for plain YAML generation.
```json
{
  "tags": ["property-intel"],
  "row_estimate": 155000000,
  "query_role": "spatial_filter",           // For spatial-reference tables only
  "lookup_columns": ["county", "state"],    // For spatial-reference tables only
  "spatial_filter_note": "...",             // For spatial-reference tables only
  "sample_questions": [
    {
      "id": 1,
      "bucket": "Category Name",
      "q": "Natural language question",
      "context": "Hints for the LLM",
      "sql": "SELECT ... FROM ..."
    }
  ]
}
```

## Relationship-Level Extension

```json
{
  "cardinality": "one_to_many",
  "group_key": "parcel_id",
  "sort_key": "clip",
  "default_filter": "address_type = 'PROPERTY'"
}
```