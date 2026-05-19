# MariaDB Dynamic Columns : working examples

Twelve copy-ready, version-annotated examples. Every snippet is verified against the MariaDB Knowledge Base and runs on MariaDB 10.6-LTS, 10.11-LTS, 11.x, and 12.x unless noted otherwise.

## Example 1 : Create a blob with mixed types

```sql
-- 10.6+ : create a table and insert one row with five typed keys.
CREATE TABLE product (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  sku VARCHAR(32) NOT NULL UNIQUE,
  attrs BLOB NULL
) ENGINE=InnoDB;

INSERT INTO product (sku, attrs) VALUES (
  'CHAIR-RED-L',
  COLUMN_CREATE(
    'color',     'red'         AS CHAR(16),
    'size',      'L'           AS CHAR(4),
    'weight_kg', 12.50         AS DECIMAL(6,2),
    'in_stock',  120           AS UNSIGNED INTEGER,
    'released',  '2026-01-15'  AS DATE
  )
);
```

## Example 2 : Read multiple typed keys in one SELECT

```sql
-- 10.6+ : SELECT each key with its matching type tag.
SELECT
  sku,
  COLUMN_GET(attrs, 'color'     AS CHAR(16))         AS color,
  COLUMN_GET(attrs, 'size'      AS CHAR(4))          AS size,
  COLUMN_GET(attrs, 'weight_kg' AS DECIMAL(6,2))     AS weight_kg,
  COLUMN_GET(attrs, 'in_stock'  AS UNSIGNED INTEGER) AS in_stock,
  COLUMN_GET(attrs, 'released'  AS DATE)             AS released
FROM product
WHERE sku = 'CHAIR-RED-L';
```

## Example 3 : Guard a read with COLUMN_EXISTS

```sql
-- 10.6+ : distinguish "key missing" from "key present with NULL value".
SELECT
  sku,
  COLUMN_EXISTS(attrs, 'weight_kg') AS has_weight,
  CASE
    WHEN COLUMN_EXISTS(attrs, 'weight_kg') = 1
    THEN COLUMN_GET(attrs, 'weight_kg' AS DECIMAL(6,2))
    ELSE NULL
  END AS weight_kg
FROM product;
```

## Example 4 : Add and update keys with COLUMN_ADD

```sql
-- 10.6+ : COLUMN_ADD upserts. 'in_stock' is overwritten, 'finish' is new.
UPDATE product
SET attrs = COLUMN_ADD(
  attrs,
  'in_stock', 95          AS UNSIGNED INTEGER,
  'finish',   'matte'     AS CHAR(16),
  'reorder',  '2026-06-01' AS DATE
)
WHERE sku = 'CHAIR-RED-L';

-- Delete a key by setting its value to NULL via COLUMN_ADD.
UPDATE product
SET attrs = COLUMN_ADD(attrs, 'finish', NULL)
WHERE sku = 'CHAIR-RED-L';
-- After this UPDATE : COLUMN_EXISTS(attrs, 'finish') = 0.
```

## Example 5 : Delete keys with COLUMN_DELETE

```sql
-- 10.6+ : remove multiple keys in one statement.
UPDATE product
SET attrs = COLUMN_DELETE(attrs, 'released', 'weight_kg')
WHERE sku = 'CHAIR-RED-L';
-- Deleting a non-existent key is a no-op (no warning).
```

## Example 6 : List all keys

```sql
-- 10.6+ : human-readable key list. NEVER parse the string in SQL.
SELECT sku, COLUMN_LIST(attrs) AS keys_in_blob
FROM product
WHERE attrs IS NOT NULL;
-- Example output : `color`,`size`,`in_stock`,`reorder`
```

## Example 7 : Convert the blob to JSON

```sql
-- 10.6+ : COLUMN_JSON dumps the blob as a JSON object.
SELECT sku, COLUMN_JSON(attrs) AS attrs_json
FROM product
WHERE sku = 'CHAIR-RED-L';
-- attrs_json example :
-- {"color":"red","size":"L","in_stock":95,"reorder":"2026-06-01"}
```

## Example 8 : Validate blob integrity with a CHECK constraint

