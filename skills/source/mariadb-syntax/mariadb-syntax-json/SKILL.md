---
name: mariadb-syntax-json
description: >
  Use when working with JSON columns in MariaDB, comparing MariaDB JSON to MySQL JSON behaviour, indexing JSON paths, validating JSON structure, or extracting / setting / merging JSON values.
  Prevents the common mistake of assuming MariaDB JSON is binary like MySQL 5.7.8+, forgetting CHECK (JSON_VALID(col)) so the column accepts arbitrary text, using deprecated JSON_MERGE instead of the explicit PATCH or PRESERVE variants, indexing a JSON path with a direct expression instead of a generated column, and confusing JSON_VALUE (scalar) with JSON_EXTRACT or JSON_QUERY (JSON sub-tree).
  Covers the JSON-as-LONGTEXT alias (D-010), JSON_VALID, JSON_EXTRACT, JSON_VALUE, JSON_QUERY, JSON_TABLE (10.6+), JSON_SET, JSON_INSERT, JSON_REPLACE, JSON_REMOVE, JSON_MERGE_PATCH (RFC 7396) versus JSON_MERGE_PRESERVE, JSON path expressions, CHECK (JSON_VALID(col)) validation, and functional indexing on JSON via virtual or persistent generated columns.
  Keywords: JSON, JSON_VALID, JSON_EXTRACT, JSON_VALUE, JSON_QUERY, JSON_TABLE, JSON_SET, JSON_INSERT, JSON_REPLACE, JSON_REMOVE, JSON_MERGE_PATCH, JSON_MERGE_PRESERVE, JSON_MERGE deprecated, JSON_CONTAINS, JSON_KEYS, JSON_LENGTH, JSON_TYPE, JSON_QUOTE, JSON_UNQUOTE, JSON path, dollar path, CHECK constraint, JSON is LONGTEXT, JSON not binary, MariaDB JSON alias, functional index JSON, virtual column JSON, persistent column JSON, generated column JSON, NESTED PATH, FOR ORDINALITY, EXISTS PATH, ON EMPTY, ON ERROR, RFC 7396, how do I index JSON, why is my JSON query slow, my JSON column accepts invalid text, JSON_EXTRACT returns wrapped, MySQL JSON migration, MariaDB JSON divergence, jsonb missing
license: MIT
compatibility: "Designed for Claude Code. Requires MariaDB 10.6-LTS, 10.11-LTS, 11.x, 12.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# MariaDB JSON : Type, Functions, Indexing, Validation

Deterministic guidance for working with the JSON type and JSON_* functions in MariaDB. Use this skill to declare valid JSON columns, choose the correct extraction function, index JSON paths through generated columns, merge documents with the correct semantics, and avoid the most damaging MySQL-vs-MariaDB JSON traps.

## Quick Reference

