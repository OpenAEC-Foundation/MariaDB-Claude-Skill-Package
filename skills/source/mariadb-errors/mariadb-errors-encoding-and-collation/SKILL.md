---
name: mariadb-errors-encoding-and-collation
description: >
  Use when emoji or Unicode characters fail to store, "Illegal mix of collations" errors appear in JOINs, ALTER TABLE CONVERT TO is risky, or migrating charset from latin1/utf8mb3 to utf8mb4.
  Prevents the common mistake of using utf8 (3-byte alias for utf8mb3) instead of utf8mb4, mixed-collation JOIN failures, or in-place CONVERT TO breaking index-prefix limits.
  Covers utf8 vs utf8mb4 (3-byte trap), collation suffixes (_ci / _cs / _bin), utf8mb4_uca1400 collation family (default 11.5+), Illegal mix of collations error, ALTER TABLE CONVERT TO CHARACTER SET risks, connection charset, index-prefix byte limits.
  Keywords: encoding, collation, charset, utf8, utf8mb3, utf8mb4, utf8mb4_uca1400, utf8mb4_general_ci, _ci _cs _bin, Illegal mix of collations, emoji not stored, character set, CONVERT TO CHARACTER SET, connection charset, SET NAMES, index prefix limit, why does my emoji disappear, question marks instead of characters, latin1 migration, old_mode, UTF8_IS_UTF8MB3, Incorrect string value, mojibake
license: MIT
compatibility: "Designed for Claude Code. Requires MariaDB 10.6-LTS, 10.11-LTS, 11.x, 12.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# MariaDB Encoding and Collation Errors

How to diagnose and fix character-set problems in MariaDB: the `utf8` 3-byte trap that loses emoji, the `Illegal mix of collations` error (1267) in JOINs, risky `ALTER TABLE CONVERT TO CHARACTER SET` operations, and connection-charset mismatches that produce mojibake.

## Quick Reference

- In MariaDB, `utf8` is an alias for `utf8mb3` (maximum 3 bytes per character). `utf8mb3` CANNOT store emoji or any code point above U+FFFF. ALWAYS use `utf8mb4` for any text that can hold user-generated content.
- `Illegal mix of collations` (error 1267) means two strings in a comparison or JOIN have different collations and neither can be coerced to the other. Fix with an explicit `COLLATE` clause or by ALTERing the columns to the same collation.
- `utf8mb4_uca1400_ai_ci` is the modern default collation from MariaDB 11.5 onward. Before 11.5 the default was `utf8mb4_general_ci`. Mixing these two in a JOIN triggers error 1267.
- `ALTER TABLE ... CONVERT TO CHARACTER SET utf8mb4` rewrites EVERY row and can grow indexed columns past the 767-byte index-prefix limit of the `COMPACT` row format. Use `ROW_FORMAT=DYNAMIC` (limit 3072 bytes) before converting.
- Set the connection charset with `SET NAMES utf8mb4`. A connection charset that does not match the column charset produces mojibake (garbled text) even when the table is correct.
- `Incorrect string value` (error 1366) on INSERT means a 4-byte character was sent to a `utf8mb3` (or `latin1`) column. The fix is to convert the column to `utf8mb4`, not to strip the character.
- MariaDB JSON is a `LONGTEXT` alias, not native binary. Use `CHECK (JSON_VALID(col))` for structure; use functional indexes on virtual columns for index access. JSON columns inherit the table charset and should be `utf8mb4`.
- NEVER use a `_bin` collation for case-insensitive search: `_bin` compares raw bytes, so `'A'` never matches `'a'`.

## Error Identification

| Symptom | Error number | Symbol | Meaning |
|---------|--------------|--------|---------|
| Emoji or CJK extension character rejected on INSERT | 1366 | `ER_TRUNCATED_WRONG_VALUE_FOR_FIELD` | A 4-byte character was sent to a `utf8mb3` or `latin1` column |
| `Illegal mix of collations (...) for operation` on a JOIN, WHERE, or UNION | 1267 | `ER_CANNOT_CONVERT_USING` / cant-aggregate | Two operands have incompatible collations with equal coercibility |
| `Specified key was too long; max key length is 767 bytes` on CREATE INDEX or ALTER | 1071 | `ER_TOO_LONG_KEY` | Indexed column prefix exceeds the row-format limit (767 on COMPACT) |
| Stored text shows as `?` characters | (no error) | (silent) | Connection charset cannot represent the data; server replaced characters |
| Stored text shows as garbled multi-character sequences (mojibake) | (no error) | (silent) | Connection charset disagrees with the actual byte encoding |

