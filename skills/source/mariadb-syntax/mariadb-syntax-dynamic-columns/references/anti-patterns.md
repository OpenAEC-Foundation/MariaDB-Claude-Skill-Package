# MariaDB Dynamic Columns : anti-patterns

Eight common mistakes when using dynamic columns, each with a concrete failing example, the reason it fails, and the correct alternative.

## Anti-Pattern 1 : Using dynamic columns for data that must be portable

```sql
-- BAD : the application also runs on a MySQL replica or a Postgres reporting clone.
CREATE TABLE product (
  id BIGINT PRIMARY KEY,
  attrs BLOB NULL  -- COLUMN_CREATE(...) on MariaDB ; unreadable elsewhere.
);
```

**Why this fails** : Dynamic columns are MariaDB-only. MySQL does NOT have COLUMN_GET, COLUMN_JSON, or any companion function. Any downstream consumer (replica, ETL, BI tool) receives an opaque binary blob it cannot decode. Once written, the data is locked to MariaDB.

**Fix** : Use JSON instead. Dump existing blobs with `COLUMN_JSON(attrs)` and write to a JSON column.

```sql
-- GOOD : portable JSON, indexable through virtual columns, readable everywhere.
CREATE TABLE product (
  id BIGINT PRIMARY KEY,
  attrs JSON NULL CHECK (attrs IS NULL OR JSON_VALID(attrs))
);
```

## Anti-Pattern 2 : COLUMN_GET with the wrong type tag and silent NULL

```sql
-- BAD : stored as UNSIGNED INTEGER, read as DATE.
INSERT INTO product (id, attrs) VALUES (
  1, COLUMN_CREATE('in_stock', 120 AS UNSIGNED INTEGER)
);

SELECT COLUMN_GET(attrs, 'in_stock' AS DATE) FROM product WHERE id = 1;
-- Returns NULL with no error. The application sees "no stock" instead of "wrong type".
```

**Why this fails** : COLUMN_GET returns NULL on type mismatch with no warning by default. The application cannot distinguish "key missing" from "key present but wrong type requested".

**Fix** : ALWAYS match the type tag to the stored type. ALWAYS guard with `COLUMN_EXISTS` when NULL is a legitimate business value.

```sql
-- GOOD : matching tag returns the correct value ; existence is checked separately.
SELECT
  CASE
    WHEN COLUMN_EXISTS(attrs, 'in_stock') = 1
    THEN COLUMN_GET(attrs, 'in_stock' AS UNSIGNED INTEGER)
    ELSE NULL
  END AS in_stock
FROM product WHERE id = 1;
```

## Anti-Pattern 3 : Indexing the dynamic-column blob directly

```sql
-- BAD : CREATE INDEX on COLUMN_GET expression in the WHERE clause.
EXPLAIN
SELECT id FROM product
WHERE COLUMN_GET(attrs, 'color' AS CHAR(16)) = 'red';
-- type = ALL, full table scan. No index is created or usable.

-- BAD : trying to put COLUMN_GET in an index definition directly.
CREATE INDEX ix_color ON product ((COLUMN_GET(attrs, 'color' AS CHAR(16))));
-- ERROR : functional indexes on arbitrary expressions are not supported.
```

**Why this fails** : The dynamic-column blob is opaque to the optimiser. `CREATE INDEX` cannot accept an arbitrary expression. Every WHERE clause that calls `COLUMN_GET` triggers a full table scan.

**Fix** : Materialise the key into a `VIRTUAL` or `PERSISTENT` generated column, then index that column. Identical pattern to indexing a JSON path.

```sql
-- GOOD : index a virtual column.
ALTER TABLE product
  ADD COLUMN k_color VARCHAR(16)
    AS (COLUMN_GET(attrs, 'color' AS CHAR(16))) VIRTUAL,
  ADD INDEX ix_product_color (k_color);

EXPLAIN
SELECT id FROM product WHERE k_color = 'red';
-- type = ref, key = ix_product_color.
```

## Anti-Pattern 4 : Treating the blob as a "free schema" with no validation

```sql
-- BAD : every application caller writes arbitrary keys with arbitrary types.
UPDATE product SET attrs = COLUMN_ADD(attrs, 'colour', 'red' AS CHAR(16));   -- British spelling.
UPDATE product SET attrs = COLUMN_ADD(attrs, 'color',  'red' AS CHAR(16));   -- US spelling.
UPDATE product SET attrs = COLUMN_ADD(attrs, 'Color',  'red' AS CHAR(16));   -- mixed case.
-- Three distinct keys now hold the same logical value. Reads miss two of them.
```

**Why this fails** : Dynamic columns provide NO schema enforcement on key names, no type checking on writes, and no required-key validation. The blob accepts anything. Without application-level discipline, the schema drifts within hours.

**Fix** : Either (a) wrap all writes in a stored procedure that normalises key names and types, OR (b) move to a JSON column with a CHECK constraint that validates structure with `JSON_SCHEMA_VALID` (10.9+) or path-by-path with `JSON_VALUE`.