- **MariaDB JSON is an ALIAS for LONGTEXT, NOT native binary storage like MySQL 5.7.8+.** Use `CHECK (JSON_VALID(col))` for structural validation. For indexing, use functional indexes on virtual or persistent generated columns ; direct expression-indexes are not supported. This is the single most consequential MySQL divergence (D-010, L-005). Verified : KB `json-data-type/`.
- **ALWAYS add `CHECK (JSON_VALID(col))` on every JSON column.** Without the CHECK constraint, `INSERT INTO t (j) VALUES ('not even json')` succeeds silently and breaks every downstream `JSON_EXTRACT`. The KB confirms : "the JSON_VALID function can be used as a CHECK constraint" ; from the JSON alias the constraint is wired automatically only when the column type is `JSON`, not when it is `LONGTEXT`.
- **JSON_VALUE returns a SCALAR ; JSON_QUERY returns a JSON sub-tree ; JSON_EXTRACT returns JSON (scalar wrapped or sub-tree).** Pick by what you actually need downstream. ALWAYS use JSON_VALUE in `WHERE`-clauses and functional indexes : it strips the JSON quoting so `WHERE JSON_VALUE(doc,'$.status') = 'open'` compares strings, not JSON-quoted strings. Verified : KB `json_value/`, `json_query/`, `json_extract/`.
- **Use JSON_MERGE_PATCH for RFC 7396 semantics ; use JSON_MERGE_PRESERVE for array concatenation. NEVER use JSON_MERGE.** `JSON_MERGE` is the deprecated alias and silently behaves like `JSON_MERGE_PRESERVE`, not like `JSON_MERGE_PATCH`. Migrations from MySQL 5.7 code that called `JSON_MERGE` for partial-update semantics produce wrong results on MariaDB unless rewritten. Verified : KB `json_merge_patch/`, `json_merge_preserve/`.
- **JSON_MERGE_PATCH (RFC 7396) : a key whose patch value is JSON `null` is REMOVED from the result.** Use this for "PATCH /resource"-style partial updates. Use `JSON_REMOVE` only when you want to drop the key by path without a patch document.
- **Functional indexes on JSON require a virtual or persistent generated column.** ALWAYS materialise the expression first, then index the column. Direct `CREATE INDEX ix ON t ((JSON_VALUE(doc,'$.k')))` syntax (as in MySQL 8 or PostgreSQL) does NOT parse on MariaDB. Verified : KB `generated-columns/`.
- **JSON_TABLE (10.6+) converts JSON arrays to relational rows** with `NESTED PATH` for hierarchies, `FOR ORDINALITY` for row counters, and `EXISTS PATH` for boolean existence checks. `ON EMPTY` and `ON ERROR` accept `NULL | DEFAULT 'literal' | ERROR` ; both default to `NULL` when omitted. Verified : KB `json_table/`.
- **JSON path uses `$` for root, `.key` for member, `[N]` for index, `[*]` for all array elements, and `**` for recursive descent.** The recursive descent token is `**`, NOT `..` (PostgreSQL / jq syntax) and `**` must be followed by another step (never the final token). Negative indices `[-N]` and ranges `[M to N]` are 10.9+. Verified : KB `jsonpath-expressions/`.
- **JSON column collation is `utf8mb4_bin` and storage is `LONGTEXT`.** Inserts in any other character set MUST be transcoded by the client to utf8mb4 ; an inbound `latin1` JSON document with non-ASCII bytes will be mangled. ALWAYS run `SET NAMES utf8mb4` on the connection.
- **Row-based replication does NOT work for JSON columns between MySQL master and MariaDB replica.** MySQL ships the binary format on the wire ; MariaDB cannot decode it. Use statement-based replication or convert the column to TEXT before replication. Verified : KB `json-data-type/`.
- **JSON comparison semantics differ from MySQL.** MariaDB compares JSON columns as strings ; MySQL compares structurally. Two JSON documents that are semantically equal but differ in whitespace or key-order compare as equal in MySQL and unequal in MariaDB. ALWAYS use `JSON_EQUALS(a, b)` for structural equality on MariaDB.

## Decision Trees

### Tree 1 : "Which extraction function do I need?"

```
What do I want to extract from the JSON document?
  A single scalar value (string, number, bool) to compare or index ?
    -> JSON_VALUE(doc, '$.path'). Returns unwrapped scalar. Use in WHERE,
       ORDER BY, and as the expression in a generated column for indexing.
  A nested object or array to feed to another JSON function ?
    -> JSON_QUERY(doc, '$.path'). Returns JSON sub-tree, never a scalar.
  Either, and I will handle wrapping downstream ?
    -> JSON_EXTRACT(doc, '$.path'). Returns JSON ; scalars come back
       JSON-quoted ("open"), which is rarely what WHERE-clauses want.
```

### Tree 2 : "How do I merge two JSON documents?"

```
What is the intent of the merge?
  Partial update : "apply these changes, NULL removes a key" (PATCH /resource) ?
    -> JSON_MERGE_PATCH(target, patch). RFC 7396 semantics.
       Patch '{"a":null}' on '{"a":1,"b":2}' yields '{"b":2}'.
  Combine two documents, preserve all keys and concatenate arrays ?
    -> JSON_MERGE_PRESERVE(a, b). PRESERVE preserves duplicates.
       Merge of [1,2] and [2,3] yields [1,2,2,3].
  Legacy code calls JSON_MERGE(a, b) ?
    -> REWRITE. JSON_MERGE is deprecated and silently aliases
       JSON_MERGE_PRESERVE. MySQL 5.7 code that expected PATCH semantics
       must be migrated to JSON_MERGE_PATCH.
```

### Tree 3 : "How do I index a JSON path?"