Errors 1366 and 1071 happen at write/DDL time and are loud. Mojibake and `?` substitution are SILENT data corruption: there is no error, only wrong bytes.

## The utf8 vs utf8mb4 Trap

In MariaDB, the historic `utf8` charset name is an alias for `utf8mb3`, which encodes a maximum of 3 bytes per character. The full UTF-8 standard needs up to 4 bytes; emoji, many CJK extension characters, and all supplementary-plane code points (above U+FFFF) require 4 bytes.

A column declared `CHARACTER SET utf8` therefore CANNOT store an emoji. On insert MariaDB either truncates the string at the bad character or rejects the row with error 1366.

```sql
-- 10.6+ : WRONG, this column cannot hold emoji
CREATE TABLE comment_bad (body VARCHAR(500) CHARACTER SET utf8);

-- 10.6+ : CORRECT
CREATE TABLE comment_good (body VARCHAR(500) CHARACTER SET utf8mb4);
```

From MariaDB 10.6.1 the canonical name of the 3-byte set is `utf8mb3`; `utf8` remains an alias. The `old_mode` system variable flag `UTF8_IS_UTF8MB3` controls the alias target:

- `UTF8_IS_UTF8MB3` set (or `old_mode` empty on current releases): `utf8` means `utf8mb3` (3-byte).
- `UTF8_IS_UTF8MB3` cleared: `utf8` means `utf8mb4` (4-byte).

NEVER rely on `old_mode` to make `utf8` safe. The flag is environment-specific and `old_mode` options are deprecated by design. ALWAYS write `utf8mb4` explicitly in DDL so the schema is unambiguous regardless of server configuration.

## Decision Tree

```
Character-set problem
|
+-- Emoji / non-BMP character rejected (error 1366) or shows as '?'
|     -> Column charset is utf8mb3 or latin1.
|        Convert the column to utf8mb4 (see CONVERT TO section).
|
+-- "Illegal mix of collations" (error 1267) on JOIN / WHERE / UNION
|     -> Two columns have different collations.
|        Quick fix : add COLLATE to one operand in the query.
|        Permanent fix : ALTER both columns to the same collation.
|
+-- "max key length is 767 bytes" (error 1071) on CREATE INDEX / ALTER
|     -> Indexed column too wide for COMPACT row format under utf8mb4.
|        Set ROW_FORMAT=DYNAMIC, or shorten the column / use an index prefix.
|
+-- Text is garbled (mojibake), no error
|     -> Connection charset disagrees with the stored bytes.
|        Run SET NAMES utf8mb4 on the connection; verify the
|        application driver charset. Do NOT re-convert the column yet.
|
+-- New tables still default to latin1 / utf8mb3
      -> Server default charset is old. Set character-set-server=utf8mb4
         in my.cnf and declare DEFAULT CHARSET explicitly per table.
```

## Collation Suffixes

A collation is a comparison-and-sort ruleset for a charset. The suffix encodes its behavior:

| Suffix | Meaning | Effect |
|--------|---------|--------|
| `_ci` | Case-insensitive | `'A' = 'a'` is TRUE |
| `_cs` | Case-sensitive | `'A' = 'a'` is FALSE, accents distinguished |
| `_bin` | Binary | Raw byte comparison; `'A' = 'a'` FALSE, no Unicode awareness |
| `_ai` | Accent-insensitive (10.10+) | `'e' = 'e-with-accent'` is TRUE |
| `_as` | Accent-sensitive (10.10+) | accents distinguished |
| `_nopad` | No-pad (10.10+) | trailing spaces are significant characters |

`utf8mb4_uca1400_ai_ci` reads as: utf8mb4 charset, UCA 14.0.0 algorithm, accent-insensitive, case-insensitive. This is the MariaDB 11.5+ default.

NEVER use `_bin` for human-facing search: it misses every case and accent variant. Use `_bin` only for exact-match technical identifiers (tokens, hashes, base64).

## Collation Family and Defaults

| MariaDB version | Default utf8mb4 collation |
|-----------------|----------------------------|
| 10.6, 10.11, 11.0 - 11.4 | `utf8mb4_general_ci` |
| 11.5+ (incl. 11.6-LTS, 12.x) | `utf8mb4_uca1400_ai_ci` |

MariaDB 10.10 added 184 UCA-14.0.0 collations (`utf8mb4_uca1400_*`). They fix sort-order anomalies present in the legacy `utf8mb4_general_ci` and `utf8mb4_unicode_ci`.

