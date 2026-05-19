# Methods : the 10-dimension migration-validation scan

This file is the complete, deterministic procedure for validating a MySQL SQL dump, schema, or codebase before importing into MariaDB. Each dimension is an IF / THEN rule with a fixed severity, a pattern hint for detecting it in a dump, and the canonical skill that owns the fix.

The scan operates on three input shapes :

- a `mysqldump` / `mariadb-dump` SQL file (DDL + DML + user statements)
- a bare schema file (CREATE TABLE / CREATE INDEX / CREATE VIEW only)
- a codebase containing SQL strings (application source files)

Run all 10 dimensions in order. Produce one row per finding. Never short-circuit.

---

## Severity definitions

| Severity | Meaning | Effect on verdict |
|----------|---------|-------------------|
| BLOCKER | the dump will not import correctly, or apps will break on the imported schema | any BLOCKER -> verdict FAIL |
| WARNING | the dump imports, but a known footgun is now latent and unmitigated | WARNING-only -> verdict PASS WITH WARNINGS |
| SUGGESTION | a non-blocking improvement ; no functional risk | does not change the verdict |

---

## MySQL-vs-MariaDB divergence table

This table is the reference for the "divergence" column of every finding. It is the canonical phrasing the validator must use.

| # | Area | MySQL 5.7 / 8.0 | MariaDB 10.6+ | Interconvertible |
|---|------|-----------------|---------------|------------------|
| 1 | JSON storage | native packed binary, validates on write | LONGTEXT alias, NO write validation | no, needs CHECK |
| 2 | Default auth plugin | `caching_sha2_password` (8.0), `sha256_password` | `ed25519`, `mysql_native_password`, `unix_socket` | no, re-create users |
| 3 | GTID format | `uuid:seqno` | `domain-server-sequence` | no, one-way load |
| 4 | Ignored-index keyword | `INVISIBLE` (8.0) | `IGNORED` (10.6+) | no, rewrite keyword |
| 5 | sql_mode default | strict, MySQL-8 specific mode list | strict (10.2.4+), different list | partial, review explicitly |
| 6 | Sequences | not supported, AUTO_INCREMENT only | `CREATE SEQUENCE` + `NEXT VALUE FOR` (10.3+) | MariaDB superset |
| 7 | Functional indexes | inline expression index `(( expr ))` (8.0.13+) | not supported, needs virtual column | no, rewrite as virtual column |
| 8 | Storage engines | InnoDB, MyISAM, MEMORY, plus MySQL-only plugins | InnoDB, Aria, MyISAM, plus MariaDB-only plugins | depends per engine |
| 9 | Window frames | `ROWS`, `RANGE`, `GROUPS` (8.0) | `ROWS`, `RANGE` only | no, rewrite GROUPS |
| 10 | Default charset | `utf8mb4` (8.0), `utf8`/`utf8mb3` legacy | `utf8mb4` recommended, `utf8mb3` legacy | yes, convert charset |

---

## Dimension 1 : JSON columns

- **Rule** : IF a column is declared `JSON` THEN emit a finding at severity WARNING.
- **Pattern hint** : match column definitions containing the token `JSON` as a type, e.g. regex `^\s*\`?\w+\`?\s+JSON\b` inside a CREATE TABLE body. Also match `CAST(... AS JSON)` only as informational, not a finding.
- **Divergence** : MySQL 5.7.8+ stores JSON in a packed binary format that validates JSON syntax on every INSERT and UPDATE. MariaDB `JSON` is an alias for `LONGTEXT` and performs NO write-time validation. A row that was valid JSON on MySQL can be corrupted by a later non-JSON UPDATE on MariaDB with no error.
- **Remediation** : after import, for every JSON column add `ALTER TABLE t ADD CONSTRAINT t_col_json CHECK (JSON_VALID(col));`. To index a JSON path, add a generated (virtual) column with `JSON_VALUE` and index that column.
- **Skill ref** : `mariadb-syntax-json`.
- **Never** : never grade a JSON column lower than WARNING, even on a "clean" dump. The missing validation is silent.

---

## Dimension 2 : Authentication plugins

- **Rule** : IF a `CREATE USER`, `ALTER USER`, `GRANT`, or `mysql.user` INSERT names plugin `caching_sha2_password` or `sha256_password` THEN emit a finding at severity BLOCKER.
- **Pattern hint** : match `IDENTIFIED WITH caching_sha2_password`, `IDENTIFIED WITH sha256_password`, or `plugin` column values `caching_sha2_password` / `sha256_password` in a `mysql.user` / `mysql.global_priv` dump segment.
- **Divergence** : MariaDB ships `mysql_native_password`, `ed25519`, `unix_socket`, `pam`, and `gssapi`. It does NOT ship `caching_sha2_password` or `sha256_password`. A user row with one of those plugins is unusable after import : the user exists but cannot authenticate.
- **Remediation** : drop the affected users and re-create them. `INSTALL SONAME 'auth_ed25519';` once, then `CREATE OR REPLACE USER 'app'@'%' IDENTIFIED VIA ed25519 USING PASSWORD('new-secret');` and re-issue the original GRANTs. Use `mysql_native_password` only if a driver cannot do ed25519.
- **Skill ref** : `mariadb-core-security-model`.
- **Never** : never downgrade this to WARNING. The application cannot connect ; it is a hard import-and-run failure.

