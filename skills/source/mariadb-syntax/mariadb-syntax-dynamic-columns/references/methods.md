# MariaDB Dynamic Columns : function reference

Complete signatures, return types, type tags, and version notes for every COLUMN_* function. All entries verified against the official MariaDB Knowledge Base.

## Function summary table

| Function | Returns | Signature | Min version |
|----------|---------|-----------|-------------|
| `COLUMN_CREATE` | dynamic-column blob (BLOB) | `COLUMN_CREATE(name1, value1 [AS type], name2, value2 [AS type], ...)` | 5.3 (named since 10.0) |
| `COLUMN_ADD` | dynamic-column blob (BLOB) | `COLUMN_ADD(blob, name1, value1 [AS type], ...)` | 5.3 (named since 10.0) |
| `COLUMN_GET` | typed scalar or NULL | `COLUMN_GET(blob, name AS type)` | 5.3 (named since 10.0) |
| `COLUMN_DELETE` | dynamic-column blob (BLOB) | `COLUMN_DELETE(blob, name1 [, name2, ...])` | 5.3 |
| `COLUMN_EXISTS` | INT (1 or 0) | `COLUMN_EXISTS(blob, name)` | 5.3 |
| `COLUMN_LIST` | VARCHAR (backtick-quoted, comma-separated names) | `COLUMN_LIST(blob)` | 5.3 |
| `COLUMN_CHECK` | INT (1 valid, 0 invalid) | `COLUMN_CHECK(blob)` | 10.0 |
| `COLUMN_JSON` | JSON document as TEXT | `COLUMN_JSON(blob)` | 10.0 |

Because this skill targets 10.6-LTS, 10.11-LTS, 11.x, and 12.x, ALL eight functions are ALWAYS available.

## Type tags supported by COLUMN_CREATE, COLUMN_ADD, and COLUMN_GET

The `AS type` clause is OPTIONAL on write and REQUIRED on read. The accepted tags are :

| Tag | Stored as | Notes |
|-----|-----------|-------|
| `BINARY` or `BINARY(N)` | Raw bytes | Default for unknown values ; also used for nested dynamic-column reads. |
| `CHAR` or `CHAR(N)` | UTF-8 text | Use for string keys. `CHAR` without `(N)` defaults to `CHAR(65535)`. |
| `INTEGER` or `SIGNED [INTEGER]` | Signed 64-bit | Range `-2^63 .. 2^63-1`. |
| `UNSIGNED [INTEGER]` | Unsigned 64-bit | Range `0 .. 2^64-1`. |
| `DECIMAL` or `DECIMAL(M,D)` | Fixed-point | Exact arithmetic ; use for money. |
| `DOUBLE` | IEEE 754 binary64 | Inexact ; do NOT use for money. |
| `DATE` | `YYYY-MM-DD` | No time component. |
| `TIME` | `HH:MM:SS[.ffffff]` | Stored to microsecond precision. |
| `DATETIME` | `YYYY-MM-DD HH:MM:SS[.ffffff]` | No time-zone metadata. |

There is NO `JSON` type tag and NO `BOOLEAN` type tag. Booleans are conventionally stored as `INTEGER` 0 or 1, or `CHAR(1)` 'Y' or 'N'.

## COLUMN_CREATE : detail

```text
COLUMN_CREATE(
  name1, value1 [AS type] ,
  name2, value2 [AS type] ,
  ...
) -> BLOB
```

- Returns a binary string that holds the named key-value pairs.
- Inputs alternate name, value, name, value.
- The `AS type` clause is OPTIONAL when the value's SQL type is already the desired stored type.
- Passing `NULL` as a value writes the key with a NULL value (key still exists ; `COLUMN_EXISTS` returns 1).
- Max keys per blob : 65 535.
- Max blob size : bound by `max_allowed_packet`.

## COLUMN_ADD : detail

```text
COLUMN_ADD(
  blob,
  name1, value1 [AS type] ,
  name2, value2 [AS type] ,
  ...
) -> BLOB
```

- Returns a NEW blob with the listed keys added or overwritten.
- Passing `NULL` as a value DELETES the key (this is the only way `COLUMN_ADD` removes keys).
- An empty input blob is treated like `COLUMN_CREATE()`.

## COLUMN_GET : detail

```text
COLUMN_GET(blob, name AS type) -> typed scalar OR NULL
```

- The `AS type` clause is MANDATORY. The MariaDB SQL parser will reject `COLUMN_GET(blob, 'k')` without a type.
- Returns NULL when :
  - The blob itself is NULL.
  - The key does not exist.
  - The blob is corrupt.
- Type coercion is performed when the stored type and the requested tag differ. The coercion follows standard MariaDB conversion rules. ALWAYS match the stored type to avoid lossy conversion (for example, a stored DECIMAL retrieved as CHAR loses arithmetic semantics).

## COLUMN_DELETE : detail

