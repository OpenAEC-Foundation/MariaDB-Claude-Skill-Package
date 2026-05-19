# anti-patterns.md : Encoding and Collation Anti-Patterns

Each entry: the broken code, WHY it fails, and the correct alternative. All verified against MariaDB 10.6-LTS, 10.11-LTS, 11.x, 12.x.

## AP-1 : Using `utf8` charset for user-generated text

### Broken

```sql
-- 10.6+ : WRONG
CREATE TABLE posts (
  body TEXT CHARACTER SET utf8        -- utf8 == utf8mb3 == 3 bytes
);
```

### Why it fails

In MariaDB, `utf8` is an alias for `utf8mb3`, which encodes a maximum of 3 bytes per character. Every emoji, every CJK extension character, and every code point above U+FFFF needs 4 bytes. Inserting one rejects the row with error 1366 (`Incorrect string value`) or, depending on `sql_mode`, silently truncates the string at the bad character. Relying on the `old_mode` `UTF8_IS_UTF8MB3` flag to redefine `utf8` is not a fix: the flag is environment-specific, deprecated by design, and makes the schema ambiguous.

### Correct

```sql
-- 10.6+ : CORRECT
CREATE TABLE posts (
  body TEXT CHARACTER SET utf8mb4 COLLATE utf8mb4_uca1400_ai_ci
);
```

ALWAYS write `utf8mb4` explicitly. NEVER write `utf8` for any column that can hold user input.

## AP-2 : JOIN across two different collations

### Broken

```sql
-- 10.6+ : WRONG, mixed collations
-- legacy table:
CREATE TABLE old_t (k VARCHAR(50) COLLATE utf8mb4_general_ci);
-- table created on 11.6:
CREATE TABLE new_t (k VARCHAR(50) COLLATE utf8mb4_uca1400_ai_ci);

SELECT * FROM old_t JOIN new_t ON old_t.k = new_t.k;
```

### Why it fails

The two columns have different collations and equal coercibility (both are columns, level 2). MariaDB cannot decide which collation governs the `=` comparison and raises error 1267 (`Illegal mix of collations`). This commonly surfaces after a server upgrade: tables created before 11.5 default to `utf8mb4_general_ci`, tables created on 11.5+ default to `utf8mb4_uca1400_ai_ci`, and a JOIN across the two breaks.

### Correct

```sql
-- 10.6+ : align the column collations permanently
ALTER TABLE old_t MODIFY k VARCHAR(50)
  CHARACTER SET utf8mb4 COLLATE utf8mb4_uca1400_ai_ci;

SELECT * FROM old_t JOIN new_t ON old_t.k = new_t.k;
```

ALWAYS standardise on one collation across the database. A per-query `COLLATE` clause is only an emergency patch and disables the index on the converted side.

## AP-3 : CONVERT TO utf8mb4 on a COMPACT VARCHAR(255) index

### Broken

```sql
-- 10.6+ : WRONG, exceeds the 767-byte limit
CREATE TABLE t (
  email VARCHAR(255) CHARACTER SET latin1,
  KEY idx_email (email)
) ROW_FORMAT=COMPACT;

ALTER TABLE t CONVERT TO CHARACTER SET utf8mb4;
-- error 1071: Specified key was too long; max key length is 767 bytes
```

### Why it fails

Under utf8mb4 an indexed `VARCHAR(255)` has a worst-case index prefix of 255 x 4 = 1020 bytes. The `COMPACT` row format caps the index prefix at 767 bytes, so the rebuild fails with error 1071.

### Correct

```sql
-- 10.6+ : switch row format first, then convert
ALTER TABLE t ROW_FORMAT=DYNAMIC;
ALTER TABLE t CONVERT TO CHARACTER SET utf8mb4
  COLLATE utf8mb4_uca1400_ai_ci;
```

`DYNAMIC` raises the index-prefix limit to 3072 bytes (16k page), so 1020 bytes fits. `DYNAMIC` is the default row format from 10.2; legacy tables may still be `COMPACT`.

## AP-4 : Mismatched connection charset (mojibake)

### Broken

```python
# Python : WRONG, no charset declared, driver may default to latin1
conn = mariadb.connect(host="db", user="app", database="mydb")
cur = conn.cursor()
cur.execute("INSERT INTO comments (body) VALUES (?)", ("accented text",))
```

### Why it fails

The table column is `utf8mb4`, but the connection charset is whatever the driver defaults to. If it is `latin1`, the server interprets every byte of the multi-byte input as a latin1 character and re-encodes it, storing mojibake. There is NO error: the corruption is silent and only visible when the data is read back.

### Correct

```python
# Python : CORRECT, charset pinned on the connection
conn = mariadb.connect(host="db", user="app", database="mydb",
                       charset="utf8mb4")
```

ALWAYS set the charset in the driver connection options so it survives reconnects. A manual `SET NAMES utf8mb4` is lost on the next reconnect.

## AP-5 : `_bin` collation for case-insensitive search

### Broken

```sql
-- 10.6+ : WRONG, _bin for a name search column
CREATE TABLE people (
  name VARCHAR(80) CHARACTER SET utf8mb4 COLLATE utf8mb4_bin
);

SELECT * FROM people WHERE name = 'alice';
-- Misses the row stored as 'Alice'
```