---

## Dimension 3 : GTID continuity

- **Rule** : IF the migration plan, dump header comments, or replication config expect GTID-tracked replication to continue across the MySQL-to-MariaDB boundary THEN emit a finding at severity BLOCKER.
- **Pattern hint** : match `SET @@GLOBAL.GTID_PURGED`, `gtid_executed`, `MASTER_AUTO_POSITION=1`, or `CHANGE MASTER ... MASTER_USE_GTID` referencing MySQL `uuid:seqno` values (a 36-character UUID followed by `:`). Also flag any human statement in the migration brief saying "keep GTID replication".
- **Divergence** : MySQL GTID is `uuid:seqno` (e.g. `3e11fa47-71ca-11e1-9e33-c80aa9429562:23`). MariaDB GTID is `domain-server-sequence` (e.g. `0-1-1234`). The two formats are NOT interconvertible. MariaDB can be a replica of a MySQL primary using BINLOG POSITION only ; MySQL cannot be a replica of a MariaDB primary at all.
- **Remediation** : treat the migration as a one-way dump-and-load. If a cut-over window needs a live replica, configure binlog-positional replication : `CHANGE MASTER TO MASTER_HOST='mysql-src', MASTER_LOG_FILE='...', MASTER_LOG_POS='...', MASTER_USE_GTID=NO;`. Discard `gtid_executed` / `GTID_PURGED` statements from the dump.
- **Skill ref** : `mariadb-core-replication-model`.

---

## Dimension 4 : INVISIBLE INDEX keyword

- **Rule** : IF DDL or app SQL uses the `INVISIBLE` keyword on an index THEN emit a finding at severity BLOCKER.
- **Pattern hint** : match `INDEX ... INVISIBLE`, `KEY ... INVISIBLE`, `ALTER TABLE ... ALTER INDEX ... INVISIBLE`, or `... VISIBLE` toggling. The token `INVISIBLE` adjacent to an index name is the trigger. Do NOT match `INVISIBLE` on a column (that is MySQL 8 invisible columns, a separate concern).
- **Divergence** : MySQL 8 marks an optimizer-ignored index with `INVISIBLE`. MariaDB 10.6+ uses `IGNORED`. The MySQL keyword produces a syntax error on MariaDB.
- **Remediation** : rewrite `INVISIBLE` to `IGNORED` and `VISIBLE` to `NOT IGNORED`. MariaDB syntax : `ALTER TABLE t ALTER INDEX idx IGNORED;` and `ALTER TABLE t ALTER INDEX idx NOT IGNORED;`.
- **Skill ref** : `mariadb-syntax-indexing`.

---

## Dimension 5 : sql_mode differences

- **Rule** : IF the dump sets a `sql_mode` that contains MySQL-8-only modes, or relies on the MySQL 8 default mode set THEN emit a finding at severity WARNING.
- **Pattern hint** : match `SET @@SESSION.sql_mode` / `SET sql_mode` statements ; flag values that differ from a known MariaDB default, in particular modes present on MySQL 8 but not MariaDB. Also flag the ABSENCE of any explicit `sql_mode` set, which means the import inherits MariaDB defaults that may differ from the MySQL origin.
- **Divergence** : both servers are strict by default (MySQL 8, MariaDB 10.2.4+), but the exact mode list differs. Behaviour such as zero-date handling, division-by-zero, and `ONLY_FULL_GROUP_BY` can shift if `sql_mode` is left implicit.
- **Remediation** : after import, set `sql_mode` explicitly to a value the application was tested against, in the server config and in the session. Do not rely on the default matching MySQL.
- **Skill ref** : `mariadb-core-defaults-and-sql-modes`.

---

## Dimension 6 : Sequences

- **Rule** : IF the schema uses `AUTO_INCREMENT` THEN emit a finding at severity SUGGESTION (informational, usually no action).
- **Pattern hint** : match `AUTO_INCREMENT` in a column definition or table option.
- **Divergence** : MySQL has only `AUTO_INCREMENT`. MariaDB additionally supports SQL-standard `CREATE SEQUENCE ... NEXT VALUE FOR ...` (10.3+). This is a MariaDB superset : an AUTO_INCREMENT dump imports unchanged.
- **Remediation** : keep `AUTO_INCREMENT` as-is during the migration. NEVER auto-convert AUTO_INCREMENT to a sequence as part of the import. Convert deliberately, table by table, only after the migration is stable and only if sequence semantics (gap-free allocation, shared counters) are actually needed.
- **Skill ref** : `mariadb-syntax-sql-ddl`.

---

## Dimension 7 : Functional indexes

