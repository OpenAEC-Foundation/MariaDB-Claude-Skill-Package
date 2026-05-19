# methods.md : Encoding and Collation Reference

Complete reference for MariaDB character sets, collations, error codes, connection variables, and the index-prefix byte math behind `CONVERT TO CHARACTER SET`.

## 1. Charset / Collation Matrix Per Version

### Unicode charsets

| Charset | Max bytes/char | Stores emoji and code points > U+FFFF | Notes |
|---------|----------------|----------------------------------------|-------|
| `utf8` | 3 | NO | Alias for `utf8mb3` (alias target controlled by `old_mode`) |
| `utf8mb3` | 3 | NO | Canonical name from 10.6.1; Basic Multilingual Plane only |
| `utf8mb4` | 4 | YES | Full UTF-8; ALWAYS use this for user-generated text |
| `ucs2` | 2 | NO | Fixed-width UCS-2, BMP only, legacy |
| `utf16` | 4 | YES | Fixed-or-variable width, legacy, rarely used |
| `utf32` | 4 | YES | Fixed-width 4 bytes, legacy |

### Default utf8mb4 collation per MariaDB version

| Version | Default utf8mb4 collation |
|---------|----------------------------|
| 10.6-LTS | `utf8mb4_general_ci` |
| 10.11-LTS | `utf8mb4_general_ci` |
| 11.0 - 11.4 | `utf8mb4_general_ci` |
| 11.5 | `utf8mb4_uca1400_ai_ci` |
| 11.6-LTS | `utf8mb4_uca1400_ai_ci` |
| 12.x | `utf8mb4_uca1400_ai_ci` |

Source: `mariadb.com/kb/en/supported-character-sets-and-collations/`. The default for the `utf8mb4` charset became `utf8mb4_uca1400_ai_ci` in MariaDB 11.5.

### Common utf8mb4 collations

| Collation | Algorithm | Case | Accent | Use |
|-----------|-----------|------|--------|-----|
| `utf8mb4_general_ci` | legacy fast | insensitive | partly insensitive | legacy default; fast but imperfect ordering |
| `utf8mb4_unicode_ci` | UCA 4.0.0 | insensitive | insensitive | older Unicode-correct option |
| `utf8mb4_uca1400_ai_ci` | UCA 14.0.0 | insensitive | insensitive | modern default (11.5+) |
| `utf8mb4_uca1400_as_ci` | UCA 14.0.0 | insensitive | sensitive | accent-distinguishing search |
| `utf8mb4_uca1400_as_cs` | UCA 14.0.0 | sensitive | sensitive | fully case+accent sensitive |
| `utf8mb4_bin` | byte | sensitive | sensitive | exact-match technical identifiers only |

MariaDB 10.10 added 184 UCA-14.0.0 collations (`utf8mb4_uca1400_*`). They support language-specific contractions and fix sort anomalies in `utf8mb4_general_ci`.

## 2. Collation Suffix Semantics

A collation name is `<charset>_<rules>_<flags>`. The trailing flags:

| Flag | Name | Semantics |
|------|------|-----------|
| `_ci` | case-insensitive | `'A' = 'a'` TRUE |
| `_cs` | case-sensitive | `'A' = 'a'` FALSE |
| `_bin` | binary | raw byte compare; no case/accent folding, no Unicode awareness |
| `_ai` | accent-insensitive (10.10+) | base letter equals accented letter |
| `_as` | accent-sensitive (10.10+) | accented letter distinct from base |
| `_nopad` | no-pad (10.10+) | trailing spaces are significant; without it, `PAD SPACE` behavior ignores trailing spaces in comparison |

Older collations use a 2-part flag (`_ci` / `_cs` / `_bin`). UCA-1400 collations use a 3-part flag combining accent and case (`_ai_ci`, `_as_ci`, `_as_cs`).

`_bin` is NOT a substitute for `_cs`. `_bin` compares encoded bytes directly and has no notion of character equivalence, so for multi-byte charsets it can order characters differently from a true case-sensitive Unicode collation.

## 3. Error-Code Reference