The practical hazard: a table created on 10.11 has `utf8mb4_general_ci` columns; a table created on 11.6 has `utf8mb4_uca1400_ai_ci` columns. Joining a column from each triggers error 1267. See the Illegal Mix section.

See `references/methods.md` for the full charset/collation matrix per version.

## Illegal Mix of Collations (Error 1267)

Error 1267 fires when a comparison, JOIN, UNION, or function call combines two string operands whose collations differ and neither has lower coercibility than the other. Coercibility ranks how "movable" a value's collation is (a literal is more coercible than a column; an explicit `COLLATE` is least coercible and wins).

```sql
-- Reproduce : two tables with different collations
CREATE TABLE a (name VARCHAR(50)) COLLATE utf8mb4_general_ci;
CREATE TABLE b (name VARCHAR(50)) COLLATE utf8mb4_uca1400_ai_ci;

-- 10.6+ : this JOIN fails with error 1267
SELECT * FROM a JOIN b ON a.name = b.name;
```

Two fixes:

```sql
-- Quick fix : force one side's collation in the query
SELECT * FROM a JOIN b
  ON a.name = b.name COLLATE utf8mb4_uca1400_ai_ci;

-- Permanent fix : align the column collations
ALTER TABLE a MODIFY name VARCHAR(50)
  CHARACTER SET utf8mb4 COLLATE utf8mb4_uca1400_ai_ci;
```

ALWAYS prefer the permanent fix. Per-query `COLLATE` clauses defeat indexes on the converted side and must be repeated in every query.

## ALTER TABLE CONVERT TO CHARACTER SET

`CONVERT TO CHARACTER SET` rewrites every row and re-encodes every string column. Two hazards:

1. It can change column data types. A `TEXT` column converts to `MEDIUMTEXT` under utf8mb4 because each character now needs more bytes.
2. Indexed columns can exceed the row-format index-prefix limit.

| Row format | Max index-prefix per column (16k page) |
|------------|-----------------------------------------|
| `COMPACT`, `REDUNDANT` | 767 bytes |
| `DYNAMIC`, `COMPRESSED` | 3072 bytes |

An indexed `VARCHAR(255)` under utf8mb4 needs 255 x 4 = 1020 bytes. That exceeds 767 (`COMPACT`) and triggers error 1071, but fits within 3072 (`DYNAMIC`).

```sql
-- 10.6+ : safe full-table conversion
ALTER TABLE customers ROW_FORMAT=DYNAMIC;
ALTER TABLE customers CONVERT TO CHARACTER SET utf8mb4
  COLLATE utf8mb4_uca1400_ai_ci;
```

`DYNAMIC` is the default row format from MariaDB 10.2. On older 10.x the global `innodb_large_prefix` had to be ON to reach 3072 bytes; on 10.6+ the limit is governed solely by the row format, so `innodb_large_prefix` is no longer relevant.

NEVER run `CONVERT TO` on a large production table without measuring lock and disk impact first. It is a full table rebuild. See `references/examples.md` for an online-DDL approach.

## Connection Charset

The stored bytes can be correct while the displayed text is garbled, because three connection variables decide how the client and server interpret strings:

| Variable | Role |
|----------|------|
| `character_set_client` | charset of incoming query text and string literals |
| `character_set_connection` | charset queries are converted to for comparison |
| `character_set_results` | charset of result rows sent back to the client |

`SET NAMES utf8mb4` sets all three to `utf8mb4` in one statement. If the application sends utf8mb4 bytes but the connection is `latin1`, the server interprets each byte as a latin1 character and stores mojibake.

```sql
-- 10.6+ : run immediately after connecting
SET NAMES utf8mb4;
```

ALWAYS configure the charset in the driver connection options (for example `charset=utf8mb4` in the DSN) rather than relying on a manual `SET NAMES`. The server default `character-set-server` does NOT change the charset of connections that negotiate their own.

See `references/methods.md` for per-driver connection-charset settings and `references/anti-patterns.md` for the mojibake anti-pattern.

## Charset Hierarchy

Charset and collation cascade: column -> table -> database -> server. A column with no explicit charset inherits the table default; the table inherits the database; the database inherits `character-set-server`.

This cascade applies ONLY at object-creation time. Changing the server or database default does NOT retroactively convert existing tables. ALWAYS declare `DEFAULT CHARSET=utf8mb4` per table and use `CONVERT TO` to fix existing tables.

```ini
# my.cnf : make new objects default to utf8mb4
[mysqld]
character-set-server = utf8mb4
collation-server     = utf8mb4_uca1400_ai_ci
```

