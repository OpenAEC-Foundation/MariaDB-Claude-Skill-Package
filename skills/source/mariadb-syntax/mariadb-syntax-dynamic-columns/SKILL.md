---
name: mariadb-syntax-dynamic-columns
description: >
  Use when storing sparse or heterogeneous attributes inside a single column, comparing dynamic columns with JSON, migrating legacy MariaDB schemas that use dynamic columns, or porting data in or out of MariaDB.
  Prevents the common mistake of using dynamic columns where JSON would be simpler, the assumption that the feature is portable to MySQL (it is MariaDB-only), and the silent NULL return on a type-mismatched COLUMN_GET.
  Covers COLUMN_CREATE, COLUMN_ADD, COLUMN_GET, COLUMN_DELETE, COLUMN_EXISTS, COLUMN_LIST, COLUMN_CHECK, COLUMN_JSON, blob storage layout, named vs numbered keys, type tags (CHAR, BINARY, INTEGER, UNSIGNED, DECIMAL, DOUBLE, DATE, TIME, DATETIME), nesting up to 10 levels, when to choose dynamic columns vs JSON, and migration patterns to JSON.
  Keywords: dynamic columns, COLUMN_CREATE, COLUMN_GET, COLUMN_LIST, COLUMN_ADD, COLUMN_DELETE, COLUMN_EXISTS, COLUMN_JSON, COLUMN_CHECK, MariaDB only, MySQL incompatible, not in MySQL, sparse columns, EAV pattern, blob storage, key-value in column, BLOB column attributes, schemaless column, nested dynamic columns, how do dynamic columns compare to JSON, migrate dynamic columns to JSON, why is COLUMN_GET returning NULL, type mismatch COLUMN_GET, index dynamic column, virtual column over COLUMN_GET, legacy MariaDB attributes blob
license: MIT
compatibility: "Designed for Claude Code. Requires MariaDB 10.6-LTS, 10.11-LTS, 11.x, 12.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# MariaDB Dynamic Columns : COLUMN_CREATE, COLUMN_GET, COLUMN_JSON and the MariaDB-only key-value blob

Deterministic guidance for working with MariaDB dynamic columns. Use this skill to read or write a dynamic-column blob, choose between dynamic columns and JSON, index a dynamic-column key, and migrate a dynamic-column blob to a JSON document.

## Quick Reference

- **Dynamic columns are MariaDB-only**. The COLUMN_* function family does NOT exist in MySQL at any version. NEVER use dynamic columns if cross-DBMS portability is a requirement. ALWAYS prefer JSON for new schemas unless a measured size or performance advantage of dynamic columns is documented for the workload. Verified : KB `dynamic-columns/`.
- **The eight functions are** : COLUMN_CREATE, COLUMN_ADD, COLUMN_GET, COLUMN_DELETE, COLUMN_EXISTS, COLUMN_LIST, COLUMN_CHECK, COLUMN_JSON. The blob is stored in a single BLOB, MEDIUMBLOB, or LONGBLOB column. There is no dedicated DYNAMIC_COLUMNS type. Verified : KB `dynamic-columns-functions/`.
- **COLUMN_GET REQUIRES a type tag** and ALWAYS returns NULL on a type mismatch. Syntax : `COLUMN_GET(blob, 'name' AS CHAR(64))`. If the key was stored as INTEGER and you ask for it `AS CHAR`, you may still receive a value because the engine coerces ; if the key does not exist, the result is NULL with NO warning. ALWAYS guard with `COLUMN_EXISTS` before relying on a NULL result to mean "absent". Verified : KB `column_get/`.
- **Supported type tags** : `BINARY[(N)]`, `CHAR[(N)]`, `DATE`, `TIME`, `DATETIME`, `DECIMAL[(M,D)]`, `DOUBLE`, `INTEGER`, `SIGNED [INTEGER]`, `UNSIGNED [INTEGER]`. There is NO `JSON` type tag ; nested dynamic columns are flagged by passing a dynamic-column blob as the value of another key. Verified : KB `dynamic-columns/`.
- **NEVER index a dynamic-column blob directly**. The blob is opaque to the optimizer. ALWAYS materialise the wanted key into a `VIRTUAL` or `PERSISTENT` generated column with `COLUMN_GET`, then index that column. The pattern is identical to indexing a JSON path through `JSON_VALUE`. Verified : KB `generated-columns/`.
- **Named columns are the preferred mode** since MariaDB 10.0. Pre-10.0 (now end-of-life) used numbered columns ; numbered mode still parses on modern servers but ALWAYS use named mode in new code. The two modes cannot be mixed inside a single blob. Verified : KB `dynamic-columns/`.
- **Nesting is limited to 10 levels deep**. A dynamic-column value may itself be a dynamic-column blob, recursively, up to 10 nested layers. Deeper structures must be flattened or stored in JSON. Verified : KB `dynamic-columns/`.
- **Max columns per blob = 65 535**. Max blob size = `max_allowed_packet` (default 1 GB on modern releases ; 16 MB on legacy 10.6 stock). For large attribute sets, increase `max_allowed_packet` ; for very wide records, redesign into a related table. Verified : KB `dynamic-columns/`.
- **COLUMN_JSON converts the blob to a JSON document** in standard JSON syntax. ALWAYS use COLUMN_JSON to export, debug, or migrate a dynamic-column blob. The reverse direction (JSON to dynamic columns) requires `COLUMN_CREATE` calls in application code ; there is NO `JSON_TO_DYNAMIC` function.
- **COLUMN_CHECK validates blob integrity**. Returns 1 for a well-formed blob, 0 otherwise. Use COLUMN_CHECK in a `CHECK` constraint to refuse corrupt writes : `CHECK (COLUMN_CHECK(attrs))`.
- **Default to JSON for new schemas**. Dynamic columns are mature but a niche choice : opaque to non-MariaDB tools, harder to debug, and offer no JSONPath equivalent. ALWAYS justify a dynamic-column choice with a concrete reason (measured smaller payload, deeper nesting needs, legacy compatibility).