```sql
-- GOOD (option a) : single write path via a procedure.
DELIMITER //
CREATE PROCEDURE set_product_color(IN p_id BIGINT, IN p_color VARCHAR(16))
BEGIN
  UPDATE product
  SET attrs = COLUMN_ADD(attrs, 'color', p_color AS CHAR(16))
  WHERE id = p_id;
END //
DELIMITER ;
```

## Anti-Pattern 5 : Storing more than `max_allowed_packet` in a single blob

```sql
-- BAD : append-only audit log inside one row's dynamic-column blob.
UPDATE audit
SET attrs = COLUMN_ADD(
  attrs,
  CONCAT('event_', UNIX_TIMESTAMP()), JSON_OBJECT(...)
)
WHERE id = 1;
-- After weeks of appends : ERROR 1153 (08S01) : Got a packet bigger than 'max_allowed_packet' bytes.
```

**Why this fails** : The whole blob is rewritten on every COLUMN_ADD and must fit in `max_allowed_packet` (default 1 GB modern, 16 MB on legacy 10.6 stock). Each UPDATE also re-transmits the entire blob between client and server. Performance degrades quadratically with size.

**Fix** : Move append-mostly data to a related table. Dynamic columns are designed for SPARSE, RELATIVELY STATIC attribute sets, not for log accumulation.

```sql
-- GOOD : event-log table with one row per event.
CREATE TABLE audit_event (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  audit_id BIGINT UNSIGNED NOT NULL,
  occurred_at DATETIME(6) NOT NULL,
  payload JSON NULL CHECK (payload IS NULL OR JSON_VALID(payload)),
  INDEX ix_audit_event_audit (audit_id, occurred_at)
) ENGINE=InnoDB;
```

## Anti-Pattern 6 : Mixing dynamic columns and JSON in the same row without a clear contract

```sql
-- BAD : two columns hold overlapping data with no rule for which wins.
CREATE TABLE product (
  id BIGINT PRIMARY KEY,
  attrs       BLOB NULL,   -- dynamic columns
  attrs_json  JSON NULL    -- JSON, sometimes synced, sometimes not
);
-- Different code paths write to one or the other ; reads must consult both.
```

**Why this fails** : Two sources of truth for one logical concept guarantees divergence. Every read needs reconciliation logic. The schema cannot answer "what is the current value of `color`?" without looking at both columns and applying an undocumented priority rule.

**Fix** : Pick one representation. If migrating, finish the migration in a single transaction (see Example 11 in `examples.md`) and DROP the legacy column.

```sql
-- GOOD : single column, single contract.
ALTER TABLE product DROP COLUMN attrs;
-- Keep only attrs_json, with CHECK (JSON_VALID(attrs_json)).
```

## Anti-Pattern 7 : Relying on COLUMN_LIST output as a parseable contract

```sql
-- BAD : parse the comma list in SQL to feed a downstream IN clause.
SELECT id FROM product
WHERE COLUMN_LIST(attrs) LIKE '%`color`%';
-- Works by accident. Breaks the day a key name contains a comma or backtick.
```

**Why this fails** : `COLUMN_LIST` returns a backtick-quoted comma-separated list intended for HUMAN display. Key names containing commas, backticks, or non-ASCII characters break naive string parsing. The order of names is also implementation-defined.

**Fix** : Use `COLUMN_EXISTS` for presence checks, `COLUMN_JSON` when you need structured access, or iterate keys in application code by name.

```sql
-- GOOD : presence check uses the dedicated function.
SELECT id FROM product
WHERE COLUMN_EXISTS(attrs, 'color') = 1;
```

## Anti-Pattern 8 : Deeply nested dynamic columns (more than 2 levels)

```sql
-- BAD : five-level nesting inside one blob.
INSERT INTO product (sku, attrs) VALUES (
  'NESTED',
  COLUMN_CREATE('a', COLUMN_CREATE('b', COLUMN_CREATE('c', COLUMN_CREATE('d', COLUMN_CREATE('e', 1 AS UNSIGNED INTEGER)))))
);

-- Reading `a.b.c.d.e` :
SELECT COLUMN_GET(
  COLUMN_GET(
    COLUMN_GET(
      COLUMN_GET(
        COLUMN_GET(attrs, 'a' AS BINARY),
        'b' AS BINARY
      ),
      'c' AS BINARY
    ),
    'd' AS BINARY
  ),
  'e' AS UNSIGNED INTEGER
) FROM product WHERE sku = 'NESTED';
-- Unreadable, slow to write, slow to debug. Max-nesting cap (10 levels) becomes a real limit.
```

**Why this fails** : Nested dynamic columns have no JSONPath equivalent. Every level needs another COLUMN_GET wrapped around the previous one. The hard cap is 10 levels ; even 3 levels are painful to maintain. The optimiser cannot help with virtual-column shortcuts past one level.

**Fix** : Flatten or migrate to JSON. JSON has `JSON_VALUE(doc, '$.a.b.c.d.e')` and `JSON_TABLE` for hierarchical reads.

```sql
-- GOOD : JSON with a path expression.
SELECT JSON_VALUE(attrs_json, '$.a.b.c.d.e')
FROM product WHERE sku = 'NESTED';
```
