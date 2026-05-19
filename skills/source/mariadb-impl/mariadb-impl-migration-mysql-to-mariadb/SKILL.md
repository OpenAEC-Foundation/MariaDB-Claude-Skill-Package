---
name: mariadb-impl-migration-mysql-to-mariadb
description: >
  Use when migrating a MySQL instance to MariaDB, comparing compatibility per MySQL version, identifying syntax divergences,
  planning replication bridging, re-validating JSON data after the binary swap, or running mariadb-upgrade to finalise the migration.
  Prevents the common mistake of expecting GTID continuity (incompatible per L-004), assuming JSON storage is the same
  (LONGTEXT-alias per L-005 and D-010), copying MySQL 8 INVISIBLE INDEX syntax (MariaDB uses IGNORED per L-006),
  keeping caching_sha2_password users (no MariaDB plug-in), skipping mariadb-upgrade after the binary swap, or copying my.cnf verbatim across the divide.
  Covers per-MySQL-version compatibility matrix (5.6 / 5.7 / 8.0 / 8.4), JSON storage divergence and CHECK (JSON_VALID(col)) remediation,
  GTID-format divergence (one-way dump-and-load only), authentication-plugin shift, mysql.user vs mysql.global_priv (10.4+),
  sql_mode default differences, role syntax differences, MariaDB sequence syntax (10.3+), INVISIBLE vs IGNORED INDEX rename,
  the mariadb-upgrade workflow, and end-to-end migration steps (in-place upgrade vs dump-and-restore).
  Keywords: migration, MySQL to MariaDB, drop-in compatibility, JSON divergence, JSON_VALID, CHECK constraint,
  GTID incompatible, domain-server-sequence, mysql.user, mysql.global_priv, mariadb-upgrade, mysql_upgrade,
  caching_sha2_password, ed25519, mysql_native_password, INSTALL SONAME auth_ed25519, role syntax, SET DEFAULT ROLE,
  sequence syntax, CREATE SEQUENCE, NEXT VALUE FOR, AUTO_INCREMENT, sql_mode default, STRICT_TRANS_TABLES,
  INVISIBLE INDEX, IGNORED INDEX, query cache removed 11.0, my.cnf, mariadb-dump, mysqldump rename,
  how do I migrate from MySQL, can mariadb replicate to mysql, can mysql connect to mariadb, replication broken after upgrade,
  why does INVISIBLE INDEX fail, why does caching_sha2_password fail, my JSON column got corrupted after migration
license: MIT
compatibility: "Designed for Claude Code. Requires MariaDB 10.6-LTS, 10.11-LTS, 11.x, 12.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# MariaDB : Migration from MySQL to MariaDB

Deterministic guide for in-place upgrades and dump-and-restore migrations from MySQL 5.6 / 5.7 / 8.0 / 8.4 to MariaDB 10.6-LTS, 10.11-LTS, 11.x, 12.x. Covers the compatibility matrix, the four hard divergences (JSON, GTID, auth-plug-in, INVISIBLE vs IGNORED INDEX), the privilege-store migration (mysql.user to mysql.global_priv), and the mandatory mariadb-upgrade workflow.

## Quick Reference