## Decision Trees

### Tree 1 : "Dynamic columns or JSON?"

```
Is the data ever read by non-MariaDB clients
  (MySQL replica, BI tool, ETL, MongoDB pipeline) ?
  Yes -> JSON. Dynamic columns are an opaque blob to every other tool.

Is portability across DBMS engines a requirement ?
  Yes -> JSON. Dynamic columns do not exist outside MariaDB.

Does the schema need standards-compliant JSONPath
  (JSON_VALUE, JSON_QUERY, JSON_TABLE) ?
  Yes -> JSON. Dynamic columns have no JSONPath equivalent.

Is the data internal to one MariaDB instance, sparse,
  and you have measured that JSON LONGTEXT is too large
  or too slow for the workload ?
  Yes -> dynamic columns are a defensible choice. Document the measurement.

Default : JSON with CHECK (JSON_VALID(col)) and functional indexes
          on virtual columns.
```

### Tree 2 : "Reading a key from a dynamic-column blob"

```
Do you know the key always exists ?
  No  -> first call COLUMN_EXISTS(blob, 'key'). Branch on 1 / 0.
  Yes -> proceed directly to COLUMN_GET.

Do you know the stored type ?
  Yes -> match the COLUMN_GET tag exactly :
         stored CHAR -> AS CHAR(N)
         stored INTEGER -> AS INTEGER (or AS SIGNED / UNSIGNED)
         stored DECIMAL -> AS DECIMAL(M,D)
         stored DATE / TIME / DATETIME -> the matching tag.
  No  -> default to AS CHAR(255) and accept that a stored INTEGER
         is returned as the string "42". Numeric comparisons on the
         result string compare lexicographically, NOT numerically.

The returned value is NULL ?
  Possible causes :
    1. The key does not exist (re-check with COLUMN_EXISTS).
    2. The blob itself is NULL.
    3. The type tag is incompatible with the stored type.
    4. The blob is corrupt (run COLUMN_CHECK).
```

### Tree 3 : "Index a dynamic-column key"

```
Is the key queried by equality or range in WHERE / JOIN / ORDER BY ?
  Yes :
    Step 1 : ALTER TABLE t
               ADD COLUMN k_status VARCHAR(32)
                 AS (COLUMN_GET(attrs, 'status' AS CHAR(32))) VIRTUAL;
    Step 2 : CREATE INDEX ix_status ON t (k_status);
    Why VIRTUAL : computed on the fly, no extra disk space, still indexable.
    Why PERSISTENT : if the COLUMN_GET expression is expensive and the column
                     is read often without using the index path.
  No, the keys are dynamic and never queried directly ?
    -> Do not index. Keep the blob compact.
```

