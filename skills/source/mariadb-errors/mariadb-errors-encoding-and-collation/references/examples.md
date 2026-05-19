# examples.md : Encoding and Collation Working Examples

Each example is self-contained and version-annotated. All SQL verified against MariaDB 10.6-LTS, 10.11-LTS, 11.x, and 12.x unless noted.

## Example 1 : Reproduce the utf8 emoji failure

```sql
-- 10.6+ : a utf8mb3 column cannot store a 4-byte character
CREATE TABLE emoji_bad (
  id   INT PRIMARY KEY AUTO_INCREMENT,
  body VARCHAR(100) CHARACTER SET utf8        -- utf8 = utf8mb3 = 3-byte
);

-- This INSERT fails with error 1366:
-- Incorrect string value: '\xF0\x9F\x98\x80' for column 'body'
INSERT INTO emoji_bad (body) VALUES ('hello world');
```

The byte sequence `F0 9F 98 80` is the emoji U+1F600. utf8mb3 has no 4-byte form, so the server rejects the row.

## Example 2 : Fix with utf8mb4

```sql
-- 10.6+ : utf8mb4 stores the full Unicode range
CREATE TABLE emoji_good (
  id   INT PRIMARY KEY AUTO_INCREMENT,
  body VARCHAR(100) CHARACTER SET utf8mb4
                    COLLATE utf8mb4_uca1400_ai_ci   -- 11.5+ default; on 10.6/10.11 use utf8mb4_general_ci
);

-- Succeeds:
INSERT INTO emoji_good (body) VALUES ('hello world emoji');

SELECT id, CHAR_LENGTH(body) AS chars, OCTET_LENGTH(body) AS bytes
FROM emoji_good;
-- chars counts characters, bytes counts storage; for a 4-byte char they differ.
```

## Example 3 : Reproduce Illegal mix of collations (error 1267)

```sql
-- 10.6+ : two tables, two different collations
CREATE TABLE users_old (
  username VARCHAR(64) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci
);
CREATE TABLE users_new (
  username VARCHAR(64) CHARACTER SET utf8mb4 COLLATE utf8mb4_uca1400_ai_ci
);

INSERT INTO users_old (username) VALUES ('alice');
INSERT INTO users_new (username) VALUES ('alice');

-- This JOIN fails with error 1267:
-- Illegal mix of collations (utf8mb4_general_ci,IMPLICIT) and
-- (utf8mb4_uca1400_ai_ci,IMPLICIT) for operation '='
SELECT * FROM users_old o
JOIN users_new n ON o.username = n.username;
```

## Example 4 : Fix error 1267 with a COLLATE clause (quick fix)

```sql
-- 10.6+ : force one operand's collation; this is a per-query workaround
SELECT * FROM users_old o
JOIN users_new n
  ON o.username = n.username COLLATE utf8mb4_uca1400_ai_ci;
```

The `COLLATE` clause has explicit coercibility (level 0) and wins. WARNING: applying `COLLATE` to an indexed column forces a full scan on that side because the index uses the column's native collation.

## Example 5 : Fix error 1267 permanently by aligning collations

```sql
-- 10.6+ : convert the old table's column to the modern collation
ALTER TABLE users_old
  MODIFY username VARCHAR(64)
    CHARACTER SET utf8mb4 COLLATE utf8mb4_uca1400_ai_ci;

-- Now the JOIN works with no COLLATE clause and uses indexes:
SELECT * FROM users_old o
JOIN users_new n ON o.username = n.username;
```

## Example 6 : Full-table CONVERT TO utf8mb4 with the correct row format

```sql
-- 10.6+ : a legacy COMPACT table with an indexed VARCHAR(255)
CREATE TABLE legacy_customers (
  id    INT PRIMARY KEY AUTO_INCREMENT,
  email VARCHAR(255) CHARACTER SET latin1,
  KEY idx_email (email)
) ROW_FORMAT=COMPACT;

-- Converting directly fails with error 1071:
-- Specified key was too long; max key length is 767 bytes
-- (255 chars x 4 bytes = 1020 bytes > 767)

-- Correct procedure: switch row format first, then convert
ALTER TABLE legacy_customers ROW_FORMAT=DYNAMIC;
ALTER TABLE legacy_customers
  CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_uca1400_ai_ci;

-- DYNAMIC allows a 3072-byte index prefix, so 1020 bytes fits.
```

## Example 7 : Online CONVERT TO on a large production table

```sql
-- 10.6+ : minimise locking on a big table
ALTER TABLE big_table ROW_FORMAT=DYNAMIC,
  ALGORITHM=INPLACE, LOCK=NONE;

-- CONVERT TO is a table rebuild; INPLACE is supported but still heavy.
-- Measure on a clone first. For zero-downtime, use an external tool
-- (pt-online-schema-change / gh-ost) or a maintenance window.
ALTER TABLE big_table
  CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_uca1400_ai_ci,
  ALGORITHM=INPLACE, LOCK=SHARED;
```

If MariaDB cannot satisfy `LOCK=NONE` for `CONVERT TO`, it raises an error rather than silently falling back to a more restrictive lock. Test the exact `ALGORITHM`/`LOCK` combination on a clone.

## Example 8 : Set the connection charset