- MariaDB JSON is a LONGTEXT alias, NOT binary like MySQL 5.7.8+. After migration, ALWAYS add `CHECK (JSON_VALID(col))` to every JSON column to preserve the MySQL validation invariant. Without it, a subsequent non-JSON UPDATE corrupts the row silently. (D-010, L-005)
- MariaDB GTID format is `domain-server-sequence`. MySQL GTID is `uuid:seqno`. They are INCOMPATIBLE. MySQL to MariaDB is a one-way dump-and-load, NOT a continuous GTID-tracked replication. (L-004)
- MySQL 8 default auth plug-in is `caching_sha2_password`. MariaDB does NOT ship it. ALWAYS re-create users with `ed25519` or `mysql_native_password` before pointing apps at MariaDB.
- MariaDB uses `ALTER TABLE t ALTER INDEX idx IGNORED`. MySQL 8 uses `INVISIBLE`. Copying MySQL 8 syntax produces a syntax error. (L-006)
- ALWAYS run `mariadb-upgrade` exactly once after the binary swap. It migrates `mysql.user` to `mysql.global_priv` (10.4+) and runs CHECK TABLE / ALTER TABLE on format-drift. NEVER run it twice in a row : the second pass logs spurious warnings that mask real first-pass errors. (Vooronderzoek §6)
- NEVER copy a MySQL `my.cnf` verbatim. `sql_mode` defaults differ, `query_cache_*` was REMOVED in MariaDB 11.0+, `default_authentication_plugin` is unsupported, and several InnoDB variables have different default scaling.
- MySQL 5.6 is a near drop-in replacement for MariaDB 10.x (in-place upgrade works). MySQL 5.7+ requires JSON-validation remediation. MySQL 8.0+ requires user re-creation. MySQL 8.4 additionally removes the query cache (MariaDB also removed it in 11.0+).
- MariaDB has `CREATE SEQUENCE` + `NEXT VALUE FOR` (10.3+) that MySQL lacks. Migrating FROM MySQL : keep AUTO_INCREMENT as-is, NEVER auto-convert to sequences during migration. Convert deliberately after stability.
- Binary names changed in 10.5+ : `mysqldump` is now `mariadb-dump`, `mysqlbinlog` is `mariadb-binlog`. Old names remain as compatibility symlinks today, but cron jobs that hard-code `mysqldump` SHOULD be updated.
- Verify privileges after migration via `SELECT user, host, plugin FROM mysql.global_priv;` (10.4+), NOT only `mysql.user`. The `mysql.user` view is still queryable as a backward-compat layer but does not reflect role grants natively.

## Decision Tree : Which Migration Path

```
START
 |
 +-- Source is MySQL 5.5 or 5.6 ?
 |       --> in-place upgrade : stop mysqld, install mariadb-server on same datadir, start mariadbd, run mariadb-upgrade.
 |       --> validate apps, then sweep JSON columns to add CHECK (JSON_VALID(col)) where appropriate.
 |
 +-- Source is MySQL 5.7 ?
 |       --> in-place upgrade is supported but JSON columns need explicit re-validation (LONGTEXT alias does NOT enforce JSON).
 |       --> after mariadb-upgrade, add CHECK (JSON_VALID(col)) on every JSON column.
 |       --> re-test app for any sha256_password users (rare in 5.7).
 |
 +-- Source is MySQL 8.0 or 8.4 ?
 |       --> dump-and-restore is the SAFE default. In-place is technically possible but high-risk due to data-dictionary differences.
 |       --> mysqldump --single-transaction --routines --triggers --events --add-drop-database from MySQL 8.
 |       --> stop MySQL, install MariaDB on a clean datadir, restore the dump.
 |       --> drop all caching_sha2_password users, re-create with ed25519 or mysql_native_password.
 |       --> add CHECK (JSON_VALID(col)) on every JSON column.
 |       --> if app uses INVISIBLE INDEX : rewrite to IGNORED INDEX.
 |       --> if app uses ALTER USER ... DEFAULT ROLE : rewrite to MariaDB SET DEFAULT ROLE syntax.
 |       --> run mariadb-upgrade once to finalise system tables.
 |
 +-- Need continuous replication during the cut-over window ?
 |       --> NEVER assume MySQL-GTID continuity (incompatible). Use binlog-positional replication MariaDB-reads-from-MySQL during the window only.
 |       --> CHANGE MASTER TO MASTER_HOST='mysql-source', MASTER_LOG_FILE='...', MASTER_LOG_POS='...', MASTER_USE_GTID=NO.
 |       --> at cut-over, stop apps, wait for replica lag = 0, STOP SLAVE, repoint apps to MariaDB. Then run mariadb-upgrade.
 |
 +-- Need to keep MySQL running as a fallback for N days ?
 |       --> set up MariaDB-replica-of-MySQL as above, but do NOT add a reverse MariaDB-to-MySQL path : MySQL cannot replicate FROM MariaDB.
 |       --> the rollback plan is "repoint apps back to MySQL" without re-syncing, accepting data loss for the window.
END
```

## Compatibility Matrix : MySQL Version vs Required Remediation