### Tree 4 : "Migrate a dynamic-column blob to JSON"

```
Goal : replace attrs BLOB (dynamic columns) with attrs_json JSON.

Step 1 : ALTER TABLE t ADD COLUMN attrs_json JSON NULL
           CHECK (JSON_VALID(attrs_json));    -- D-010 reminder

Step 2 : UPDATE t SET attrs_json = COLUMN_JSON(attrs);
         -- One-shot conversion. Run on a quiet window or chunked
         -- (WHERE id BETWEEN ? AND ?) on a large table.

Step 3 : Verify : SELECT id FROM t
                  WHERE attrs IS NOT NULL AND attrs_json IS NULL;
         -- Expect zero rows. Rows that fail are likely corrupt blobs
         -- (run COLUMN_CHECK to confirm) and need manual repair.

Step 4 : Rewrite application reads from COLUMN_GET to JSON_VALUE.

Step 5 : Drop the legacy column :
         ALTER TABLE t DROP COLUMN attrs;
         ALTER TABLE t CHANGE COLUMN attrs_json attrs JSON NULL
                       CHECK (JSON_VALID(attrs));
```

## Patterns

### Pattern 1 : Create a row with a dynamic-column blob

```sql
-- 10.6+ : write a sparse attribute set into a single column.
CREATE TABLE product (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  sku VARCHAR(32) NOT NULL UNIQUE,
  attrs BLOB NULL
) ENGINE=InnoDB;

INSERT INTO product (sku, attrs) VALUES (
  'CHAIR-RED-L',
  COLUMN_CREATE(
    'color',  'red'           AS CHAR(16),
    'size',   'L'             AS CHAR(4),
    'weight', 12.5            AS DECIMAL(6,2),
    'stock',  120             AS UNSIGNED INTEGER
  )
);
```

### Pattern 2 : Read a typed key with explicit existence check

```sql
-- 10.6+ : guard with COLUMN_EXISTS, then COLUMN_GET with the right type tag.
SELECT
  id,
  sku,
  CASE
    WHEN COLUMN_EXISTS(attrs, 'stock') = 1
    THEN COLUMN_GET(attrs, 'stock' AS UNSIGNED INTEGER)
    ELSE NULL
  END AS stock_level,
  COLUMN_GET(attrs, 'color' AS CHAR(16)) AS color
FROM product
WHERE sku = 'CHAIR-RED-L';
```

### Pattern 3 : Add or update a key with COLUMN_ADD

```sql
-- 10.6+ : COLUMN_ADD inserts new keys and overwrites existing keys.
UPDATE product
SET attrs = COLUMN_ADD(
  attrs,
  'stock',     90              AS UNSIGNED INTEGER,
  'reorder',   '2026-06-01'    AS DATE,
  'finish',    'matte'         AS CHAR(16)
)
WHERE sku = 'CHAIR-RED-L';
-- 'stock' is overwritten ; 'reorder' and 'finish' are added.
-- Passing NULL as the value for a key DELETES the key.
```

### Pattern 4 : Delete one or more keys with COLUMN_DELETE

```sql
-- 10.6+ : drop a key from the blob.
UPDATE product
SET attrs = COLUMN_DELETE(attrs, 'finish', 'weight')
WHERE sku = 'CHAIR-RED-L';
-- Multiple keys can be deleted in one call.
-- Deleting a non-existent key is a no-op (no warning).
```

### Pattern 5 : List all keys in a blob

```sql
-- 10.6+ : COLUMN_LIST returns a comma-separated list of backtick-quoted names.
SELECT sku, COLUMN_LIST(attrs) AS keys
FROM product
WHERE attrs IS NOT NULL;
-- Example output : `color`,`size`,`stock`,`reorder`
-- NEVER parse this string in SQL. Use it for debug / display only ;
-- iterate keys in application code by name.
```

### Pattern 6 : Export to JSON with COLUMN_JSON

```sql
-- 10.6+ : single-shot conversion to a JSON document.
SELECT id, sku, COLUMN_JSON(attrs) AS attrs_json
FROM product
WHERE sku = 'CHAIR-RED-L';
-- attrs_json example :
-- {"color":"red","size":"L","stock":90,"reorder":"2026-06-01","color":"red"}
-- ALWAYS use COLUMN_JSON for export, audit logs, and migration to JSON column.
```

