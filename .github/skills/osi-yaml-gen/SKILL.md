---
name: osi-yaml-gen
description: 'Generate OSI v0.1.1 YAML files with COTALITY vendor extensions for data products from a schema provided as CSV or XLSX. Use when the user wants to build/scaffold a data-product YAML, convert a CSV/XLSX/data-dictionary schema into Cotality extension YAML, validate an existing YAML against the enrichment checklist, or annotate columns with data types, geometry, and metric roles.'
argument-hint: 'attach a CSV/XLSX schema, paste columns, or attach a .yaml to validate'
---

# OSI YAML Generation

Generate a complete OSI v0.1.1 YAML file (with COTALITY vendor extensions) for a
data product, then adversarially review it for geometry issues, missing
enrichments, and LLM pitfalls.

## When to Use
- User attaches a data dictionary (`.xlsx` / `.csv`) and wants a YAML
- User pastes a list of columns and types
- User wants an existing `.yaml` validated against the enrichment checklist

## Inputs
- A schema as CSV, XLSX, or pasted text (column name, type, description)
- OR an existing YAML file to validate

## Procedure

1. **Parse the schema.** Extract each column's name, data type, and description.
   If an XLSX has multiple sheets, identify the one holding the field list.

2. **Map data types** to OSI types using the
   [extension schema reference](./references/cotality-extension-schema.md)
   (`STRING`, `INT64`, `FLOAT64`, `GEOGRAPHY`, `DATE`, `TIMESTAMP`, `BOOL`).

3. **Annotate roles** per column: `is_id`, `is_metric`, `filterable`,
   `is_flag`, `internal`, and `unit` for metrics. See the reference for rules.

4. **Detect geometry.** Check every column against
   [geometry-issues.md](./references/geometry-issues.md). Flag string columns
   holding WKT/WKB/GeoJSON and set `geo_role`, `srid`, `encoding` correctly.

5. **Fill the template.** Populate [assets/yaml-template.yaml](./assets/yaml-template.yaml),
   using [field-examples.md](./references/field-examples.md) as a pattern for each
   column block. Replace all `{PLACEHOLDER}` tokens.

6. **Adversarial review.** Produce a short report listing:
   - geometry problems (wrong type/encoding)
   - missing or weak enrichments
   - likely LLM/text2SQL pitfalls

7. **Ask the data owner.** List anything you couldn't infer confidently
   (units, primary key, value meanings) as explicit questions.

## Output
- The full YAML file (default path `datasets/text2sql/{TABLE_NAME}.yaml`)
- The adversarial review notes
- Open questions for the data owner

## Rules
- Always emit OSI `version: "0.1.1"`.
- Never invent column meanings — flag uncertainty instead.
- Default `srid` to `4326` unless the schema says otherwise.