| Error | Symbol | Trigger | Root cause | Fix |
|-------|--------|---------|------------|-----|
| 1366 | `ER_TRUNCATED_WRONG_VALUE_FOR_FIELD` | `Incorrect string value: '\xF0\x9F...' for column` on INSERT/UPDATE | A 4-byte UTF-8 character sent to a `utf8mb3` or `latin1` column | Convert the column to `utf8mb4` |
| 1267 | cannot-aggregate | `Illegal mix of collations (X,...) and (Y,...) for operation '='` | Two operands with different collations and equal coercibility | `COLLATE` clause in the query, or ALTER columns to one collation |
| 1271 | `ER_CANT_AGGREGATE_NCOLLATIONS` | `Illegal mix of collations for operation 'UNION'` (3+ operands) | Three or more operands cannot be aggregated to a common collation | Align all column collations |
| 1071 | `ER_TOO_LONG_KEY` | `Specified key was too long; max key length is 767 bytes` | Indexed column prefix exceeds row-format limit | `ROW_FORMAT=DYNAMIC`, shorten column, or use index prefix |
| 1253 | `ER_COLLATION_CHARSET_MISMATCH` | `COLLATION 'x' is not valid for CHARACTER SET 'y'` | Collation does not belong to the named charset | Pick a collation whose prefix matches the charset |

Mojibake and `?`-substitution produce NO error. They are silent corruption from a connection-charset mismatch.

## 4. Coercibility

Coercibility ranks how readily an expression's collation yields to another in a conflict. Lower number wins.

| Level | Coercibility | Example |
|-------|--------------|---------|
| 0 | Explicit | a value with an explicit `COLLATE` clause |
| 1 | No collation | concatenation of strings with different collations (already an error) |
| 2 | Implicit | a column value |
| 3 | (system constant) | system functions like `USER()` |
| 4 | Coercible | a string literal |
| 5 | Numeric | a numeric or temporal value |
| 6 | Ignorable | `NULL` or an expression derived from `NULL` |

Error 1267 fires when two operands have EQUAL coercibility and DIFFERENT collations. A literal (level 4) compared to a column (level 2) takes the column collation, no error. Two columns (both level 2) with different collations cannot resolve, so error 1267 fires. An explicit `COLLATE` (level 0) always wins.

Source: `mariadb.com/kb/en/coercibility/`.

## 5. Connection-Charset Variables

| Variable | Scope | Role |
|----------|-------|------|
| `character_set_client` | Global, Session | charset of incoming query text and unintroduced string literals |
| `character_set_connection` | Global, Session | charset literals and comparisons are converted to inside the server |
| `character_set_results` | Global, Session | charset of result-set data sent back to the client |
| `character_set_database` | Global, Session | charset of the default database; set BY the server, do not set manually |
| `character_set_server` | Global | server-wide default for new databases |
| `collation_connection` | Global, Session | collation paired with `character_set_connection` |
| `collation_server` | Global | server-wide default collation |

`SET NAMES x` is equivalent to setting `character_set_client`, `character_set_connection`, and `character_set_results` all to `x`, and `collation_connection` to the default collation of `x`.

```sql
-- 10.6+ : these two are equivalent
SET NAMES utf8mb4;

SET character_set_client     = utf8mb4,
    character_set_connection = utf8mb4,
    character_set_results    = utf8mb4;
```

`SET NAMES utf8mb4 COLLATE utf8mb4_uca1400_ai_ci` additionally pins `collation_connection`.

## 6. Per-Driver Connection Charset

ALWAYS set the charset in the driver, not via a manual `SET NAMES` after connect (a reconnect would silently drop it).

| Driver | Setting |
|--------|---------|
| MariaDB Connector/C, Connector/Python | `charset='utf8mb4'` connection argument |
| JDBC (MariaDB Connector/J) | `?characterEncoding=utf8mb4` in the JDBC URL |
| PHP PDO | `charset=utf8mb4` in the DSN: `mysql:host=...;dbname=...;charset=utf8mb4` |
| PHP mysqli | `mysqli_set_charset($link, 'utf8mb4')` |
| Node.js mysql2 | `charset: 'utf8mb4'` in the connection config |
| Go go-sql-driver/mysql | `?charset=utf8mb4` or `?collation=utf8mb4_uca1400_ai_ci` in the DSN |

The server option `character-set-client-handshake = FALSE` in `my.cnf` forces all connections to honor `character-set-server` and ignore the client's handshake charset request. This is the Frappe/ERPNext hardening pattern.

## 7. Charset Hierarchy (Cascade)

Charset and collation cascade at object-creation time:

```
column  ->  table  ->  database  ->  server (character-set-server)
```

A column with no explicit charset inherits the table's `DEFAULT CHARSET`; the table inherits the database charset; the database inherits `character_set_server`.

CRITICAL: the cascade is evaluated ONCE, at CREATE time. Changing `character-set-server` or running `ALTER DATABASE ... CHARACTER SET` does NOT retroactively re-encode existing tables or columns. Existing data is fixed only by `ALTER TABLE ... CONVERT TO CHARACTER SET`.

## 8. ALTER TABLE CONVERT TO Byte-Limit Math

`CONVERT TO CHARACTER SET` rebuilds the table, re-encodes every string column, and can widen column data types so they still hold the same character count.