- **Rule** : IF an index definition contains a parenthesised expression instead of a bare column list THEN emit a finding at severity BLOCKER.
- **Pattern hint** : match `INDEX`/`KEY` definitions where the index body contains an inner pair of parentheses or an operator, e.g. `KEY idx ((col1 + col2))`, `INDEX idx ((JSON_VALUE(doc,'$.x')))`, `KEY idx ((CAST(...)))`. A plain `KEY idx (col1, col2)` is NOT a finding ; a prefix `KEY idx (col1(20))` is NOT a finding.
- **Divergence** : MySQL 8.0.13+ supports inline functional (expression) indexes. MariaDB does NOT support indexing an expression directly. The MySQL DDL fails to import.
- **Remediation** : replace each functional index with a generated (virtual) column carrying the expression, then index that column. Example : `ALTER TABLE t ADD COLUMN c_calc INT AS (col1 + col2) VIRTUAL, ADD INDEX idx_c_calc (c_calc);`.
- **Skill ref** : `mariadb-syntax-indexing`.

---

## Dimension 8 : Storage engines

- **Rule** : IF an `ENGINE=` clause names an engine that is not guaranteed present on the MariaDB target THEN emit a finding at severity WARNING.
- **Pattern hint** : match `ENGINE=<name>` on CREATE TABLE. Treat `InnoDB`, `Aria`, `MyISAM`, `MEMORY`, `CSV`, `ARCHIVE`, `MRG_MyISAM` as commonly present. Flag `ROCKSDB`, `SPIDER`, `CONNECT`, `S3`, `COLUMNSTORE`, `BLACKHOLE`, `FEDERATED` / `FEDERATEDX`, and any unknown name.
- **Divergence** : engine availability differs between a MySQL source and a MariaDB target. A MySQL-only engine (or a MariaDB plugin engine not installed on the target) makes the CREATE TABLE fail or silently fall back depending on `sql_mode`.
- **Remediation** : confirm the engine plugin is installed on the target (`SHOW ENGINES;` / `INSTALL SONAME`). If the engine has no MariaDB equivalent, convert the table to `InnoDB` before import, or rewrite the DDL.
- **Skill ref** : `mariadb-core-storage-engines`.

---

## Dimension 9 : Window-function frames

- **Rule** : IF a window-function `OVER ()` clause uses a `GROUPS` frame unit THEN emit a finding at severity BLOCKER.
- **Pattern hint** : match `OVER ( ... GROUPS BETWEEN ... )` or `GROUPS UNBOUNDED PRECEDING`. The token `GROUPS` immediately followed by a frame extent inside an `OVER` clause is the trigger. `ROWS` and `RANGE` frames are NOT findings.
- **Divergence** : MariaDB supports `ROWS` and `RANGE` window frames only. `GROUPS` is a MySQL 8 / PostgreSQL feature absent from MariaDB. A view or query using `GROUPS` fails to parse.
- **Remediation** : rewrite the frame using `ROWS` (physical offset) or `RANGE` (logical value offset), whichever preserves the intended semantics. If the original query genuinely depended on peer-group counting, emulate it with `DENSE_RANK()` plus a self-join or a derived table.
- **Skill ref** : `mariadb-syntax-window-and-cte`.

---

## Dimension 10 : Charset

- **Rule** : IF a `CHARACTER SET` / `CHARSET` / `COLLATE` clause uses `utf8` or `utf8mb3` THEN emit a finding at severity SUGGESTION.
- **Pattern hint** : match `CHARSET=utf8`, `CHARACTER SET utf8`, `CHARACTER SET utf8mb3`, or `COLLATE utf8_*` / `COLLATE utf8mb3_*`. Note that bare `utf8` is an alias for `utf8mb3` on both servers.
- **Divergence** : `utf8` / `utf8mb3` is a 3-byte subset of UTF-8 that cannot store emoji or any code point above U+FFFF. Both MySQL 8 and MariaDB recommend `utf8mb4`. The dump imports, but the column carries a latent storage limit.
- **Remediation** : convert tables and columns to `utf8mb4` with an appropriate collation. On MariaDB 11.5+ the default `utf8mb4` collation is `utf8mb4_uca1400_ai_ci` ; through 11.4 it is `utf8mb4_general_ci`. Pick the collation deliberately.
- **Skill ref** : `mariadb-errors-encoding-and-collation`.

---

## Standing post-import step

Regardless of the verdict, the validation report ALWAYS closes with the standing recommendation : run `mariadb-upgrade` exactly once after the import completes. It migrates system tables (including `mysql.user` to `mysql.global_priv` on 10.4+) and runs CHECK TABLE on format drift. NEVER run it twice in a row : the second pass logs spurious warnings that mask real first-pass errors. The full workflow is owned by `mariadb-impl-migration-mysql-to-mariadb`.

---

## Scan procedure summary

1. Parse the input into statements (DDL, DML, user, replication, SET).
2. For each of the 10 dimensions, apply its IF / THEN rule against every relevant statement.
3. Collect one finding row per match : finding, severity, location, divergence, remediation, skill-ref.
4. Sort findings BLOCKER -> WARNING -> SUGGESTION.
5. Compute the verdict : any BLOCKER -> FAIL ; WARNING-only -> PASS WITH WARNINGS ; none -> PASS.
6. Emit the findings table, the verdict line, and the standing `mariadb-upgrade` recommendation.
