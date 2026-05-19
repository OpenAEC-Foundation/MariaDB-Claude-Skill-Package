# MariaDB JSON : Methods Reference

Complete JSON_* function reference with signature, return type, version, and KB link. Verified against KB `json-functions/`, `json-data-type/`, `json_value/`, `json_query/`, `json_extract/`, `json_table/`, `json_merge_patch/`, `json_merge_preserve/`, `generated-columns/`, `jsonpath-expressions/` on 2026-05-19.

## 1. JSON data type

```text
column_name JSON [NOT NULL] [DEFAULT '...'] [CHECK (JSON_VALID(column_name))]
```

- `JSON` is an alias for `LONGTEXT COLLATE utf8mb4_bin` (D-010, L-005). Not a native binary type.
- The `JSON` alias auto-wires a hidden `CHECK (JSON_VALID(col))` ; always declare the CHECK explicitly so the schema is self-documenting and survives a dump-and-reload to a `LONGTEXT` column.
- Storage is plain text. Row-based replication between MySQL master and MariaDB replica is NOT supported for JSON columns.
- Comparison semantics on `=` and ordering operators are STRING-wise. For structural equality use `JSON_EQUALS(a, b)`.
- KB : `https://mariadb.com/kb/en/json-data-type/`.

## 2. Extraction functions

| Function | Signature | Returns | Version | KB |
|----------|-----------|---------|---------|-----|
| JSON_VALUE | `JSON_VALUE(doc, path)` | Scalar (unwrapped) | 10.2.3+ | `/kb/en/json_value/` |
| JSON_QUERY | `JSON_QUERY(doc, path)` | JSON sub-tree (object or array) | 10.2.3+ | `/kb/en/json_query/` |
| JSON_EXTRACT | `JSON_EXTRACT(doc, path[, path]*)` | JSON (scalar wrapped or sub-tree) | 10.2+ | `/kb/en/json_extract/` |

Selection rule :

- `WHERE` and `ORDER BY` predicates and generated-column expressions for indexing : `JSON_VALUE`.
- Feeding the result into another JSON_* call : `JSON_QUERY` or `JSON_EXTRACT`.
- Generic extraction where you handle the wrapping yourself : `JSON_EXTRACT`.

## 3. Validation and type functions

| Function | Signature | Returns | Version | KB |
|----------|-----------|---------|---------|-----|
| JSON_VALID | `JSON_VALID(doc)` | Boolean 1 / 0 | 10.0.16+ | `/kb/en/json_valid/` |
| JSON_TYPE | `JSON_TYPE(doc[, path])` | Scalar string (OBJECT, ARRAY, STRING, INTEGER, DOUBLE, BOOLEAN, NULL) | 10.2.3+ | `/kb/en/json_type/` |
| JSON_SCHEMA_VALID | `JSON_SCHEMA_VALID(schema, doc)` | Boolean 1 / 0 | 11.1+ | `/kb/en/json_schema_valid/` |

`JSON_VALID(doc)` is the canonical CHECK-constraint expression. Returns 0 for invalid JSON, never raises an error.

## 4. Mutation functions

| Function | Signature | Returns | Effect | KB |
|----------|-----------|---------|--------|-----|
| JSON_SET | `JSON_SET(doc, path, val [, path, val]*)` | JSON | Insert OR update at path | `/kb/en/json_set/` |
| JSON_INSERT | `JSON_INSERT(doc, path, val [, path, val]*)` | JSON | Insert ONLY if path missing | `/kb/en/json_insert/` |
| JSON_REPLACE | `JSON_REPLACE(doc, path, val [, path, val]*)` | JSON | Replace ONLY if path exists | `/kb/en/json_replace/` |
| JSON_REMOVE | `JSON_REMOVE(doc, path [, path]*)` | JSON | Remove path | `/kb/en/json_remove/` |
| JSON_ARRAY_APPEND | `JSON_ARRAY_APPEND(doc, path, val [, path, val]*)` | JSON | Append to array(s) at path(s) | `/kb/en/json_array_append/` |
| JSON_ARRAY_INSERT | `JSON_ARRAY_INSERT(doc, path, val [, path, val]*)` | JSON | Insert at specific array index | `/kb/en/json_array_insert/` |

All mutation functions return the new document ; the column is updated only when the result is written back via `UPDATE t SET doc = JSON_SET(doc, ...)`.

## 5. Merge functions

