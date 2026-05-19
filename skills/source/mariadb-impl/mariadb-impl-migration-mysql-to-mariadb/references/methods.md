# Methods : MySQL to MariaDB Migration

Reference detail for the migration skill. All version annotations apply to MariaDB 10.6-LTS, 10.11-LTS, 11.x, 12.x unless stated otherwise.

## 1. Complete Compatibility Matrix

### 1.1 JSON storage and validation

| Aspect | MySQL 5.7.8+ / 8.x | MariaDB 10.2.7+ |
|--------|---------------------|------------------|
| Storage format | Native binary (packed) | LONGTEXT alias |
| Insert-time validation | Enforced | None unless CHECK constraint |
| Index on JSON path | Functional index on expression | Virtual column + index |
| `JSON_VALID` function | Returns 1 or 0 | Returns 1 or 0 |
| Comparison semantics | JSON-aware (numeric vs string distinguished) | String comparison |

Remediation : `ALTER TABLE t ADD CONSTRAINT chk_t_col_json CHECK (JSON_VALID(col));` per JSON column post-restore. For path indexing : `ALTER TABLE t ADD COLUMN key_col TYPE AS (JSON_VALUE(col, '$.path')) VIRTUAL, ADD INDEX (key_col);`.

### 1.2 GTID format (L-004)

| Aspect | MySQL 5.6+ | MariaDB 10.0.2+ |
|--------|-------------|------------------|
| Format | `uuid:seqno`, e.g. `3e11fa47-71ca-11e1-9e33-c80aa9429562:23` | `domain-server-sequence`, e.g. `0-1-1234` |
| System variable | `gtid_executed`, `gtid_purged` | `gtid_binlog_pos`, `gtid_slave_pos`, `gtid_current_pos` |
| CHANGE MASTER flag | `MASTER_AUTO_POSITION=1` | `MASTER_USE_GTID=slave_pos|current_pos|no` |
| Interop | uuid-based | domain-based |
| Replication MySQL to MariaDB | n/a | YES, but use POSITIONAL (`MASTER_USE_GTID=NO`) |
| Replication MariaDB to MySQL | NOT SUPPORTED | n/a |

Implication : a cut-over window can use MariaDB-replica-of-MySQL with binlog position. NEVER attempt continuous GTID-tracked replication.

### 1.3 Authentication plug-ins

| Plug-in | MySQL 8 default | Available in MariaDB | Recommended replacement |
|---------|-----------------|----------------------|--------------------------|
| `caching_sha2_password` | yes | NO | `ed25519` (preferred) or `mysql_native_password` |
| `mysql_native_password` | available (deprecated 8.4) | yes (default 10.4-10.10), still supported 10.11+ / 11.x / 12.x | Keep for legacy drivers |
| `sha256_password` | available | NO | `ed25519` |
| `ed25519` | NOT AVAILABLE | yes (INSTALL SONAME 'auth_ed25519') | Native MariaDB choice |
| `unix_socket` | as `auth_socket` | yes (default for root on Debian/Ubuntu) | Local-only |

### 1.4 mysql.user vs mysql.global_priv

- Before MariaDB 10.4 : privileges live in `mysql.user` (same as MySQL 5.x / 8.x).
- From 10.4+ : privileges live in `mysql.global_priv`. `mysql.user` is retained as a backward-compatible VIEW for tooling that greps the legacy table.
- `mariadb-upgrade` migrates the storage. NEVER skip it after a binary swap.
- For inspection prefer `SELECT user, host, plugin, JSON_DETAILED(priv) FROM mysql.global_priv;`.

### 1.5 sql_mode defaults

| Version | Default sql_mode (subset) |
|---------|---------------------------|
| MySQL 5.7 | `ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION` |
| MySQL 8.0 | same as 5.7 plus `NO_AUTO_CREATE_USER` removal |
| MySQL 8.4 | similar, with continued tightening |
| MariaDB 10.2.4+ | `STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION` |
| MariaDB 10.11+ / 11.x / 12.x | same as 10.2.4+ baseline |

Action : after migration, explicitly SET the desired sql_mode in `my.cnf` rather than relying on the implicit default. Do NOT carry over MySQL `ONLY_FULL_GROUP_BY` to MariaDB unless apps already comply.

### 1.6 Role syntax

| Operation | MySQL 8.x | MariaDB 10.0.5+ |
|-----------|-----------|------------------|
| Create role | `CREATE ROLE r;` | `CREATE ROLE r;` |
| Grant role to user | `GRANT r TO 'u'@'h';` | `GRANT r TO 'u'@'h';` |
| Activate at session | `SET ROLE r;` | `SET ROLE r;` |
| Set default role | `ALTER USER 'u'@'h' DEFAULT ROLE r1, r2;` | `SET DEFAULT ROLE r FOR 'u'@'h';` (ONE role per statement) |
| Show grants of role | `SHOW GRANTS FOR r;` | `SHOW GRANTS FOR r;` |

Note : MariaDB enforces one default role per user ; granted secondary roles must be activated per session.

### 1.7 Sequence syntax (MariaDB only, 10.3+)