```
The query filters on a stable JSON key path ?
  Yes :
    Step 1 : Add a generated column over the path.
      ALTER TABLE t
        ADD COLUMN status VARCHAR(32)
          AS (JSON_VALUE(doc, '$.status')) PERSISTENT;
      VIRTUAL : zero disk, computed on read, still indexable.
      PERSISTENT : materialised, faster reads, costs disk + write CPU.
    Step 2 : Index the column.
      CREATE INDEX ix_t_status ON t (status);
    Step 3 : Verify with EXPLAIN that the query uses the index.

  No, JSON keys are dynamic ?
    -> Cannot be efficiently indexed. Consider Dynamic Columns,
       a key-value table, or schema redesign.
```

### Tree 4 : "Do I store this as JSON or normalise it?"

```
Is the document structure stable, queried often, and joined to other tables ?
  Yes -> Normalise. Columns and rows. Indexes work as designed.
Is the document a flexible attribute bag with sparse, varied keys ?
  Yes -> JSON with CHECK (JSON_VALID(col)). Add functional indexes for
         the few paths you query often.
Is the document a black-box payload that you store and replay ?
  Yes -> LONGTEXT or JSON. Whether to use JSON depends on whether
         downstream code needs JSON_* functions.
Are you considering Dynamic Columns instead ?
  See mariadb-syntax-dynamic-columns. Default to JSON for new schemas
  because of standards compliance and JSONPath ergonomics.
```

## Patterns

### Pattern 1 : JSON column with validation

```sql
-- 10.2+ : JSON alias auto-wires CHECK (JSON_VALID(payload)),
-- but ALWAYS write it explicitly so the schema is self-documenting
-- and survives a dump-and-reload to a LONGTEXT column.
CREATE TABLE event (
  id       BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  payload  JSON NOT NULL CHECK (JSON_VALID(payload)),
  created_at DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
) ENGINE=InnoDB;

-- The CHECK constraint rejects invalid JSON :
INSERT INTO event (payload) VALUES ('not json');  -- ERROR 4025 : CONSTRAINT failed
INSERT INTO event (payload) VALUES ('{"type":"ok"}');  -- accepted.
```

### Pattern 2 : JSON_VALUE versus JSON_QUERY versus JSON_EXTRACT

```sql
-- Sample document.
SET @doc = '{"status":"open","tags":["red","blue"]}';

-- JSON_VALUE : unwrapped scalar. Use this for comparisons and indexes.
SELECT JSON_VALUE(@doc, '$.status');        -- open

-- JSON_QUERY : JSON sub-tree. Use this for nested objects or arrays.
SELECT JSON_QUERY(@doc, '$.tags');          -- ["red", "blue"]

-- JSON_EXTRACT : JSON in all cases ; scalars come back JSON-quoted.
SELECT JSON_EXTRACT(@doc, '$.status');      -- "open"  (with quotes)

-- The trap : comparing JSON_EXTRACT to an unquoted string never matches.
SELECT 1 FROM dual WHERE JSON_EXTRACT(@doc,'$.status') = 'open';   -- 0 rows
SELECT 1 FROM dual WHERE JSON_VALUE  (@doc,'$.status') = 'open';   -- 1 row
```

### Pattern 3 : Functional index on a JSON path (D-010)

```sql
-- 10.6+ : MariaDB JSON is LONGTEXT, so a direct expression-index
-- is NOT supported. Materialise the path in a generated column first.
CREATE TABLE event (
  id       BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  payload  JSON NOT NULL CHECK (JSON_VALID(payload)),
  -- VIRTUAL : computed on read, zero disk, still indexable.
  event_type VARCHAR(64) AS (JSON_VALUE(payload, '$.type')) VIRTUAL,
  INDEX ix_event_type (event_type)
) ENGINE=InnoDB;

-- The query that uses the index :
SELECT id FROM event WHERE event_type = 'order.created';

-- Verify with EXPLAIN : expect type=ref, key=ix_event_type.
EXPLAIN SELECT id FROM event WHERE event_type = 'order.created';
```

### Pattern 4 : JSON_MERGE_PATCH for partial-update API

```sql
-- 10.2+ : RFC 7396 PATCH semantics. NULL in patch removes a key.
SET @current = '{"name":"alice","email":"a@x","verified":true}';
SET @patch   = '{"email":"alice@x","verified":null}';

SELECT JSON_MERGE_PATCH(@current, @patch);
-- {"name":"alice","email":"alice@x"}
-- "verified" was removed because the patch value was JSON null.
```

### Pattern 5 : JSON_MERGE_PRESERVE for array concatenation