### Index-prefix limits per row format (16k page size)

| Row format | Max index prefix per column |
|------------|------------------------------|
| `REDUNDANT` | 767 bytes |
| `COMPACT` | 767 bytes |
| `DYNAMIC` | 3072 bytes |
| `COMPRESSED` | 3072 bytes |

On 4k pages the limit is 768 bytes; on 8k pages 1536 bytes; on 16k pages 3072 bytes for `DYNAMIC`/`COMPRESSED`.

### The VARCHAR(255) calculation

utf8mb4 uses up to 4 bytes per character. An indexed column's worst-case index prefix is `column_char_length x 4`.

| Column | utf8mb4 worst-case index bytes | Fits COMPACT (767)? | Fits DYNAMIC (3072)? |
|--------|-------------------------------|---------------------|----------------------|
| `VARCHAR(191)` | 764 | YES | YES |
| `VARCHAR(255)` | 1020 | NO (error 1071) | YES |
| `VARCHAR(768)` | 3072 | NO | YES (at the boundary) |

This is why `VARCHAR(191)` was historically used as the "longest safely indexable utf8mb4 column" on `COMPACT`. On `DYNAMIC`, `VARCHAR(255)` indexes cleanly.

### innodb_large_prefix

On MariaDB before 10.x default-DYNAMIC, the global `innodb_large_prefix` toggled the limit between 767 (OFF) and 3072 (ON). From the era where `DYNAMIC` is the default row format (10.2+), the index-prefix limit is governed solely by the table's row format and `innodb_large_prefix` is no longer the relevant control. Do NOT add `innodb_large_prefix` to new configurations.

### Safe conversion order

```sql
-- 10.6+ : 1) ensure DYNAMIC, 2) then CONVERT
ALTER TABLE t ROW_FORMAT=DYNAMIC;
ALTER TABLE t CONVERT TO CHARACTER SET utf8mb4
  COLLATE utf8mb4_uca1400_ai_ci;
```

## 9. Inspecting Charset and Collation

```sql
-- 10.6+ : server and connection defaults
SHOW VARIABLES LIKE 'character_set\_%';
SHOW VARIABLES LIKE 'collation\_%';

-- table-level charset and collation
SHOW TABLE STATUS FROM mydb LIKE 'mytable';
SHOW CREATE TABLE mydb.mytable;

-- column-level via INFORMATION_SCHEMA
SELECT COLUMN_NAME, CHARACTER_SET_NAME, COLLATION_NAME
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA = 'mydb' AND TABLE_NAME = 'mytable';

-- available charsets and their default collation
SELECT CHARACTER_SET_NAME, DEFAULT_COLLATE_NAME, MAXLEN
FROM information_schema.CHARACTER_SETS
WHERE CHARACTER_SET_NAME LIKE 'utf8%';
```

`MAXLEN` from `INFORMATION_SCHEMA.CHARACTER_SETS` is the max bytes per character: `utf8mb3` reports 3, `utf8mb4` reports 4.

## 10. JSON and Charset

MariaDB `JSON` is an alias for `LONGTEXT`. It has no native binary storage; it is text plus an automatic `JSON_VALID` interpretation hint. A JSON column inherits the table charset and collation like any text column.

ALWAYS create JSON columns on a `utf8mb4` table so JSON string values can hold the full Unicode range. Add `CHECK (JSON_VALID(col))` to enforce structure; build functional indexes on virtual columns (`JSON_VALUE(...)`) for index access. A JSON column on a `utf8mb3` table silently truncates 4-byte characters inside JSON string values.

## 11. Source Verification

- `mariadb.com/docs/server/reference/data-types/string-data-types/character-sets/unicode` : `utf8`/`utf8mb3` 3-byte max, `utf8mb4` 4-byte supplementary characters.
- `mariadb.com/docs/server/server-management/variables-and-modes/old_mode` : `UTF8_IS_UTF8MB3` flag (10.6.1+), deprecated by design.
- `mariadb.com/kb/en/supported-character-sets-and-collations/` : collation list, suffix semantics, default `utf8mb4_uca1400_ai_ci` from 11.5, 184 UCA-14.0.0 collations in 10.10.
- `mariadb.com/docs/server/reference/data-types/string-data-types/character-sets/setting-character-sets-and-collations` : `SET NAMES`, connection variables, charset cascade, `CONVERT TO` widens data types.
- `mariadb.com/docs/server/server-usage/storage-engines/innodb/innodb-row-formats` : 767-byte (COMPACT/REDUNDANT) vs 3072-byte (DYNAMIC/COMPRESSED) index-prefix limits.
- `mariadb.com/kb/en/coercibility/` : coercibility levels 0-6, `COLLATE` clause.