```text
COLUMN_DELETE(blob, name1 [, name2, ...]) -> BLOB
```

- Returns a NEW blob with the listed keys removed.
- Deleting a non-existent key is silent (no warning, no error).
- Multiple keys may be removed in a single call.

## COLUMN_EXISTS : detail

```text
COLUMN_EXISTS(blob, name) -> 1 (exists) or 0 (absent)
```

- Returns 1 if the named key exists, 0 otherwise.
- Returns 0 when the blob is NULL.
- A key whose value is NULL still returns 1 ("present with NULL value").

## COLUMN_LIST : detail

```text
COLUMN_LIST(blob) -> VARCHAR
```

- Returns a comma-separated list of backtick-quoted column names.
- Example return : `` `color`,`size`,`stock` ``.
- The result is intended for display and debug, NOT for programmatic parsing in SQL.
- Order of names is implementation-defined and SHOULD NOT be relied on.

## COLUMN_CHECK : detail

```text
COLUMN_CHECK(blob) -> 1 (well-formed) or 0 (corrupt)
```

- Use in a `CHECK` constraint to refuse malformed writes :

```sql
CONSTRAINT chk_attrs CHECK (attrs IS NULL OR COLUMN_CHECK(attrs) = 1)
```

- Returns 1 for an empty blob produced by `COLUMN_CREATE()` with no arguments.
- Returns 0 for any random byte string that is not a valid dynamic-column blob.

## COLUMN_JSON : detail

```text
COLUMN_JSON(blob) -> JSON document as TEXT
```

- Converts the dynamic-column blob to a standards-compliant JSON document.
- Nested dynamic-column values become nested JSON objects.
- BINARY values are base64-encoded.
- DATE / TIME / DATETIME values become ISO-8601 strings.
- The inverse direction (JSON to dynamic columns) does NOT have a single-function shortcut ; build the blob in application code via repeated `COLUMN_CREATE` and `COLUMN_ADD` calls.

## Storage layout

- Dynamic columns live inside an ordinary `BLOB`, `MEDIUMBLOB`, or `LONGBLOB` column.
- There is no dedicated SQL type. `CREATE TABLE` does NOT mention "dynamic columns".
- The blob is a self-describing binary format : each key has a name (or number), a type tag, and a value.
- Two storage modes exist : NAMED (preferred, 10.0+) and NUMBERED (pre-10.0 legacy). The two modes are NOT mixable in a single blob.

## Limits

| Constraint | Value |
|------------|-------|
| Max keys per blob | 65 535 |
| Max blob size | bounded by `max_allowed_packet` (default 1 GB on modern releases) |
| Max nesting depth | 10 levels |
| Key-name encoding | utf8mb4 |

## Indexing

- Dynamic columns CANNOT be indexed directly. The blob is opaque to the optimiser.
- ALWAYS expose the indexable key through a `VIRTUAL` or `PERSISTENT` generated column :

```sql
ALTER TABLE t
  ADD COLUMN k_color VARCHAR(16)
    AS (COLUMN_GET(attrs, 'color' AS CHAR(16))) VIRTUAL,
  ADD INDEX ix_k_color (k_color);
```

- The expression in the generated column must match the COLUMN_GET signature in WHERE clauses for the optimiser to recognise it.

## Version availability matrix

| Function | 10.6-LTS | 10.11-LTS | 11.x | 12.x |
|----------|---------|-----------|------|------|
| COLUMN_CREATE | yes | yes | yes | yes |
| COLUMN_ADD | yes | yes | yes | yes |
| COLUMN_GET | yes | yes | yes | yes |
| COLUMN_DELETE | yes | yes | yes | yes |
| COLUMN_EXISTS | yes | yes | yes | yes |
| COLUMN_LIST | yes | yes | yes | yes |
| COLUMN_CHECK | yes | yes | yes | yes |
| COLUMN_JSON | yes | yes | yes | yes |

No deprecations were announced through 12.x at the time of skill creation. Re-verify before each release-cycle update.

## MySQL portability

- Dynamic columns DO NOT EXIST in MySQL at any version.
- Migrating to MySQL REQUIRES converting the blob to JSON (or another supported representation) BEFORE export.
- Use `SELECT id, COLUMN_JSON(attrs) FROM t` to produce a portable dump.

## KB references

- `https://mariadb.com/kb/en/dynamic-columns/`
- `https://mariadb.com/kb/en/dynamic-columns-functions/`
- `https://mariadb.com/kb/en/column_create/`
- `https://mariadb.com/kb/en/column_add/`
- `https://mariadb.com/kb/en/column_get/`
- `https://mariadb.com/kb/en/column_delete/`
- `https://mariadb.com/kb/en/column_exists/`
- `https://mariadb.com/kb/en/column_list/`
- `https://mariadb.com/kb/en/column_check/`
- `https://mariadb.com/kb/en/column_json/`
- `https://mariadb.com/kb/en/generated-columns/`