### Why it fails

`_bin` compares raw encoded bytes. `'A'` (0x41) and `'a'` (0x61) are different bytes, so `'Alice'` never equals `'alice'`. `_bin` also has no accent folding. For human-facing search this drops most legitimate matches.

### Correct

```sql
-- 10.6+ : case- and accent-insensitive search
CREATE TABLE people (
  name VARCHAR(80) CHARACTER SET utf8mb4 COLLATE utf8mb4_uca1400_ai_ci
);
```

Use `_ai_ci` for human-facing search. Reserve `_bin` for exact-match technical identifiers (tokens, hashes) that must compare byte-for-byte.

## AP-6 : mysqldump without `--default-character-set`

### Broken

```bash
# WRONG : no charset flag on dump or restore
mariadb-dump mydb > mydb.sql
mariadb mydb < mydb.sql
```

### Why it fails

`mariadb-dump` chooses a default charset for the dump's `SET NAMES` statements. If that default does not match the data charset, every multi-byte string is re-encoded incorrectly on restore. The dump file looks fine and the restore reports no error, but the restored data is mojibake.

### Correct

```bash
# CORRECT : same charset on both ends
mariadb-dump --default-character-set=utf8mb4 --single-transaction \
  mydb > mydb.sql
mariadb --default-character-set=utf8mb4 mydb < mydb.sql
```

ALWAYS pass `--default-character-set=utf8mb4` to BOTH `mariadb-dump` and the restore command.

## AP-7 : Changing a column collation without checking dependent indexes and keys

### Broken

```sql
-- 10.6+ : WRONG, blind collation change
ALTER TABLE orders
  MODIFY status VARCHAR(20) COLLATE utf8mb4_bin;
-- 'status' is referenced by a foreign key from order_history.status
```

### Why it fails

A foreign-key relationship requires the parent and child columns to share the same charset AND collation. Changing only one side breaks the FK constraint or fails the ALTER. Changing a column's collation also silently rebuilds every index on that column, which on a large table is a heavy operation, and can change query results if the application relied on the old comparison behavior.

### Correct

```sql
-- 10.6+ : find dependents first
SELECT TABLE_NAME, COLUMN_NAME, CONSTRAINT_NAME, REFERENCED_TABLE_NAME
FROM information_schema.KEY_COLUMN_USAGE
WHERE REFERENCED_TABLE_NAME = 'orders'
   OR (TABLE_NAME = 'orders' AND REFERENCED_TABLE_NAME IS NOT NULL);

-- Then change the parent and every child column together,
-- in one coordinated migration.
ALTER TABLE orders        MODIFY status VARCHAR(20)
  CHARACTER SET utf8mb4 COLLATE utf8mb4_uca1400_ai_ci;
ALTER TABLE order_history MODIFY status VARCHAR(20)
  CHARACTER SET utf8mb4 COLLATE utf8mb4_uca1400_ai_ci;
```

ALWAYS inventory foreign keys and indexes before changing a column collation.

## AP-8 : Assuming the server default charset applies retroactively

### Broken

```sql
-- 10.6+ : WRONG, expecting old tables to be fixed
SET GLOBAL character_set_server = 'utf8mb4';
-- Operator now believes every existing table is utf8mb4
```

### Why it fails

The charset cascade (column to table to database to server) is evaluated ONCE, at object-creation time. Changing `character_set_server` or running `ALTER DATABASE ... CHARACTER SET utf8mb4` affects only objects created AFTERWARD. Existing tables and columns keep their original charset. The operator thinks the database is utf8mb4, but legacy tables still silently lose emoji.

### Correct

```sql
-- 10.6+ : set the default for NEW objects ...
SET GLOBAL character_set_server = 'utf8mb4';
-- ... and explicitly convert EXISTING tables
ALTER TABLE legacy_a ROW_FORMAT=DYNAMIC;
ALTER TABLE legacy_a CONVERT TO CHARACTER SET utf8mb4
  COLLATE utf8mb4_uca1400_ai_ci;
-- repeat for every existing table (see examples.md Example 13)
```

ALWAYS audit existing tables with `INFORMATION_SCHEMA.COLUMNS` and convert each one explicitly. The server default never migrates existing data.

## AP-9 : Putting JSON in a utf8mb3 table

### Broken

```sql
-- 10.6+ : WRONG, JSON column on a 3-byte table
CREATE TABLE events (
  payload JSON
) CHARACTER SET utf8;
```

### Why it fails

MariaDB `JSON` is an alias for `LONGTEXT` and inherits the table charset. On a `utf8`/`utf8mb3` table, any 4-byte character inside a JSON string value is truncated or rejected, even though the JSON structure itself is valid. The `JSON` type does not protect against this; it is text storage.

### Correct

```sql
-- 10.6+ : JSON on a utf8mb4 table, with validation
CREATE TABLE events (
  payload LONGTEXT
          CHARACTER SET utf8mb4 COLLATE utf8mb4_bin
          CHECK (JSON_VALID(payload))
) CHARACTER SET utf8mb4;
```

ALWAYS create JSON columns on a `utf8mb4` table. Add `CHECK (JSON_VALID(col))` for structure and build functional indexes on virtual `JSON_VALUE` columns for index access. MariaDB JSON is a LONGTEXT alias, not native binary storage.