```sql
-- 10.6+ : run immediately after connecting, before any query
SET NAMES utf8mb4;

-- Pin the collation too, to avoid relying on the version default:
SET NAMES utf8mb4 COLLATE utf8mb4_uca1400_ai_ci;

-- Verify:
SHOW VARIABLES LIKE 'character_set_client';
SHOW VARIABLES LIKE 'character_set_connection';
SHOW VARIABLES LIKE 'character_set_results';
```

Preferred: configure the charset in the driver (`charset=utf8mb4` in the DSN) so it survives reconnects. See `methods.md` section 6.

## Example 9 : Diagnose mojibake from a connection mismatch

```sql
-- 10.6+ : simulate the bug. utf8mb4 bytes interpreted as latin1.
CREATE TABLE charset_demo (txt VARCHAR(100)) CHARACTER SET utf8mb4;

-- Connection wrongly set to latin1, then a multi-byte string inserted:
SET NAMES latin1;
INSERT INTO charset_demo (txt) VALUES ('cafe accented');
-- The server reads the utf8mb4 bytes as latin1 and re-encodes:
-- the stored bytes are now corrupted.

-- Read it back with the correct connection charset:
SET NAMES utf8mb4;
SELECT txt, HEX(txt) FROM charset_demo;
-- HEX reveals whether the stored bytes are valid utf8mb4 or mojibake.
```

The fix for already-corrupted data is a controlled re-encode (dump with the wrong charset declared, restore with the right one) and is risky. ALWAYS fix the connection charset BEFORE inserting more data.

## Example 10 : Charset-safe mysqldump round-trip

```bash
# 10.6+ : mariadb-dump is the renamed mysqldump (10.5+)
mariadb-dump --default-character-set=utf8mb4 \
             --single-transaction \
             --routines --events \
             mydb > mydb.sql

# Restore with the SAME charset flag:
mariadb --default-character-set=utf8mb4 mydb < mydb.sql
```

Omitting `--default-character-set` on either side lets the tool pick a default that may not match the data, corrupting every multi-byte string.

## Example 11 : Audit a database for non-utf8mb4 columns

```sql
-- 10.6+ : every column not on utf8mb4
SELECT TABLE_SCHEMA, TABLE_NAME, COLUMN_NAME,
       DATA_TYPE, CHARACTER_SET_NAME, COLLATION_NAME
FROM information_schema.COLUMNS
WHERE CHARACTER_SET_NAME IS NOT NULL
  AND CHARACTER_SET_NAME <> 'utf8mb4'
  AND TABLE_SCHEMA NOT IN ('mysql','information_schema',
                           'performance_schema','sys')
ORDER BY TABLE_SCHEMA, TABLE_NAME;
```

## Example 12 : Find collation mismatches between joinable columns

```sql
-- 10.6+ : columns with the same name and charset but different collation
SELECT a.TABLE_NAME AS table_a, b.TABLE_NAME AS table_b,
       a.COLUMN_NAME, a.COLLATION_NAME AS coll_a,
       b.COLLATION_NAME AS coll_b
FROM information_schema.COLUMNS a
JOIN information_schema.COLUMNS b
  ON a.COLUMN_NAME = b.COLUMN_NAME
 AND a.CHARACTER_SET_NAME = b.CHARACTER_SET_NAME
 AND a.COLLATION_NAME <> b.COLLATION_NAME
 AND a.TABLE_SCHEMA = b.TABLE_SCHEMA
 AND a.TABLE_NAME < b.TABLE_NAME
WHERE a.TABLE_SCHEMA = 'mydb';
-- Any row here is a latent error 1267 waiting for a JOIN.
```

## Example 13 : Generate ALTER statements to standardise a whole database

```sql
-- 10.6+ : produce one CONVERT TO statement per table
SELECT CONCAT(
  'ALTER TABLE `', TABLE_SCHEMA, '`.`', TABLE_NAME, '` ',
  'ROW_FORMAT=DYNAMIC; ',
  'ALTER TABLE `', TABLE_SCHEMA, '`.`', TABLE_NAME, '` ',
  'CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_uca1400_ai_ci;'
) AS ddl
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'mydb'
  AND TABLE_TYPE = 'BASE TABLE';
-- Review every generated line before running; CONVERT TO is a rebuild.
```

## Example 14 : Choose a collation for a search column

```sql
-- 10.6+ : accent-insensitive search (cafe finds cafe-with-accent)
CREATE TABLE products (
  name VARCHAR(120)
       CHARACTER SET utf8mb4 COLLATE utf8mb4_uca1400_ai_ci
);

-- 10.6+ : exact-match technical token, case- and byte-exact
CREATE TABLE api_tokens (
  token CHAR(43)
        CHARACTER SET utf8mb4 COLLATE utf8mb4_bin
);
```

Use `_ai_ci` for human-facing search. Use `_bin` ONLY for technical identifiers that must match byte-for-byte.

## Example 15 : Verify a column is genuinely utf8mb4 end to end

```sql
-- 10.6+ : insert a 4-byte char and confirm round-trip
CREATE TABLE verify_mb4 (s VARCHAR(10) CHARACTER SET utf8mb4);
SET NAMES utf8mb4;
INSERT INTO verify_mb4 (s) VALUES (_utf8mb4 0xF09F9880);  -- U+1F600
SELECT s, CHAR_LENGTH(s) AS chars, OCTET_LENGTH(s) AS bytes,
       HEX(s) AS stored_bytes
FROM verify_mb4;
-- Expect chars=1, bytes=4, stored_bytes=F09F9880.
-- Any other result means a charset is wrong somewhere in the path.
```