| Function | Signature | Semantics | KB |
|----------|-----------|-----------|-----|
| JSON_MERGE_PATCH | `JSON_MERGE_PATCH(a, b [, c]*)` | RFC 7396 ; later values OVERWRITE, JSON `null` REMOVES the key. | `/kb/en/json_merge_patch/` |
| JSON_MERGE_PRESERVE | `JSON_MERGE_PRESERVE(a, b [, c]*)` | Preserve all keys, concatenate arrays. | `/kb/en/json_merge_preserve/` |
| JSON_MERGE | `JSON_MERGE(a, b [, c]*)` | DEPRECATED. Silently aliases `JSON_MERGE_PRESERVE`. NEVER use. | `/kb/en/json_merge/` |

PATCH vs PRESERVE quick check :

```text
JSON_MERGE_PATCH    ('[1,2]', '[2,3]')      -> [2, 3]
JSON_MERGE_PRESERVE ('[1,2]', '[2,3]')      -> [1, 2, 2, 3]

JSON_MERGE_PATCH    ('{"a":1,"b":2}', '{"a":null}')   -> {"b":2}
JSON_MERGE_PRESERVE ('{"a":1,"b":2}', '{"a":null}')   -> {"a":[1,null],"b":2}
```

## 6. Containment and search functions

| Function | Signature | Returns | KB |
|----------|-----------|---------|-----|
| JSON_CONTAINS | `JSON_CONTAINS(target, candidate [, path])` | Boolean | `/kb/en/json_contains/` |
| JSON_CONTAINS_PATH | `JSON_CONTAINS_PATH(doc, one\|all, path [, path]*)` | Boolean | `/kb/en/json_contains_path/` |
| JSON_EXISTS | `JSON_EXISTS(doc, path)` | Boolean | `/kb/en/json_exists/` |
| JSON_EQUALS | `JSON_EQUALS(a, b)` | Boolean (structural) | `/kb/en/json_equals/` |
| JSON_OVERLAPS | `JSON_OVERLAPS(a, b)` | Boolean | 10.9+ ; `/kb/en/json_overlaps/` |
| JSON_SEARCH | `JSON_SEARCH(doc, one\|all, search [, escape [, path]*])` | Path string or NULL | `/kb/en/json_search/` |

`JSON_EQUALS` is the function to use when you need MySQL-style structural equality on MariaDB.

## 7. Structural and length functions

| Function | Signature | Returns | KB |
|----------|-----------|---------|-----|
| JSON_KEYS | `JSON_KEYS(doc [, path])` | JSON array of top-level keys at path | `/kb/en/json_keys/` |
| JSON_LENGTH | `JSON_LENGTH(doc [, path])` | Integer | `/kb/en/json_length/` |
| JSON_DEPTH | `JSON_DEPTH(doc)` | Integer (max nesting) | `/kb/en/json_depth/` |
| JSON_ARRAY | `JSON_ARRAY([val, ...])` | JSON array | `/kb/en/json_array/` |
| JSON_OBJECT | `JSON_OBJECT([key, val [, key, val]*])` | JSON object | `/kb/en/json_object/` |

## 8. String / quoting functions

| Function | Signature | Returns | KB |
|----------|-----------|---------|-----|
| JSON_QUOTE | `JSON_QUOTE(str)` | JSON-quoted string with escapes | `/kb/en/json_quote/` |
| JSON_UNQUOTE | `JSON_UNQUOTE(json_str)` | Raw string with quotes and escapes removed | `/kb/en/json_unquote/` |

`JSON_VALUE` already calls `JSON_UNQUOTE` on its result ; you only need explicit `JSON_UNQUOTE` when starting from `JSON_EXTRACT`.

## 9. Formatting functions

| Function | Signature | Returns | KB |
|----------|-----------|---------|-----|
| JSON_COMPACT | `JSON_COMPACT(doc)` | JSON without whitespace | `/kb/en/json_compact/` |
| JSON_DETAILED | `JSON_DETAILED(doc)` | Pretty-printed JSON | `/kb/en/json_detailed/` |
| JSON_LOOSE | `JSON_LOOSE(doc)` | JSON with spaces around tokens | `/kb/en/json_loose/` |
| JSON_NORMALIZE | `JSON_NORMALIZE(doc)` | Canonical form (keys sorted) | `/kb/en/json_normalize/` |

