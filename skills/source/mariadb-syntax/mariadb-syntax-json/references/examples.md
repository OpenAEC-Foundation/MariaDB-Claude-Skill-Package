# MariaDB JSON : Working Examples

12+ verified, runnable examples for MariaDB 10.6-LTS, 10.11-LTS, 11.x, 12.x. Every query verified against the KB pages listed in `methods.md`. Version annotations on every example.

## Example 1 : CREATE TABLE with JSON and validation

```sql
-- 10.2+ : JSON column with explicit CHECK (JSON_VALID(...)).
CREATE TABLE event (
  id         BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  payload    JSON NOT NULL CHECK (JSON_VALID(payload)),
  created_at DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
) ENGINE=InnoDB;

-- Valid insert :
INSERT INTO event (payload)
VALUES ('{"type":"order.created","order_id":42}');

-- Invalid insert : raises ER_CONSTRAINT_FAILED (4025).
INSERT INTO event (payload)
VALUES ('not even json');
```

## Example 2 : JSON_EXTRACT versus JSON_VALUE versus JSON_QUERY

```sql
-- 10.2+ : three extraction functions on the same document.
SET @doc = '{"status":"open","priority":7,"tags":["red","blue"]}';

SELECT
  JSON_EXTRACT(@doc, '$.status')  AS extract_scalar,    -- "open"   (JSON-quoted)
  JSON_VALUE  (@doc, '$.status')  AS value_scalar,      -- open     (unwrapped)
  JSON_EXTRACT(@doc, '$.tags')    AS extract_array,     -- ["red","blue"]
  JSON_QUERY  (@doc, '$.tags')    AS query_array;       -- ["red","blue"]

-- The trap : JSON_EXTRACT in a string comparison.
SELECT JSON_EXTRACT(@doc,'$.status') = 'open' AS extract_eq,  -- 0  (NEVER matches)
       JSON_VALUE  (@doc,'$.status') = 'open' AS value_eq;    -- 1  (matches)
```

## Example 3 : JSON_TABLE projecting an array (10.6+)

```sql
-- 10.6+ : flatten a JSON array into relational rows.
SET @doc = '[
  {"name":"Laptop","color":"black","price":1000},
  {"name":"Jeans","color":"blue"}
]';

SELECT id, name, color, price
FROM JSON_TABLE(@doc, '$[*]' COLUMNS (
  id    FOR ORDINALITY,
  name  VARCHAR(40)   PATH '$.name',
  color VARCHAR(20)   PATH '$.color',
  price DECIMAL(10,2) PATH '$.price' DEFAULT '0' ON EMPTY
)) AS jt;

-- 1  Laptop  black  1000.00
-- 2  Jeans   blue      0.00
```

## Example 4 : JSON_TABLE with NESTED PATH (10.6+)

```sql
-- 10.6+ : NESTED PATH explodes a nested array per outer row.
SET @doc = '{
  "orders":[
    {"id":1,"items":["a","b"]},
    {"id":2,"items":["c"]}
  ]
}';

SELECT order_id, item
FROM JSON_TABLE(@doc, '$.orders[*]' COLUMNS (
  order_id INT PATH '$.id',
  NESTED PATH '$.items[*]' COLUMNS (
    item VARCHAR(40) PATH '$'
  )
)) AS jt;

-- 1  a
-- 1  b
-- 2  c
```

## Example 5 : JSON_MERGE_PATCH for partial-update API

```sql
-- 10.2+ : RFC 7396 PATCH semantics.
SET @current = '{"name":"alice","email":"a@x","verified":true}';
SET @patch   = '{"email":"alice@x","verified":null}';

SELECT JSON_MERGE_PATCH(@current, @patch) AS patched;
-- {"name":"alice","email":"alice@x"}
-- The key "verified" was REMOVED because the patch value was JSON null.
```

## Example 6 : JSON_MERGE_PRESERVE for array concat and key preservation

```sql
-- 10.2+ : PRESERVE concatenates arrays and preserves all keys.
SELECT JSON_MERGE_PRESERVE('[1,2]', '[2,3]')           AS arr_merge,    -- [1, 2, 2, 3]
       JSON_MERGE_PRESERVE('{"k":1}', '{"k":2}')       AS key_merge;    -- {"k":[1,2]}

-- NEVER use the deprecated JSON_MERGE :
-- SELECT JSON_MERGE('[1]','[2]');   -- alias for PRESERVE, NOT for PATCH.
```

## Example 7 : Functional index on JSON path via persistent column (D-010)

```sql
-- 10.6+ : standard pattern for indexing a JSON path.
CREATE TABLE event (
  id      BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  payload JSON NOT NULL CHECK (JSON_VALID(payload)),
  -- PERSISTENT : materialised, fastest lookup, costs disk + write CPU.
  event_type VARCHAR(64) AS (JSON_VALUE(payload, '$.type')) PERSISTENT,
  -- VIRTUAL : computed on read, zero disk overhead, still indexable.
  -- Pick one based on read / write balance.
  INDEX ix_event_type (event_type)
) ENGINE=InnoDB;

INSERT INTO event (payload) VALUES
  ('{"type":"order.created","order_id":1}'),
  ('{"type":"order.shipped","order_id":1}'),
  ('{"type":"payment.received","order_id":1}');

-- The query that uses the index :
EXPLAIN SELECT id FROM event WHERE event_type = 'order.created';
-- Expect : type=ref, key=ix_event_type.
```

