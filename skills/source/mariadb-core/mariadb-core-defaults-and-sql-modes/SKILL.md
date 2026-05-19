---
name: mariadb-core-defaults-and-sql-modes
description: >
  Use when investigating "my query worked yesterday and now doesn't", upgrading between LTS releases, migrating from MySQL, or setting up a new MariaDB instance with explicit sql_mode and charset.
  Prevents the common mistake of relying on implicit defaults that change between versions, mixing sql_mode-strict with legacy data, or assuming utf8 means utf8mb4.
  Covers sql_mode per version (STRICT_TRANS_TABLES, ANSI_QUOTES, NO_ZERO_DATE, etc.), default server charset_set_server shift from latin1 to utf8mb4 in 11.6, default utf8mb4 collation shift to utf8mb4_uca1400_ai_ci in 11.5, default authentication plug-in evolution, default storage engine (InnoDB), default binlog format MIXED since 10.2.3, my.cnf defaults that break apps on upgrade.
  Keywords: mariadb defaults, sql_mode, STRICT_TRANS_TABLES, ANSI_QUOTES, default charset, utf8mb4, utf8mb4_uca1400, default authentication, default storage engine, my.cnf defaults, why does this query fail now, upgrade broke my app, sql mode change between versions, NO_ZERO_DATE, ONLY_FULL_GROUP_BY, ER_BAD_FIELD_ERROR after upgrade, sync_binlog, binlog_format MIXED, lower_case_table_names, ed25519
license: MIT
compatibility: "Designed for Claude Code. Requires MariaDB 10.6-LTS, 10.11-LTS, 11.x, 12.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# MariaDB Defaults and SQL Modes

How implicit server defaults shift between MariaDB versions, why an unchanged query starts failing after an upgrade, and what every production `my.cnf` MUST pin explicitly. Read this before upgrading LTS lines, migrating from MySQL, or filing a "regression" bug.

## Quick Reference

- MariaDB `sql_mode` default differs per LTS line ; ALWAYS pin `sql_mode` explicitly in `[mysqld]` for production.
- 10.2.4+ default `sql_mode` : `STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION`. Same string applies to 10.6, 10.11, 11.x, 12.x stock builds.
- `utf8` is a 3-byte alias for `utf8mb3`. Use `utf8mb4` for full Unicode (emoji, supplementary planes above U+FFFF).
- TWO SEPARATE default changes : (A) the server default charset `character_set_server` flipped from `latin1` to `utf8mb4` in 11.6 ; (B) the default collation OF the `utf8mb4` charset flipped from `utf8mb4_general_ci` to `utf8mb4_uca1400_ai_ci` one release earlier, in 11.5. Because the server charset stayed `latin1` through 11.5, the effective `collation_server` only becomes `utf8mb4_uca1400_ai_ci` in 11.6 (when the server charset itself becomes `utf8mb4`). 10.6 LTS and 10.11 LTS stock builds still default to `latin1` / `latin1_swedish_ci`. Debian / Ubuntu distro packages override to `utf8mb4`.
- Default `binlog_format` is `MIXED` since 10.2.3 (NOT `ROW`).
- Default `sync_binlog=0` trades durability for write throughput ; set `sync_binlog=1` for crash-safe binlog on a primary.
- Default `innodb_flush_log_at_trx_commit=1` (full ACID).
- Default `default_storage_engine=InnoDB` since 10.2.
- Default authentication plug-in is still `mysql_native_password` (SHA-1) ; ALWAYS choose `ed25519` explicitly for new users (available since 10.1.21).
- `lower_case_table_names` defaults : Linux 0, Windows 1, macOS 2. CANNOT be changed after `mariadb-install-db`.
- `sql_mode` scope is session and global ONLY. There is NO per-database `sql_mode`.
- Stored programs and views lock in the `sql_mode` value that was active at CREATE-time. Later global changes do NOT affect them.
- ALWAYS run `SELECT @@version, @@sql_mode, @@character_set_server, @@collation_server, @@binlog_format, @@sync_binlog;` before declaring a config "the same as before".
- NEVER edit `/etc/my.cnf` and assume the running server picked it up ; mysqld reads option files at startup only.

## Default sql_mode by Version

| Version range | Default sql_mode |
|---|---|
| 10.1.6 and earlier | (empty) |
| 10.1.7 to 10.2.3 | `NO_ENGINE_SUBSTITUTION,NO_AUTO_CREATE_USER` |
| 10.2.4 and later (incl. 10.6, 10.11, 11.x, 12.x) | `STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION` |