## mysqldump Charset Safety

`mariadb-dump` (the renamed `mysqldump`, 10.5+) writes `SET NAMES` statements into the dump based on `--default-character-set`. A dump taken or restored with a mismatched charset corrupts every multi-byte string.

```bash
# 10.6+ : charset-safe dump and restore
mariadb-dump --default-character-set=utf8mb4 --single-transaction \
  mydb > mydb.sql
mariadb --default-character-set=utf8mb4 mydb < mydb.sql
```

ALWAYS pass `--default-character-set=utf8mb4` to BOTH the dump and the restore command. See `references/examples.md` for a charset-safe round-trip.

## Detecting Non-utf8mb4 Columns

Audit a database for columns that still use a 3-byte or single-byte charset:

```sql
-- 10.6+ : list every column not on utf8mb4
SELECT TABLE_SCHEMA, TABLE_NAME, COLUMN_NAME,
       CHARACTER_SET_NAME, COLLATION_NAME
FROM information_schema.COLUMNS
WHERE CHARACTER_SET_NAME IS NOT NULL
  AND CHARACTER_SET_NAME <> 'utf8mb4'
  AND TABLE_SCHEMA NOT IN ('mysql','information_schema',
                           'performance_schema','sys');
```

See `references/examples.md` for a script that also reports collation mismatches across joined tables.

## What This Skill Does NOT Cover

- General schema design and charset choice for new tables: see `mariadb-impl-schema-design`.
- MySQL-to-MariaDB migration end to end: see `mariadb-impl-migration-mysql-to-mariadb`.
- JSON syntax and functions: see `mariadb-syntax-json`.
- Backup and restore tooling beyond charset flags: see `mariadb-impl-backup-recovery`.

## Reference Links

- `references/methods.md` : charset/collation matrix per version, collation-suffix semantics, error-code reference (1267, 1366, 1071), connection-charset variables, ALTER CONVERT TO byte-limit math.
- `references/examples.md` : 10+ working examples (reproduce utf8 emoji failure, fix with utf8mb4, Illegal-mix reproduction and COLLATE fix, full-table CONVERT TO with ROW_FORMAT, connection charset, charset-safe mysqldump round-trip, INFORMATION_SCHEMA audit).
- `references/anti-patterns.md` : 8+ real anti-patterns (utf8 for user text, mixed-collation JOIN, CONVERT TO on COMPACT VARCHAR(255), connection mismatch, _bin for case-insensitive search, mysqldump without charset flag, collation change ignoring indexes, server-default assumed retroactive).

## Source Verification

All facts in this skill were verified via WebFetch against:
- `mariadb.com/docs/server/reference/data-types/string-data-types/character-sets/unicode` : `utf8` is an alias for `utf8mb3` (max 3 bytes), `utf8mb4` stores supplementary characters in 4 bytes, alias controlled by `old_mode`.
- `mariadb.com/docs/server/server-management/variables-and-modes/old_mode` : `UTF8_IS_UTF8MB3` flag (10.6.1+), when set `utf8` aliases `utf8mb3`, when cleared `utf8mb4`; `old_mode` deprecated by design.
- `mariadb.com/kb/en/supported-character-sets-and-collations/` : `utf8mb4_general_ci` default through 11.4, `utf8mb4_uca1400_ai_ci` default from 11.5; 184 UCA-14.0.0 collations added in 10.10; `_ci`/`_cs`/`_bin`/`_ai`/`_as`/`_nopad` suffix semantics.
- `mariadb.com/docs/server/reference/data-types/string-data-types/character-sets/setting-character-sets-and-collations` : `SET NAMES`, `character_set_client`/`character_set_connection`/`character_set_results`, charset cascade column->table->database->server, `CONVERT TO CHARACTER SET` widens data types.
- `mariadb.com/docs/server/server-usage/storage-engines/innodb/innodb-row-formats` : 767-byte index-prefix limit on `COMPACT`, 3072-byte limit on `DYNAMIC` (16k page), `innodb_large_prefix` superseded by row format.
- `mariadb.com/kb/en/coercibility/` : coercibility 0 (explicit) to 6 (ignorable), `COLLATE` clause resolves collation conflicts.
- Vooronderzoek Cluster-1 and Cluster-3 : utf8 3-byte trap, latin1 lock-in on stock 10.6/10.11, default charset migrated to utf8mb4 in 11.6, `utf8mb4` + `innodb_default_row_format=dynamic` requirement.