```sql
-- 10.3+ / 10.6+ / 10.11+ / 11.x / 12.x
CREATE SEQUENCE order_seq
  START WITH 1000
  INCREMENT BY 1
  MINVALUE 1000
  MAXVALUE 9999999999
  CACHE 50
  NOCYCLE;

SELECT NEXT VALUE FOR order_seq;

INSERT INTO orders (id, payload)
  VALUES (NEXT VALUE FOR order_seq, '{"k":"v"}');
```

MySQL has no equivalent ; `AUTO_INCREMENT` is the only MySQL idiom. During migration, keep MySQL's AUTO_INCREMENT as-is. Convert to sequences AFTER the migration is stable, only if there is a business reason.

### 1.8 INVISIBLE vs IGNORED INDEX (L-006)

| MySQL 8.0+ | MariaDB 10.6+ |
|-------------|----------------|
| `ALTER TABLE t ALTER INDEX idx INVISIBLE;` | `ALTER TABLE t ALTER INDEX idx IGNORED;` |
| `ALTER TABLE t ALTER INDEX idx VISIBLE;` | `ALTER TABLE t ALTER INDEX idx NOT IGNORED;` |
| `CREATE INDEX idx ON t (col) INVISIBLE;` | not supported inline ; create then ALTER IGNORED |

### 1.9 Query cache

- MySQL 8.0 : removed.
- MariaDB through 10.11 : present (`query_cache_type`, `query_cache_size`) but discouraged on write-heavy workloads.
- MariaDB 11.0+ : REMOVED. Remove `query_cache_*` from my.cnf or startup fails.

### 1.10 Binary names (10.5+)

- `mysqldump` is renamed to `mariadb-dump`. Symlink remains today.
- `mysqlbinlog` is renamed to `mariadb-binlog`. Symlink remains today.
- `mysql` CLI is renamed to `mariadb`. Symlink remains.
- Production cron jobs : update to the new names ; do NOT rely on symlinks indefinitely.

## 2. mariadb-upgrade Reference

### 2.1 Purpose

After a binary upgrade (MySQL to MariaDB, or MariaDB minor or major), mariadb-upgrade :

- Updates system tables in the `mysql` schema (adds new fields, migrates `mysql.user` to `mysql.global_priv` on 10.4+).
- Runs CHECK TABLE on all tables and ALTER TABLE on any flagged for format-drift.
- Updates the `mysql.event` table live (no restart needed).

ALWAYS run exactly ONCE per binary swap. NEVER run twice in a row : the second pass logs spurious warnings that mask real first-pass errors.

### 2.2 Flag reference

| Flag | Description |
|------|-------------|
| `-?, --help` | Show help |
| `--check-if-upgrade-is-needed` | Quick check ; returns 0 if upgrade needed, 1 if not |
| `-f, --force` | Force run even if version check passes |
| `-h, --host <host>` | Server host (default localhost) |
| `-u, --user <user>` | DB user (typically root) |
| `-p, --password` | Prompt for password |
| `-P, --port <port>` | TCP port |
| `-S, --socket <path>` | Unix socket path |
| `--protocol <p>` | tcp / socket / pipe / memory |
| `--ssl` | Enable TLS |
| `--ssl-ca <path>` | CA cert |
| `--ssl-cert <path>` | Client cert |
| `--ssl-key <path>` | Client key |
| `-s, --upgrade-system-tables` | Update only the `mysql` schema tables ; skip CHECK TABLE on user tables |
| `-v, --verbose` | Increase verbosity (repeatable for more detail) |
| `--silent` | Reduce output |
| `-k, --version-check` | Verify program / server version match before running |
| `--write-binlog` | Log the schema-modification commands to the binary log (default OFF) |
| `-t, --tmpdir <dir>` | Temp dir for sort and copy |

### 2.3 Standard invocation

```bash
# Standard post-binary-swap upgrade
mariadb-upgrade -u root -p

# Faster, system-only path (skips user-table CHECK)
mariadb-upgrade -u root -p --upgrade-system-tables

# Force re-run after a partial failure (rare)
mariadb-upgrade -u root -p --force

# Replication-aware run (write changes to binlog so they propagate to replicas)
mariadb-upgrade -u root -p --write-binlog
```

### 2.4 What mariadb-upgrade does NOT do

- It does NOT migrate `caching_sha2_password` users to ed25519. You MUST do that manually.
- It does NOT add `CHECK (JSON_VALID(col))` to JSON columns. You MUST sweep manually.
- It does NOT rewrite `INVISIBLE INDEX` to `IGNORED INDEX`. Those statements fail at the binary swap if present in stored DDL ; you MUST rewrite manually.

## 3. JSON Re-validation Script Template