Distro packages (Debian, Ubuntu, RHEL) sometimes ship overrides in `/etc/mysql/mariadb.conf.d/*.cnf`. ALWAYS verify with `SELECT @@sql_mode;` on the live instance.

## Decision Tree : Setting sql_mode

1. Are you running a new production instance ?
   - YES : pin `sql_mode='STRICT_TRANS_TABLES,STRICT_ALL_TABLES,NO_ENGINE_SUBSTITUTION,NO_ZERO_DATE,NO_ZERO_IN_DATE,ERROR_FOR_DIVISION_BY_ZERO'` in `[mysqld]`.
   - NO : continue.
2. Are you upgrading from MariaDB <= 10.1 ?
   - YES : explicitly set `sql_mode=''` first, deploy, then enable strict modes one at a time after data validation. Stock 10.2.4+ default flips strict ON.
3. Are you migrating from MySQL 8 ?
   - YES : copy the source `@@sql_mode` exactly (`SELECT @@sql_mode` on MySQL), pin it in MariaDB `[mysqld]`, then plan to remove `ONLY_FULL_GROUP_BY` and `NO_AUTO_VALUE_ON_ZERO` only after auditing app code.
4. Do you need ANSI standard quoting (`"col"` as identifier) ?
   - YES : set `sql_mode='ANSI_QUOTES,...'` GLOBALLY, never per-session, because stored programs capture sql_mode at CREATE time.

## Decision Tree : Default Charset

1. Is the install fresh and < 11.6 ?
   - YES : ALWAYS set in `[mysqld]` :
     ```ini
     character-set-server = utf8mb4
     collation-server = utf8mb4_uca1400_ai_ci
     ```
2. Is the install 11.6+ ?
   - YES : the stock default is already `utf8mb4_uca1400_ai_ci`. Still pin explicitly to survive future default-shifts.
3. Are you migrating an existing DB ?
   - YES : convert per table with `ALTER TABLE t CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_uca1400_ai_ci;`. The server default only affects NEW tables, not existing ones.
4. Do you see `Incorrect string value: '\xF0\x9F...'` errors ?
   - That is the emoji-in-utf8mb3 trap. The column is `utf8` (3-byte), not `utf8mb4`. Convert the column.

## SQL Mode Scope and Stored-Program Lock-In

`sql_mode` exists at TWO scopes only :

- `@@global.sql_mode` : applied to every new connection at login time. Requires SUPER (or SET_USER_ID) privilege to change.
- `@@session.sql_mode` : per-connection override, lasts until disconnect.

There is NO `ALTER DATABASE ... SQL_MODE ...` and NO per-table `sql_mode`. Anyone telling you otherwise is conflating MariaDB with another DBMS.

Stored procedures, functions, triggers, events, and views CAPTURE the current `@@session.sql_mode` value at CREATE time and lock it into the routine metadata. Subsequent `SET GLOBAL sql_mode = ...` does NOT change the routine's effective mode. To rotate a procedure's mode you MUST `DROP` and recreate with the desired session sql_mode active. Verify with :

```sql
SELECT routine_name, sql_mode
FROM information_schema.routines
WHERE routine_schema = DATABASE();
```

## my.cnf Option-File Lookup

MariaDB `mysqld` reads option files at startup in this order on Linux (later files override earlier ones for the same option) :

1. `/etc/my.cnf`
2. `/etc/mysql/my.cnf`
3. `$MYSQL_HOME/my.cnf` (typically `/var/lib/mysql/my.cnf` or `/usr/local/mysql/my.cnf`)
4. The file named by `--defaults-extra-file=<path>` if passed on the command line
5. `~/.my.cnf`
6. `~/.mylogin.cnf` (obfuscated credentials for the `mariadb` client only)

Distro packages add an include directive in `/etc/mysql/my.cnf` :

```ini
!includedir /etc/mysql/mariadb.conf.d/
!includedir /etc/mysql/conf.d/
```

Files in those directories are read in alphabetical order. A `99-custom.cnf` overrides a `50-server.cnf`.

Section selection : `[mysqld]` for the server daemon, `[mariadb]` for MariaDB-specific options ignored by MySQL clients, `[client]` for every official client (`mariadb`, `mariadb-dump`, etc.), `[mariadb-10.11]` for version-specific overrides. The `[server]` group is read by all server-side programs.

ALWAYS reload (`systemctl restart mariadb`) after editing option files ; `SET GLOBAL` changes the running value but does NOT persist across restart unless ALSO written to the option file.

## Default Variables That Bite on Upgrade

