# MariaDB Version Matrix : Methods Reference

Authoritative reference for MariaDB LTS dates, feature-introduction matrix, breaking
changes per major version, and the `mariadb-upgrade` tool.

All entries verified against `mariadb.org/about` and `mariadb.com/kb` on 2026-05-19.

---

## 1. Release Cadence and EOL Table

MariaDB versioning : `major.minor.patch`. One minor per major is designated LTS. The
remaining minors are interim / short-support (~1 year). LTS gets ~5 years community,
optionally extended via paid Enterprise / Extended support.

### Long-Term Support (LTS) releases

| Version | GA Date | Community EOL | Enterprise EOL | Extended EOL | Notes |
|---------|---------|---------------|----------------|--------------|-------|
| 10.6 | 2021-07-06 | 2026-07-06 | 2028-08-23 | 2029-08-23 | First LTS with atomic DDL, IGNORED indexes, JSON_TABLE |
| 10.11 | 2023-02-16 | 2028-02-16 | 2028-02-16 | 2028-02-16 | utf8mb4_uca1400 collation family, system_versioning_insert_history |
| 11.4 | 2024-05-29 | 2029-05-29 | 2030-01-16 | 2033-01-16 | First LTS of 11.x ; query cache fully removed (11.0+) |
| 11.8 | 2025-06-04 | 2028-06-04 | 2030-10-22 | 2033-10-22 | Frappe v16 baseline |

Source : `https://mariadb.org/about/#maintenance-policy` (verified 2026-05-19).

### Interim / short-support releases

Each interim minor receives ~1 year of community bug-fix support and then EOL.

| Version | GA Date | Status | Use case |
|---------|---------|--------|----------|
| 10.7 | 2022-02 | EOL | Replaced by 10.11 LTS |
| 10.8 | 2022-05 | EOL | Replaced by 10.11 LTS ; introduced descending indexes |
| 10.9 | 2022-08 | EOL | Replaced by 10.11 LTS ; auto-partition by SYSTEM_TIME INTERVAL |
| 10.10 | 2022-11 | EOL | Replaced by 10.11 LTS |
| 11.0 | 2023-06 | EOL | Query cache removed, optimizer rewrite |
| 11.1 | 2023-08 | EOL | Online ALTER for partitioned tables |
| 11.2 | 2023-11 | EOL | RAND_BYTES, PROC_HANDLER_DIAGNOSTICS |
| 11.3 | 2024-02 | EOL | Vector preview, SHOW ANALYZE FORMAT=JSON |
| 11.5 | 2024-08 | EOL | Bundled vector functions (preview) |
| 11.6 | 2024-11 | EOL | Default charset utf8mb4 on stock build |
| 11.7 | 2025-02 | EOL | Vector indexes GA preview |
| 12.0 | 2025-08 | Interim | file_key_management_use_pbkdf2 ; wait for LTS designation |
| 12.x next | TBD | Interim | Dev only ; future LTS |

NEVER pick an interim release for production. The skip-pattern between LTS releases
exists exactly because interim minors carry the cost of being on the bleeding edge
without the support guarantee.

### How to verify support state

```sql
SELECT VERSION();                  -- e.g. 11.4.4-MariaDB
SELECT @@version_compile_machine;  -- e.g. x86_64
SELECT @@version_compile_os;       -- e.g. Linux
```

Then map VERSION() against the table above. Anything past Community EOL is unsupported
and must not run unpatched in production.

---

## 2. Feature-Introduction Matrix (Top 30)

Rows = feature, columns = "first MariaDB version that shipped it". A cell of "10.6+"
means the feature is available in 10.6 LTS and all later versions. "10.6 LTS" without
the plus marks the feature as backported to the LTS baseline.