```sql
-- 10.6+ : refuse corrupt blob writes at schema level.
CREATE TABLE product_v2 (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  sku VARCHAR(32) NOT NULL UNIQUE,
  attrs BLOB NULL,
  CONSTRAINT chk_attrs_blob
    CHECK (attrs IS NULL OR COLUMN_CHECK(attrs) = 1)
) ENGINE=InnoDB;

-- A well-formed blob passes :
INSERT INTO product_v2 (sku, attrs) VALUES (
  'OK-ROW',
  COLUMN_CREATE('color', 'red' AS CHAR(8))
);

-- A random byte string fails :
INSERT INTO product_v2 (sku, attrs) VALUES (
  'BAD-ROW',
  0xDEADBEEF
);
-- ERROR 4025 : CONSTRAINT `chk_attrs_blob` failed.
```

## Example 9 : Index a dynamic-column key via a virtual column

```sql
-- 10.6+ : the blob is opaque ; expose 'color' as a virtual column and index it.
ALTER TABLE product
  ADD COLUMN k_color VARCHAR(16)
    AS (COLUMN_GET(attrs, 'color' AS CHAR(16))) VIRTUAL,
  ADD INDEX ix_product_color (k_color);

-- The optimiser uses ix_product_color for :
EXPLAIN
SELECT id, sku
FROM product
WHERE k_color = 'red';
-- Expect : type = ref, key = ix_product_color.
```

## Example 10 : Nested dynamic columns (read two levels)

```sql
-- 10.6+ : a dynamic-column value may itself be a dynamic-column blob.
INSERT INTO product (sku, attrs) VALUES (
  'BIKE-MTB-29',
  COLUMN_CREATE(
    'frame',  'aluminium' AS CHAR(16),
    'wheels', COLUMN_CREATE(
                'diameter', 29  AS UNSIGNED INTEGER,
                'tubeless', 'Y' AS CHAR(1)
              )
  )
);

-- Read 'wheels.diameter' : outer COLUMN_GET as BINARY, then inner COLUMN_GET.
SELECT
  sku,
  COLUMN_GET(
    COLUMN_GET(attrs, 'wheels' AS BINARY),
    'diameter' AS UNSIGNED INTEGER
  ) AS wheel_diameter
FROM product
WHERE sku = 'BIKE-MTB-29';
-- ALWAYS flatten or migrate to JSON if nesting exceeds 2 levels.
```

## Example 11 : Migrate one column from dynamic to JSON in a transaction

```sql
-- 10.6+ : safe online migration to a JSON column.
START TRANSACTION;

-- Step 1 : add a JSON column protected by JSON_VALID (D-010).
ALTER TABLE product
  ADD COLUMN attrs_json JSON NULL
    CHECK (attrs_json IS NULL OR JSON_VALID(attrs_json));

-- Step 2 : single-shot conversion.
UPDATE product
SET attrs_json = COLUMN_JSON(attrs)
WHERE attrs IS NOT NULL;

-- Step 3 : verify zero conversion failures.
SELECT COUNT(*) AS failed_rows
FROM product
WHERE attrs IS NOT NULL AND attrs_json IS NULL;
-- Expect 0. If non-zero : ROLLBACK and inspect rows with COLUMN_CHECK.

COMMIT;

-- Step 4 : after the application has been updated to read attrs_json,
-- drop the legacy blob.
ALTER TABLE product DROP COLUMN attrs;
```

## Example 12 : Export a dynamic-column table to JSON for MySQL import

```sql
-- 10.6+ : produce a portable JSON dump because MySQL has no COLUMN_*.
SELECT
  id,
  sku,
  COLUMN_JSON(attrs) AS attrs_json
INTO OUTFILE '/var/lib/mysql-files/product_export.csv'
FIELDS TERMINATED BY ',' OPTIONALLY ENCLOSED BY '"'
LINES TERMINATED BY '\n'
FROM product;
-- The destination MySQL schema receives attrs_json as a native JSON column.
-- NEVER attempt to ship the raw BLOB to MySQL : the format is unreadable
-- without a MariaDB COLUMN_JSON call.
```