### Pattern 7 : Index a dynamic-column key via a virtual column

```sql
-- 10.6+ : the blob is opaque ; index a virtual column instead.
ALTER TABLE product
  ADD COLUMN k_color VARCHAR(16)
    AS (COLUMN_GET(attrs, 'color' AS CHAR(16))) VIRTUAL,
  ADD INDEX ix_product_color (k_color);

-- The optimiser uses ix_product_color for :
SELECT id, sku FROM product WHERE k_color = 'red';
-- WHERE COLUMN_GET(attrs, 'color' AS CHAR(16)) = 'red'   -- also works,
-- but ONLY because MariaDB matches the expression to k_color's definition.
-- Always prefer querying the virtual column directly for clarity.
```

### Pattern 8 : Validate blob integrity with CHECK + COLUMN_CHECK

```sql
-- 10.6+ : refuse corrupt writes at schema level.
CREATE TABLE product_v2 (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  sku VARCHAR(32) NOT NULL UNIQUE,
  attrs BLOB NULL,
  CONSTRAINT chk_attrs_blob CHECK (attrs IS NULL OR COLUMN_CHECK(attrs) = 1)
) ENGINE=InnoDB;
-- An INSERT or UPDATE that produces a malformed blob fails with
-- "CONSTRAINT 'chk_attrs_blob' failed".
```

### Pattern 9 : Migrate to JSON in a single transaction

```sql
-- 10.6+ : add a JSON column, populate from COLUMN_JSON, verify, drop legacy.
START TRANSACTION;

ALTER TABLE product
  ADD COLUMN attrs_json JSON NULL
    CHECK (attrs_json IS NULL OR JSON_VALID(attrs_json));

UPDATE product
  SET attrs_json = COLUMN_JSON(attrs)
  WHERE attrs IS NOT NULL;

-- Verify no rows failed conversion.
SELECT COUNT(*) AS failed
FROM product
WHERE attrs IS NOT NULL AND attrs_json IS NULL;
-- If failed > 0, ROLLBACK and inspect with COLUMN_CHECK(attrs).

COMMIT;
```

### Pattern 10 : Nested dynamic columns (up to 10 levels)

```sql
-- 10.6+ : a dynamic-column value can itself be a dynamic-column blob.
INSERT INTO product (sku, attrs) VALUES (
  'BIKE-MTB-29',
  COLUMN_CREATE(
    'frame',   'aluminium' AS CHAR(16),
    'wheels',  COLUMN_CREATE(
                 'diameter', 29  AS UNSIGNED INTEGER,
                 'tubeless', 'Y' AS CHAR(1)
               )
  )
);
-- To read a nested key, COLUMN_GET the outer key as BINARY first,
-- then COLUMN_GET the inner key from the result.
SELECT
  COLUMN_GET(
    COLUMN_GET(attrs, 'wheels' AS BINARY),
    'diameter' AS UNSIGNED INTEGER
  ) AS diameter
FROM product
WHERE sku = 'BIKE-MTB-29';
-- ALWAYS flatten or migrate to JSON if nesting exceeds 2 levels :
-- the SQL is unreadable and error-prone.
```

## Reference Links

- `references/methods.md` : full COLUMN_* function signatures, type tags, return types, version notes, and KB links.
- `references/examples.md` : 10+ working dynamic-column examples (create, read, add, delete, list, JSON, virtual-column index, migration).
- `references/anti-patterns.md` : 6+ dynamic-column mistakes with the fix.

## External References

- MariaDB KB : `https://mariadb.com/kb/en/dynamic-columns/`
- MariaDB KB : `https://mariadb.com/kb/en/dynamic-columns-functions/`
- MariaDB KB : `https://mariadb.com/kb/en/column_create/`
- MariaDB KB : `https://mariadb.com/kb/en/column_get/`
- MariaDB KB : `https://mariadb.com/kb/en/column_add/`
- MariaDB KB : `https://mariadb.com/kb/en/column_delete/`
- MariaDB KB : `https://mariadb.com/kb/en/column_exists/`
- MariaDB KB : `https://mariadb.com/kb/en/column_list/`
- MariaDB KB : `https://mariadb.com/kb/en/column_check/`
- MariaDB KB : `https://mariadb.com/kb/en/column_json/`
- MariaDB KB : `https://mariadb.com/kb/en/generated-columns/`