| Feature | First version | Brief note | Source |
|---------|---------------|------------|--------|
| Window functions (ROWS / RANGE frames) | 10.2+ | GROUPS frame NOT supported (see L-002) | KB `window-functions` |
| Recursive CTEs (WITH RECURSIVE) | 10.2.2+ | SQL:1999 syntax | KB `with` |
| CHECK constraints (enforced) | 10.2.1+ | Before 10.2.1 they were parsed but not enforced | KB `constraint` |
| Default row format DYNAMIC for InnoDB | 10.2+ | `innodb_default_row_format=DYNAMIC` | KB `innodb-row-formats` |
| Built-in JSON functions (alias of LONGTEXT) | 10.2+ | JSON column type added 10.2 ; storage is LONGTEXT (L-005) | KB `json-data-type` |
| Generated columns (VIRTUAL / PERSISTENT) | 10.2+ | Indexable ; functional indexes via VIRTUAL columns | KB `generated-columns` |
| System-versioned tables | 10.3+ | WITH SYSTEM VERSIONING ; SQL:2011 (MariaDB-only) | KB `system-versioned-tables` |
| Sequences (CREATE SEQUENCE) | 10.3+ | Oracle / PostgreSQL syntax (MariaDB-only) | KB `create-sequence` |
| Invisible columns | 10.3+ | SELECT * skips ; explicit name shows (MariaDB-only) | KB `invisible-columns` |
| INSTANT ADD COLUMN | 10.3+ | End-of-table only in 10.3 ; any-position in 10.4+ | KB `instant-add-column-for-innodb` |
| Built-in semi-synchronous replication | 10.3+ | Removed need for separate plug-in install | KB `semisynchronous-replication` |
| mysql.global_priv (replaces mysql.user) | 10.4+ | mysql.user remains as compatibility view | KB `grant` |
| Application-time periods (PERIOD FOR) | 10.4+ | UPDATE / DELETE FOR PORTION OF (MariaDB-only) | KB `application-time-periods` |
| INSTANT ADD COLUMN at any position | 10.4+ | Removed end-of-table restriction | KB `instant-add-column-for-innodb` |
| INSTANT DROP COLUMN | 10.4+ | Metadata-only column drop | KB `alter-table` |
| ed25519 authentication plugin | 10.1.21+ | Default recommendation over mysql_native_password | KB `authentication-plugin-ed25519` |
| Bitemporal tables (system + application time) | 10.5+ | Combination of WITH SYSTEM VERSIONING + PERIOD FOR | KB `application-time-periods` |
| RETURNING on INSERT | 10.5+ | DELETE RETURNING since 10.0 ; UPDATE RETURNING still not supported through 11.8 | KB `insert-returning` |
| mariadb-dump / mariadb-binlog tool renames | 10.5+ | Old mysqldump / mysqlbinlog kept as compatibility symlinks | KB `mariadb-dump` |
| slave_parallel_mode=optimistic default | 10.5.1+ | Default changed ; pre-10.5.1 default was conservative | KB `parallel-replication` |
| Atomic DDL (crash-safe CREATE / DROP / ALTER) | 10.6+ | Via binary log ; no orphan .frm | KB `atomic-ddl` |
| IGNORED indexes | 10.6+ | Maintained but invisible to optimizer | KB `ignored-indexes` |
| JSON_TABLE | 10.6+ | Projects JSON arrays into virtual rows | KB `json_table` |
| Default tmp encryption variables stabilised | 10.6+ | encrypt_tmp_files, encrypt_binlog mature | KB `data-at-rest-encryption` |
| Descending indexes (CREATE INDEX ... DESC) | 10.8+ | Avoids backward scan for ORDER BY col DESC | KB `descending-indexes` |
| INTERVAL ... AUTO partition by SYSTEM_TIME | 10.9.1+ | Auto-partition creation for history | KB `system-versioned-tables` |
| system_versioning_insert_history variable | 10.11+ | Allows direct insertion of history rows | KB `system-versioned-tables` |
| utf8mb4_uca1400 collation family | 10.11+ | Unicode 14.0 ; default on 11.6+ stock builds | KB `unicode-collation-algorithm` |
| Query cache REMOVED | 11.0+ | Deprecated since 10.1.7 ; gone from 11.0 onwards | KB `query-cache` |
| InnoDB undo-tablespace consolidation | 11.0+ | innodb_undo_tablespaces removed | KB `innodb-undo-log` |
| Default charset utf8mb4 on stock build | 11.6+ | LTS 10.6 / 10.11 still ship latin1 default on stock | KB `character-set-and-collation-overview` |
| file_key_management_use_pbkdf2 | 12.0.1+ | PBKDF2 iterations and digest selectable | KB `file-key-management-encryption-plugin` |

### Features that DO NOT exist in MariaDB

| Missing feature | Equivalent / workaround |
|-----------------|--------------------------|
| Window-function `GROUPS` frame | Use `ROWS` or `RANGE` only (L-002) |
| Materialized views | `CREATE TABLE AS SELECT` + EVENT-scheduled refresh |
| Native binary JSON storage | JSON is LONGTEXT alias ; add `CHECK (JSON_VALID(col))` (L-005) |
| MySQL-style GTID (`uuid:seqno`) | MariaDB GTID is `domain-server-sequence` ; not interconvertible (L-004) |
| `NULLS FIRST / NULLS LAST` in ORDER BY | Use `ORDER BY col IS NULL, col` |
| UPDATE RETURNING | Two-statement workaround (UPDATE then SELECT) |

---

## 3. Breaking-Changes Per Major Version

### 10.6 to 10.11

- **utf8mb4_uca1400 collations introduced** as recommended default. Existing tables
  on `utf8mb4_general_ci` or `utf8mb4_unicode_ci` keep working ; new objects should
  use `utf8mb4_uca1400_ai_ci`. Cross-collation joins emit `Illegal mix of collations`.