| Variable | Default | Why it bites |
|---|---|---|
| `sql_mode` | `STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION` (10.2.4+) | Inserts that worked on 10.1 now fail with `Data too long`, `Incorrect string value`, `Out of range value` |
| `character_set_server` | `latin1` (< 11.6) / `utf8mb4` (11.6+) | Tables created without explicit CHARSET lock to latin1, breaking emoji |
| `collation_server` | `latin1_swedish_ci` (< 11.6) / `utf8mb4_uca1400_ai_ci` (11.6+) | Follows `character_set_server` ; the default collation of `utf8mb4` itself changed from `utf8mb4_general_ci` to `utf8mb4_uca1400_ai_ci` in 11.5. Cross-table JOIN raises `Illegal mix of collations` |
| `binlog_format` | `MIXED` (10.2.3+) | Statement-based subset can drift on non-deterministic UDFs ; row-based applies cleanly |
| `sync_binlog` | `0` | Crash-test reveals up to several seconds of binlog loss |
| `innodb_flush_log_at_trx_commit` | `1` | Setting `2` for throughput silently loses 1s on OS crash |
| `default_storage_engine` | `InnoDB` (10.2+) | A node configured `MyISAM` in `my.cnf` silently creates non-transactional tables |
| `lower_case_table_names` | Linux 0 / Win 1 / macOS 2 | Linux-built dump restored on Windows server breaks on case-folded table names |
| `default_authentication_plugin` | `mysql_native_password` | New users get SHA-1 hashing ; clients with `caching_sha2_password` only refuse to connect |

## Production-Ready `my.cnf` Snippet

```ini
[mysqld]
# Locale and identifiers
character-set-server         = utf8mb4
collation-server             = utf8mb4_uca1400_ai_ci
sql_mode                     = STRICT_TRANS_TABLES,STRICT_ALL_TABLES,NO_ENGINE_SUBSTITUTION,NO_ZERO_DATE,NO_ZERO_IN_DATE,ERROR_FOR_DIVISION_BY_ZERO
default_storage_engine       = InnoDB
innodb_default_row_format    = dynamic

# Durability
sync_binlog                  = 1
innodb_flush_log_at_trx_commit = 1
innodb_doublewrite           = 1

# Binlog
log_bin                      = /var/log/mysql/mariadb-bin
binlog_format                = MIXED
expire_logs_days             = 14

# Auth
plugin_load_add              = auth_ed25519

[client]
default-character-set        = utf8mb4
```

## When Defaults DO Change Within an LTS Line

LTS minor releases occasionally tighten defaults :

- `innodb_buffer_pool_chunk_size` deprecated 10.11.12+.
- `slave_parallel_mode=optimistic` default since 10.5.1+.
- Semi-sync replication built-in (no plug-in install) since 10.3+.

ALWAYS read the release notes for the exact minor version you upgrade to. The LTS guarantee is API/feature stability, not zero-default-change.

## Verification After Any Config Change

```sql
SELECT @@version, @@sql_mode, @@character_set_server, @@collation_server,
       @@binlog_format, @@sync_binlog, @@innodb_flush_log_at_trx_commit,
       @@default_storage_engine, @@lower_case_table_names;
```

Compare the result to your `my.cnf` snippet line-by-line. Drift between the file and the live value means another option file (later in the lookup order) or a `SET GLOBAL` from a previous session overrode it.

## Reference Files

- `references/methods.md` : full sql_mode flag table with version + impact ; charset / collation default matrix per LTS ; option-file section grammar ; system-variable scope semantics.
- `references/examples.md` : 10+ working snippets covering pinning sql_mode, charset migration, procedure lock-in, sync_binlog crash-recovery, my.cnf inheritance.
- `references/anti-patterns.md` : 8 production failure modes with WHY-it-fails and the corrected pattern.

## Sources

- MariaDB KB `sql-mode` : https://mariadb.com/docs/server/server-management/variables-and-modes/sql_mode
- Character set overview : https://mariadb.com/kb/en/character-set-and-collation-overview/
- Unicode / utf8 alias : https://mariadb.com/kb/en/unicode/
- Replication and binary log variables : https://mariadb.com/kb/en/replication-and-binary-log-system-variables/
- InnoDB system variables : https://mariadb.com/kb/en/innodb-system-variables/
- ed25519 authentication : https://mariadb.com/kb/en/authentication-plugin-ed25519/
- Identifier case sensitivity : https://mariadb.com/kb/en/identifier-case-sensitivity/
- Binary log formats : https://mariadb.com/kb/en/binary-log-formats/