```sql
-- 10.6+ / 10.11+ / 11.x / 12.x
-- Generate the ALTER statements for every JSON column in the current database,
-- then review and run them.

SELECT CONCAT(
  'ALTER TABLE `', TABLE_SCHEMA, '`.`', TABLE_NAME,
  '` ADD CONSTRAINT chk_', TABLE_NAME, '_', COLUMN_NAME, '_json',
  ' CHECK (JSON_VALID(`', COLUMN_NAME, '`));'
) AS ddl
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA = DATABASE()
  AND DATA_TYPE = 'longtext'
  AND COLUMN_TYPE = 'longtext'
  -- Heuristic : only apply where the column was JSON in MySQL.
  -- For a clean migration, prefer to derive this list from the original MySQL information_schema.
  AND COLUMN_NAME REGEXP '^(payload|data|attributes|json_|meta)$';
```

Note : on a MySQL 5.7+ source, `information_schema.COLUMNS.DATA_TYPE='json'` is the authoritative list ; dump that list BEFORE the migration so the post-migration sweep targets exactly those columns.

## 4. User Migration Script Template

```sql
-- 10.6+ / 10.11+ / 11.x / 12.x
-- Install ed25519 plug-in (one-off)
INSTALL SONAME 'auth_ed25519';

-- For each application user, drop and re-create with ed25519.
-- The application MUST be ready to send the new password.

DROP USER IF EXISTS 'app'@'%';
CREATE USER 'app'@'%' IDENTIFIED VIA ed25519 USING PASSWORD('NEW_STRONG_SECRET');
GRANT SELECT, INSERT, UPDATE, DELETE ON appdb.* TO 'app'@'%';

-- For service accounts that cannot rotate passwords immediately,
-- fall back to mysql_native_password (compatible with MySQL drivers).

DROP USER IF EXISTS 'svc'@'10.0.%';
CREATE USER 'svc'@'10.0.%' IDENTIFIED WITH mysql_native_password BY 'TRANSITIONAL_SECRET';
GRANT SELECT ON appdb.* TO 'svc'@'10.0.%';
```

Pre-hashed ed25519 (no plain-text password in DDL) :

```sql
-- One-off : load the password-hashing UDF
CREATE FUNCTION ed25519_password RETURNS STRING SONAME 'auth_ed25519.so';

-- Compute hash from an interactive shell, then create user with the hash
SELECT ed25519_password('NEW_STRONG_SECRET');
-- copy the returned hash, then :
CREATE USER 'app'@'%'
  IDENTIFIED VIA ed25519 USING 'ZIgUREUg5PVgQ6LskhXmO+eZLS0nC8be6HPjYWR4YJY';
```

## 5. my.cnf Migration Checklist

| Setting | Action |
|---------|--------|
| `default_authentication_plugin=caching_sha2_password` | REMOVE. Not supported in MariaDB. |
| `query_cache_type=ON`, `query_cache_size=...` | REMOVE if targeting MariaDB 11.0+. Keep but reconsider on 10.x. |
| `sql_mode=...` | Re-evaluate. MariaDB defaults differ. Set explicitly. |
| `character_set_server=utf8mb4` | KEEP. Strongly recommended on both. |
| `collation_server` | Re-evaluate. MariaDB 10.10+ supports `utf8mb4_uca1400_*` (modern UCA). |
| `innodb_buffer_pool_size` | Keep value, but note MariaDB 10.11.12+ deprecates `innodb_buffer_pool_chunk_size`. |
| `innodb_default_row_format=dynamic` | KEEP (Frappe / ERPNext requirement). |
| `binlog_format=ROW` | KEEP if using row-based replication. MariaDB default is MIXED since 10.2.4. |
| `gtid_mode=ON`, `enforce_gtid_consistency=ON` | REMOVE. MariaDB has no `gtid_mode`. Use `gtid_strict_mode` and `MASTER_USE_GTID`. |
| `caching_sha2_password_*` variables | REMOVE. |
| `plugin_load_add=auth_ed25519` | ADD for persistent ed25519 availability. |

## 6. Validation After Migration

```sql
-- Privilege store check (10.4+)
SELECT user, host, plugin
FROM mysql.global_priv;

-- Plug-in availability
SELECT plugin_name, plugin_status
FROM information_schema.plugins
WHERE plugin_name IN ('ed25519', 'mysql_native_password', 'unix_socket');

-- JSON validation coverage
SELECT TABLE_NAME, COLUMN_NAME
FROM information_schema.CHECK_CONSTRAINTS
WHERE CONSTRAINT_SCHEMA = DATABASE()
  AND CHECK_CLAUSE LIKE '%json_valid%';

-- Server version + edition confirmation
SELECT VERSION(), @@version_comment;
```

## 7. Sources

- `https://mariadb.com/kb/en/upgrading-from-mysql-to-mariadb/`
- `https://mariadb.com/docs/release-notes/community-server/about/compatibility-and-differences/mariadb-vs-mysql-compatibility`
- `https://mariadb.com/docs/release-notes/community-server/about/compatibility-and-differences/mariadb-vs-mysql-features`
- `https://mariadb.com/docs/server/clients-and-utilities/deployment-tools/mariadb-upgrade`
- `https://mariadb.com/kb/en/authentication-plugin-ed25519/`
- `https://mariadb.com/kb/en/ignored-indexes/`
- `https://mariadb.com/kb/en/create-sequence/`
- Vooronderzoek `docs/research/vooronderzoek-mariadb.md` §6