- **SUPER privilege split** into fine-grained privileges (BINLOG ADMIN, REPLICATION
  CLIENT ADMIN, FEDERATED ADMIN, etc.) progressively from 10.5 ; 10.11 completes the
  matrix. Scripts using `GRANT SUPER ON *.* TO ...` should be reviewed.
- **mariadb-* tool renames complete** : `mysqldump` -> `mariadb-dump`, `mysqlbinlog`
  -> `mariadb-binlog`, `mysql` -> `mariadb`. Old names remain as symlinks but cron
  scripts should be migrated.
- **system_versioning_insert_history (10.11+)** : new variable that allows direct
  insertion of historical rows into system-versioned tables for data migration.

### 10.11 to 11.x (especially 11.4)

- **Query cache fully removed (11.0+)**. Deprecated since 10.1.7, removed in 11.0.
  Configuration entries `query_cache_type`, `query_cache_size`, `query_cache_limit`
  emit "Unknown system variable" on 11.x. Remove from `my.cnf` before upgrade.
- **InnoDB undo-tablespace consolidation** : `innodb_undo_tablespaces` removed, undo
  logs unified inside the system tablespace plus per-instance separate files.
- **Default authentication plugin** : recommended switch to `ed25519` ; old
  `mysql_native_password` still works.
- **innodb_buffer_pool_chunk_size deprecation (10.11.12+ and 11.x)** : variable
  ignored, buffer pool resizes in 1 MB increments up to `innodb_buffer_pool_size_max`.

### 11.4 to 11.8

- **Vector support GA** : new VECTOR data type, distance functions, vector indexes.
  No removal of existing features ; pure addition.
- **Default charset utf8mb4 on stock builds (since 11.6)** : `CREATE TABLE` without
  explicit charset now gets `utf8mb4` instead of `latin1`. Schemas relying on the
  implicit latin1 default break on charset-sensitive joins.

### 11.8 to 12.x

- **file_key_management_use_pbkdf2 (12.0.1+)** : key derivation now supports PBKDF2
  with selectable iterations and digest function. New deployments should enable.
- 12.x is still pre-LTS at time of writing ; treat all changes as preview.

---

## 4. mariadb-upgrade Tool Reference

`mariadb-upgrade` (10.5+, formerly `mysql_upgrade`) is the post-binary-swap utility
that brings system tables, stored routines, and views up to the format expected by
the new server.

### What it does

1. Checks all tables in all databases for incompatibilities with the new server.
2. Upgrades the `mysql.*` system tables : grant tables, time-zone tables, performance
   schema definitions.
3. Recreates stored procedures, functions, triggers and views to the new format.
4. Updates the `mysql_upgrade_info` file with the new version-marker.

### Invocation

```bash
# Standard invocation, requires SUPER (or equivalent fine-grained) privilege
mariadb-upgrade --user=root --password

# Skip table checks (faster, riskier ; use only when you have a full backup)
mariadb-upgrade --user=root --password --upgrade-system-tables

# Force re-run (only if you must ; usually unwanted)
mariadb-upgrade --user=root --password --force
```

### Rules

- ALWAYS run exactly ONCE after a binary upgrade.
- NEVER run twice in a row. It is idempotent but the second pass spews warnings that
  hide real first-pass errors.
- ALWAYS take a `mariadb-dump --all-databases --single-transaction` backup BEFORE
  the upgrade. Restoration from binlog alone is not a substitute.
- ALWAYS verify with `SELECT VERSION();` after the run that the server reports the
  expected new version.
- ALWAYS read `SHOW WARNINGS;` post-run. Any non-empty warning list deserves an
  explicit fix before the system is returned to traffic.

### Version-jump rule

Upgrade one major version at a time. Path :

```
10.4 (EOL) -> 10.5 -> 10.6 LTS -> 10.11 LTS -> 11.4 LTS -> 11.8 LTS -> next LTS
```

Skipping a major step (e.g. 10.6 directly to 11.4) is unsupported and will leave
system tables in an inconsistent state. Path is justified per `mariadb.com/kb/en/upgrading/`
which documents upgrade guides ONLY for adjacent steps : 10.6 to 10.11, 10.11 to 11.4,
11.4 to 11.8. No 10.6-to-11.4 guide exists.

---

## 5. Cross-Reference

- `mariadb-core-architecture` : engine architecture, threading model
- `mariadb-core-storage-engines` : engine choice (InnoDB / Aria / ColumnStore / Spider)
- `mariadb-core-replication-model` : asynchronous / semi-sync / Galera trade-offs
- `mariadb-core-security-model` : privilege model, ed25519, encryption plugin
- `mariadb-core-defaults-and-sql-modes` : sql_mode, charset defaults, timezone
- `mariadb-impl-migration-mysql-to-mariadb` : MySQL-to-MariaDB conversion