```sql
-- 10.2+ : preserves all keys, concatenates arrays.
SELECT JSON_MERGE_PRESERVE('[1,2]', '[2,3]');         -- [1, 2, 2, 3]
SELECT JSON_MERGE_PRESERVE('{"k":1}', '{"k":2}');     -- {"k":[1,2]}

-- NEVER use the deprecated JSON_MERGE :
-- it silently aliases JSON_MERGE_PRESERVE and is removed from
-- the SQL standard road map. Migrating MySQL code that expected
-- PATCH semantics from JSON_MERGE will silently break.
```

### Pattern 6 : JSON_SET, JSON_INSERT, JSON_REPLACE, JSON_REMOVE

```sql
-- JSON_SET   : insert OR update.
SELECT JSON_SET('{"a":1}', '$.a', 9, '$.b', 2);     -- {"a":9,"b":2}

-- JSON_INSERT : only if path is missing.
SELECT JSON_INSERT('{"a":1}', '$.a', 9, '$.b', 2);  -- {"a":1,"b":2}

-- JSON_REPLACE : only if path exists.
SELECT JSON_REPLACE('{"a":1}', '$.a', 9, '$.b', 2); -- {"a":9}

-- JSON_REMOVE : drop a path.
SELECT JSON_REMOVE('{"a":1,"b":2}', '$.b');         -- {"a":1}
```

### Pattern 7 : JSON_TABLE to project an array to rows (10.6+)

```sql
-- 10.6+ : pivot a JSON array into relational rows with ordinality.
SET @doc = '[
  {"name":"Laptop","color":"black","price":1000},
  {"name":"Jeans","color":"blue"}
]';

SELECT id, name, color, price
FROM JSON_TABLE(@doc, '$[*]' COLUMNS (
  id    FOR ORDINALITY,
  name  VARCHAR(40) PATH '$.name',
  color VARCHAR(20) PATH '$.color',
  price DECIMAL(10,2) PATH '$.price' DEFAULT '0' ON EMPTY
)) AS jt;
-- 1  Laptop  black  1000.00
-- 2  Jeans   blue      0.00
```

### Pattern 8 : NESTED PATH for hierarchies

```sql
-- 10.6+ : explode nested arrays with NESTED PATH.
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

### Pattern 9 : JSON path expressions cheat-sheet

```sql
-- $              : root.
-- $.key          : member access.
-- $."with space" : non-identifier member.
-- $[0]           : first array element.
-- $[*]           : every array element.
-- $.a.b.c        : nested.
-- $**.price      : recursive descent (** must be followed by step).
-- $[-1]          : last element (10.9+).
-- $[0 to 2]      : range (10.9+).

SELECT JSON_VALUE('{"a":{"b":{"c":42}}}', '$.a.b.c');     -- 42
SELECT JSON_QUERY('[[1,2],[3,4]]', '$[1]');               -- [3, 4]
```

### Pattern 10 : Detecting and rejecting invalid JSON on import

```sql
-- Staging pattern : import as LONGTEXT, validate, promote.
CREATE TABLE event_staging (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  raw LONGTEXT NOT NULL
) ENGINE=InnoDB;

-- Find rows that fail validation BEFORE moving to the JSON-typed table.
SELECT id, LEFT(raw, 80) AS preview
FROM event_staging
WHERE JSON_VALID(raw) = 0;

-- Then move the validated rows into the typed table.
INSERT INTO event (payload)
SELECT raw FROM event_staging WHERE JSON_VALID(raw) = 1;
```

## Reference Links

- `references/methods.md` : full JSON_* function reference with signature, version, return-type, KB link.
- `references/examples.md` : 12+ working JSON examples covering DDL, extraction, mutation, merging, projection, indexing.
- `references/anti-patterns.md` : 8+ real-world JSON mistakes on MariaDB with the fix.

## External References

- MariaDB KB : `https://mariadb.com/kb/en/json-data-type/`
- MariaDB KB : `https://mariadb.com/kb/en/json-functions/`
- MariaDB KB : `https://mariadb.com/kb/en/json_value/`
- MariaDB KB : `https://mariadb.com/kb/en/json_query/`
- MariaDB KB : `https://mariadb.com/kb/en/json_extract/`
- MariaDB KB : `https://mariadb.com/kb/en/json_table/`
- MariaDB KB : `https://mariadb.com/kb/en/json_merge_patch/`
- MariaDB KB : `https://mariadb.com/kb/en/json_merge_preserve/`
- MariaDB KB : `https://mariadb.com/kb/en/jsonpath-expressions/`
- MariaDB KB : `https://mariadb.com/kb/en/generated-columns/`