`JSON_NORMALIZE` is useful when computing a hash over a JSON document : normalise first, then `SHA2`.

## 10. JSON_TABLE (10.6+)

```text
JSON_TABLE(json_doc, context_path COLUMNS (column_spec [, column_spec]*)) [AS] alias

column_spec :
    col_name type PATH path_str [on_empty] [on_error]
  | col_name FOR ORDINALITY
  | col_name type EXISTS PATH path_str
  | NESTED PATH path_str COLUMNS (column_spec [, column_spec]*)

on_empty :
    {NULL | DEFAULT 'literal' | ERROR} ON EMPTY

on_error :
    {NULL | DEFAULT 'literal' | ERROR} ON ERROR
```

- Added in MariaDB 10.6. Verified : KB `json_table/`.
- `FOR ORDINALITY` column starts at 1 per outer-context row.
- `EXISTS PATH` returns 1 if the path exists at the row, 0 otherwise.
- `NESTED PATH` explodes nested arrays without writing a self-join.
- When `ON EMPTY` is omitted the default is `NULL ON EMPTY`. Same for `ON ERROR`.
- `ON ERROR` only fires for JSON structure errors (non-scalar where scalar expected), NOT for datatype conversion errors. A non-numeric string in a numeric column produces `NULL` regardless of `ON ERROR`.

## 11. JSONPath expressions supported

| Token | Meaning | Version |
|-------|---------|---------|
| `$` | Root | all |
| `.key` | Member access (identifier) | all |
| `."name with space"` | Member access (non-identifier) | all |
| `.*` | All members | all |
| `[N]` | Array index (0-based) | all |
| `[*]` | All array elements | all |
| `**` | Recursive descent ; MUST be followed by another step | all |
| `[-N]` | Negative index from end | 10.9+ |
| `[last-N]` | N-th from last | 10.9+ |
| `[M to N]` | Range | 10.9+ |
| `lax $...` | Lax mode prefix (default and only mode) | all |

- `strict` mode and `filter()` predicates are NOT supported on MariaDB.
- Recursive descent uses `**`, NOT `..`. A path of `$..key` will not parse.
- Negative indices and ranges are 10.9+. On 10.6 and 10.11 they do not parse.

KB : `https://mariadb.com/kb/en/jsonpath-expressions/`.

## 12. Generated columns for JSON indexing

```text
column_name type [GENERATED ALWAYS] AS (expression) {VIRTUAL | PERSISTENT | STORED}
```

- VIRTUAL : computed on read, zero disk overhead.
- PERSISTENT (alias STORED) : materialised on insert / update.
- Both can be indexed.
- The canonical JSON-index pattern :

```sql
ALTER TABLE t
  ADD COLUMN status VARCHAR(32)
    AS (JSON_VALUE(doc, '$.status')) PERSISTENT;
CREATE INDEX ix_t_status ON t (status);
```

- DIRECT expression-indexes `CREATE INDEX ix ON t ((JSON_VALUE(doc,'$.k')))` (MySQL 8 style) do NOT parse on MariaDB.
- KB : `https://mariadb.com/kb/en/generated-columns/`.

## 13. System variables relevant to JSON

| Variable | Default | Effect |
|----------|---------|--------|
| character_set_server | latin1 on stock 10.6 / 10.11 | Server-level default ; ALWAYS override to `utf8mb4` for any JSON workload. |
| collation_server | latin1_swedish_ci | Override to `utf8mb4_uca1400_ai_ci` (11.x) or `utf8mb4_0900_ai_ci`-compatible collation. |
| sql_mode | varies | Enable `STRICT_TRANS_TABLES` so out-of-range scalars in `JSON_VALUE` casts raise errors instead of silently truncating. |

## 14. Inspection queries

```sql
-- Show all JSON-typed (aliased) columns.
SELECT table_schema, table_name, column_name, column_type
FROM information_schema.columns
WHERE data_type IN ('longtext','json')
  AND (column_type = 'json' OR column_comment LIKE '%json%')
ORDER BY 1, 2, 3;

-- Find rows that fail JSON_VALID in a column declared as LONGTEXT.
SELECT id FROM legacy_blob_table WHERE JSON_VALID(payload) = 0;

-- Inspect generated-column definitions (catch missing JSON-indexes).
SELECT table_schema, table_name, column_name, generation_expression
FROM information_schema.columns
WHERE generation_expression LIKE '%JSON_%';
```