## Example 8 : JSON_SET, JSON_INSERT, JSON_REPLACE, JSON_REMOVE side-by-side

```sql
-- 10.2+ : four mutation functions on the same document.
SET @doc = '{"a":1,"b":2}';

SELECT JSON_SET    (@doc, '$.a', 9, '$.c', 3) AS set_doc;      -- {"a":9,"b":2,"c":3}
SELECT JSON_INSERT (@doc, '$.a', 9, '$.c', 3) AS insert_doc;   -- {"a":1,"b":2,"c":3}
SELECT JSON_REPLACE(@doc, '$.a', 9, '$.c', 3) AS replace_doc;  -- {"a":9,"b":2}
SELECT JSON_REMOVE (@doc, '$.b')              AS remove_doc;   -- {"a":1}
```

## Example 9 : UPDATE with JSON_SET writing back to the column

```sql
-- 10.2+ : in-place mutation of a JSON column.
CREATE TABLE user_profile (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  prefs JSON NOT NULL CHECK (JSON_VALID(prefs))
);

INSERT INTO user_profile (prefs) VALUES ('{"theme":"light","notify":true}');

-- Toggle the theme and keep the rest.
UPDATE user_profile
SET prefs = JSON_SET(prefs, '$.theme', 'dark')
WHERE id = 1;

SELECT prefs FROM user_profile WHERE id = 1;
-- {"theme":"dark","notify":true}
```

## Example 10 : JSON path wildcard and recursive descent

```sql
-- 10.2+ : wildcards.
SET @doc = '{
  "user":{"name":"alice","addr":{"city":"AMS","zip":"1011"}},
  "shop":{"name":"acme","addr":{"city":"NYC","zip":"10001"}}
}';

-- All "name" values at top level :
SELECT JSON_EXTRACT(@doc, '$.*.name');       -- ["alice","acme"]

-- Every "city" anywhere in the tree (recursive descent) :
SELECT JSON_EXTRACT(@doc, '$**.city');       -- ["AMS","NYC"]
-- NB the token is **, NOT .. ; $..city will NOT parse.
```

## Example 11 : Building JSON with JSON_OBJECT and JSON_ARRAY

```sql
-- 10.2+ : construct JSON from columns.
CREATE TABLE customer (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(80) NOT NULL,
  email VARCHAR(120) NOT NULL,
  tags JSON NOT NULL CHECK (JSON_VALID(tags)) DEFAULT '[]'
);

INSERT INTO customer (name, email, tags)
VALUES ('alice', 'a@x', JSON_ARRAY('vip','newsletter'));

SELECT JSON_OBJECT(
  'id',    id,
  'name',  name,
  'email', email,
  'tags',  tags
) AS doc
FROM customer
WHERE id = 1;
-- {"id":1,"name":"alice","email":"a@x","tags":["vip","newsletter"]}
```

## Example 12 : JSON_CONTAINS and JSON_CONTAINS_PATH

```sql
-- 10.2+ : containment and path-existence checks.
SET @doc = '{"role":"admin","perms":["read","write"]}';

-- Does the array contain a specific value at $.perms ?
SELECT JSON_CONTAINS(@doc, '"write"', '$.perms') AS has_write;  -- 1

-- Does the document have a "billing" key anywhere ?
SELECT JSON_CONTAINS_PATH(@doc, 'one', '$.billing') AS has_billing;  -- 0

-- Does the document have BOTH "role" AND "perms" ?
SELECT JSON_CONTAINS_PATH(@doc, 'all', '$.role', '$.perms') AS has_both;  -- 1
```

## Example 13 : Staging-and-promote for legacy LONGTEXT to JSON migration

```sql
-- Pattern for migrating an existing LONGTEXT column to typed JSON.
-- Step 1 : find invalid rows in the source column.
SELECT id, LEFT(raw_json, 80) AS preview
FROM legacy_event
WHERE JSON_VALID(raw_json) = 0;

-- Step 2 : fix or remove invalid rows, then promote the column.
ALTER TABLE legacy_event
  MODIFY COLUMN raw_json JSON NOT NULL CHECK (JSON_VALID(raw_json));

-- Step 3 : add a generated column + index for the hot query path.
ALTER TABLE legacy_event
  ADD COLUMN event_type VARCHAR(64) AS (JSON_VALUE(raw_json, '$.type')) PERSISTENT,
  ADD INDEX ix_event_type (event_type);
```

## Example 14 : JSON_NORMALIZE for canonical hashing

```sql
-- 10.7+ : compute a stable hash for deduplication regardless of key order.
SET @a = '{"a":1,"b":2}';
SET @b = '{"b":2,"a":1}';

SELECT
  SHA2(JSON_NORMALIZE(@a), 256) AS hash_a,
  SHA2(JSON_NORMALIZE(@b), 256) AS hash_b,
  (SHA2(JSON_NORMALIZE(@a), 256) = SHA2(JSON_NORMALIZE(@b), 256)) AS same_doc;
-- same_doc = 1
```