| Area | MySQL 5.6 | MySQL 5.7 | MySQL 8.0 | MySQL 8.4 |
|------|-----------|-----------|-----------|-----------|
| In-place upgrade viable | yes (best path) | yes | risky | risky |
| Dump-and-restore viable | yes | yes | yes (recommended) | yes (recommended) |
| JSON re-validation needed | not applicable (no native JSON pre-5.7.8) | yes (LONGTEXT alias) | yes | yes |
| Default auth plug-in needs migration | no | rare (sha256_password) | yes (caching_sha2_password) | yes |
| INVISIBLE INDEX rename | not applicable | not applicable | yes (to IGNORED) | yes |
| Role syntax adaptation | not applicable | not applicable | yes (DEFAULT ROLE) | yes |
| sql_mode review needed | yes (defaults differ) | yes | yes | yes |
| Query cache removal | no | no | no | n/a (MariaDB 11.0+ has no cache) |

## The Four Hard Divergences

### 1. JSON Storage (D-010, L-005)

MariaDB `JSON` is a LONGTEXT alias with an optional `CHECK (JSON_VALID(col))` hook. MySQL 5.7.8+ stores JSON in a packed binary format that validates on every write. After migration, raw INSERT or UPDATE on a JSON column accepts non-JSON text silently. ALWAYS add the CHECK :

```sql
-- After dump-restore, on every JSON column
-- 10.6+ / 10.11+ / 11.x / 12.x
ALTER TABLE orders
  ADD CONSTRAINT orders_payload_json CHECK (JSON_VALID(payload));
```

To index a JSON path, MariaDB requires a virtual column + index (NO functional indexes on JSON expressions directly) :

```sql
-- 10.6+ / 10.11+ / 11.x / 12.x
ALTER TABLE orders
  ADD COLUMN buyer_id VARCHAR(36)
    AS (JSON_VALUE(payload, '$.buyer_id')) VIRTUAL,
  ADD INDEX idx_orders_buyer (buyer_id);
```

### 2. GTID Format (L-004)

MariaDB : `domain-server-sequence`, e.g. `0-1-1234`. MySQL : `uuid:seqno`, e.g. `3e11fa47-71ca-11e1-9e33-c80aa9429562:23`. They are NOT interconvertible. During a cut-over, MariaDB can replicate FROM MySQL using BINLOG POSITION only (`MASTER_USE_GTID=NO`), and MySQL CANNOT replicate from MariaDB at all. Plan one-shot dump-and-load for two-way scenarios.

### 3. Authentication Plug-in

MySQL 8 default `caching_sha2_password` is not in MariaDB. MariaDB native options are `mysql_native_password` (compatible), `ed25519` (recommended, OpenSSH-grade), and `unix_socket` (local-only). After migration, scan and re-create :

```sql
-- 10.6+ / 10.11+ / 11.x / 12.x
INSTALL SONAME 'auth_ed25519';     -- one-off; or set plugin_load_add=auth_ed25519 in my.cnf
CREATE OR REPLACE USER 'app'@'%'
  IDENTIFIED VIA ed25519 USING PASSWORD('new-strong-secret');
GRANT SELECT, INSERT, UPDATE, DELETE ON appdb.* TO 'app'@'%';
```

### 4. INVISIBLE vs IGNORED INDEX (L-006)

```sql
-- WRONG (MySQL 8 syntax, fails in MariaDB)
ALTER TABLE t ALTER INDEX idx INVISIBLE;

-- RIGHT (MariaDB 10.6+)
ALTER TABLE t ALTER INDEX idx IGNORED;
```

## Step-by-Step : In-Place Upgrade (MySQL 5.6 / 5.7 to MariaDB)

1. Verify backup is fresh and restorable. `mariabackup --backup --target-dir=/backup/pre-migration --user=root` (or equivalent mysqldump --single-transaction --routines).
2. Stop the MySQL service : `systemctl stop mysqld`.
3. Remove MySQL packages WITHOUT removing the datadir : `apt-get remove --purge mysql-server mysql-server-core-*` or `yum remove mysql-server`.
4. Install MariaDB pointing at the SAME datadir : `apt-get install mariadb-server` (Debian/Ubuntu) or `yum install MariaDB-server` (RHEL family).
5. Start MariaDB : `systemctl start mariadb`.
6. Run `mariadb-upgrade` exactly ONCE : `mariadb-upgrade -u root -p`. This migrates system tables, including `mysql.user` to `mysql.global_priv` on 10.4+.
7. Validate apps. Run smoke tests.
8. Sweep JSON columns and add `CHECK (JSON_VALID(col))` where appropriate (see Divergence 1).
9. If users on `sha256_password` (rare in 5.7) : re-create as `ed25519` or `mysql_native_password`.

## Step-by-Step : Dump-and-Restore (MySQL 8.0 / 8.4 to MariaDB)

1. On MySQL : `mysqldump --single-transaction --routines --triggers --events --add-drop-database --databases db1 db2 > dump.sql`.
2. Provision a clean MariaDB datadir. Install MariaDB server, start it.
3. On MariaDB : restore the dump : `mariadb -u root -p < dump.sql`. Expect occasional warnings on `caching_sha2_password` user statements ; ignore and re-create users in step 6.
4. Run `mariadb-upgrade -u root -p` once.
5. Sweep JSON columns and add `CHECK (JSON_VALID(col))`.
6. Drop and re-create all app users with `ed25519` (or `mysql_native_password` if drivers are old). See Divergence 3.
7. Search dumped DDL / app code for `INVISIBLE INDEX` and rewrite to `IGNORED INDEX`. See Divergence 4.
8. Search dumped DDL / app code for `ALTER USER ... DEFAULT ROLE r1, r2` and rewrite to MariaDB : `SET DEFAULT ROLE r1 FOR 'user'@'host'` (one role per statement).
9. Review `my.cnf` : remove `default_authentication_plugin`, remove `query_cache_*` if targeting 11.0+, re-review `sql_mode`.
10. Repoint apps. Monitor `SHOW PROCESSLIST` and `SHOW ENGINE INNODB STATUS` during ramp-up.

## Cross-References

- Schema-design with JSON CHECK constraints : `mariadb-impl-schema-design`
- JSON syntax patterns : `mariadb-syntax-json`
- Replication-positional bridging during cut-over : `mariadb-impl-replication-setup`
- Index syntax including IGNORED : `mariadb-syntax-indexing`
- Encoding and collation pitfalls discovered post-migration : `mariadb-errors-encoding-and-collation`
- User and privilege model post-10.4+ : `mariadb-core-security-model`

## References

- `references/methods.md` : full compatibility matrix per area, mariadb-upgrade flag reference, user-migration script template, JSON re-validation script template
- `references/examples.md` : 10+ end-to-end migration scripts (in-place 5.7, dump-and-restore 8.0, JSON CHECK sweep, ed25519 re-creation, GTID-positional bridge, INVISIBLE rename, sequence introduction post-stability, one-shot MariaDB-to-MySQL export)
- `references/anti-patterns.md` : 8 documented anti-patterns with why-it-fails and the correct alternative

## Source Verification

- `https://mariadb.com/kb/en/upgrading-from-mysql-to-mariadb/` : in-place upgrade workflow, drop-in compatibility statement for MySQL 5.5 / 5.6
- `https://mariadb.com/docs/release-notes/community-server/about/compatibility-and-differences/mariadb-vs-mysql-compatibility` : JSON storage difference, GTID interop limit, system table differences
- `https://mariadb.com/docs/release-notes/community-server/about/compatibility-and-differences/mariadb-vs-mysql-features` : MariaDB-only features (SEQUENCE 10.3+, virtual columns 5.2+, system-versioned tables 10.3+, dynamic columns 5.3+)
- `https://mariadb.com/docs/server/clients-and-utilities/deployment-tools/mariadb-upgrade` : mariadb-upgrade purpose, flags (--force, --upgrade-system-tables, --check-if-upgrade-is-needed, --version-check, --write-binlog), when to run
- `https://mariadb.com/kb/en/authentication-plugin-ed25519/` : ed25519 plug-in INSTALL SONAME, CREATE USER VIA ed25519 USING PASSWORD syntax
- Vooronderzoek `docs/research/vooronderzoek-mariadb.md` §6 (Migration MySQL to MariaDB) and §1 / §3 cross-cuts
- LESSONS L-004 (GTID incompatible), L-005 (JSON LONGTEXT alias), L-006 (IGNORED vs INVISIBLE), L-001 (KB path redirects), L-007 (per-statement version verification)
- DECISIONS D-010 (LONGTEXT-alias must appear in Quick Reference of every JSON-touching skill)
