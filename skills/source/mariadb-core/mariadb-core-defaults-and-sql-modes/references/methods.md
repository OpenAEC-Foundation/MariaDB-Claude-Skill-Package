# Methods : MariaDB Defaults and SQL Modes

Full reference for sql_mode flags, default values per LTS line, option-file grammar, and system-variable scope.

## sql_mode Flag Reference

Verified against `https://mariadb.com/docs/server/server-management/variables-and-modes/sql_mode`. Each row : flag, introduced version, meaning, practical impact when ON.

| Flag | Introduced | Meaning | Practical Impact |
|---|---|---|---|
| `STRICT_TRANS_TABLES` | 5.0 | Aborts invalid statements on transactional engines (InnoDB, Aria) | `INSERT` with out-of-range or too-long value FAILS instead of truncating |
| `STRICT_ALL_TABLES` | 5.0 | Aborts invalid statements on every engine (incl. MyISAM) | Same as above, plus MyISAM no longer silently truncates |
| `ERROR_FOR_DIVISION_BY_ZERO` | 5.0 | Division by zero returns error instead of NULL | `SELECT 1/0` raises `ER_DIVISION_BY_ZERO` instead of returning NULL |
| `NO_AUTO_CREATE_USER` | 5.0 | `GRANT` cannot implicitly create user | Must `CREATE USER` before `GRANT` |
| `NO_ENGINE_SUBSTITUTION` | 5.0 | Unknown / disabled engine raises error instead of falling back to InnoDB | `CREATE TABLE ... ENGINE=Aria` FAILS if Aria plug-in not loaded |
| `NO_ZERO_DATE` | 5.0 | Rejects `'0000-00-00'` literal | Legacy data with all-zero date FAILS to INSERT |
| `NO_ZERO_IN_DATE` | 5.0 | Rejects dates with zero month OR day (`'2024-00-15'`, `'2024-03-00'`) | Same as above for partial-zero |
| `ONLY_FULL_GROUP_BY` | 5.0 | Every non-aggregated column in `SELECT` must appear in `GROUP BY` | `SELECT a, b, COUNT(*) FROM t GROUP BY a` FAILS unless `b` is functionally dependent on `a` |
| `ANSI_QUOTES` | 5.0 | `"col"` treated as identifier quote (like backtick) | String literals MUST use single-quote `'string'` ; backtick still works |
| `PIPES_AS_CONCAT` | 5.0 | `||` operator becomes string concat (like Oracle) instead of logical OR | `SELECT 'a' || 'b'` returns `'ab'` ; logical OR must use `OR` keyword |
| `REAL_AS_FLOAT` | 5.0 | `REAL` type alias maps to `FLOAT` instead of `DOUBLE` | Affects DDL only |
| `IGNORE_SPACE` | 5.0 | Space allowed between function name and `(` ; function names become reserved | `SELECT COUNT (*)` parses ; identifier `count` now reserved |
| `NO_BACKSLASH_ESCAPES` | 5.0 | Backslash is a literal character, not escape | `'\\n'` is two characters, not newline |
| `HIGH_NOT_PRECEDENCE` | 5.0 | `NOT a BETWEEN b AND c` parses as `(NOT a) BETWEEN b AND c` | Pre-SQL-92 precedence ; rarely useful |
| `PAD_CHAR_TO_FULL_LENGTH` | 5.1 | `CHAR(n)` retrieval keeps trailing spaces instead of trimming | Application sees right-padded strings |
| `EMPTY_STRING_IS_NULL` | 10.1 | `''` as INSERT value treated as NULL | Empty string columns become NULL |
| `SIMULTANEOUS_ASSIGNMENT` | 10.3.5 | `UPDATE t SET a=b, b=a` swaps atomically | MariaDB-specific |
| `TIME_ROUND_FRACTIONAL` | 10.4 | Fractional seconds rounded instead of truncated | `TIME(3) := '12:34:56.7894'` becomes `12:34:56.789` |
| `ANSI` (composite) | 5.0 | Sets `REAL_AS_FLOAT,PIPES_AS_CONCAT,ANSI_QUOTES,IGNORE_SPACE` | Plus changes default transaction isolation ; rare in practice |
| `TRADITIONAL` (composite) | 5.0 | Sets `STRICT_TRANS_TABLES,STRICT_ALL_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION,NO_AUTO_CREATE_USER` | Maximum strictness ; recommended for new production |
| `ORACLE` (composite) | 10.3 | Sets `PIPES_AS_CONCAT,ANSI_QUOTES,IGNORE_SPACE,NO_KEY_OPTIONS,NO_TABLE_OPTIONS,NO_FIELD_OPTIONS,NO_AUTO_CREATE_USER,SIMULTANEOUS_ASSIGNMENT` and enables PL/SQL parser | Use only when supporting Oracle migrations |
| `MYSQL323` | 5.0 | MySQL 3.23 compatibility (sets `HIGH_NOT_PRECEDENCE`) | Legacy ; do NOT use |
| `MYSQL40` | 5.0 | MySQL 4.0 compatibility | Legacy ; do NOT use |
| `NO_AUTO_VALUE_ON_ZERO` | 5.0 | `INSERT ... AUTO_INCREMENT col VALUES (0)` keeps the 0 instead of generating next value | MySQL-compatibility helper for dumps with explicit zero |
| `NO_KEY_OPTIONS` / `NO_FIELD_OPTIONS` / `NO_TABLE_OPTIONS` | 5.0 | `SHOW CREATE TABLE` omits MariaDB-specific options | Used to produce MySQL-compatible dumps |

## Composite Mode Expansion

When `sql_mode` includes a composite (`TRADITIONAL`, `ANSI`, `ORACLE`), MariaDB expands it to the individual flags AT SET-time. After `SET sql_mode='TRADITIONAL'`, `SELECT @@sql_mode` returns the expanded comma-separated flag list, NOT the literal string `TRADITIONAL`.

There is NO `sql_mode='ALL'` value. Setting `sql_mode='ALL'` produces `ER_WRONG_VALUE_FOR_VAR : Variable 'sql_mode' can't be set to the value of 'ALL'`. The closest practical equivalent is `TRADITIONAL`.

## Default sql_mode by Version

| Version | Default Value |
|---|---|
| 5.5, 10.0, 10.1.0 to 10.1.6 | (empty string) |
| 10.1.7 to 10.2.3 | `NO_ENGINE_SUBSTITUTION,NO_AUTO_CREATE_USER` |
| 10.2.4 to 12.x (current) | `STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION` |

The 10.2.4 flip is the most consequential default change in the project's history. Any code that worked silently on 10.1 should be tested under 10.2.4+ before assuming compatibility.

## sql_mode Scope Semantics

Two scopes exist :

- `@@global.sql_mode` : new connections inherit this at login. Requires SUPER (or SET_USER_ID 10.5+).
- `@@session.sql_mode` : per-connection override. Lost at disconnect.

There is NO `sql_mode` at database, table, column, role, or user level. Anyone claiming `ALTER DATABASE ... SQL_MODE=...` exists is wrong.

Stored programs (procedures, functions, triggers, events, views) bind the active `@@session.sql_mode` at CREATE time into routine metadata. Verify with :

```sql
SELECT routine_name, sql_mode FROM information_schema.routines WHERE routine_schema=DATABASE();
SELECT trigger_name, sql_mode FROM information_schema.triggers WHERE trigger_schema=DATABASE();
SELECT event_name,  sql_mode FROM information_schema.events  WHERE event_schema=DATABASE();
SELECT table_name,  sql_mode FROM information_schema.views   WHERE table_schema=DATABASE();
```

Rotation requires DROP + CREATE under the new session sql_mode.

## Default Character Set and Collation

There are TWO separate default-changes, one release apart. Do not conflate them :

- FACT A : the server-wide default charset `character_set_server` flipped from `latin1` to `utf8mb4` in **MariaDB 11.6**.
- FACT B : the default collation OF the `utf8mb4` charset flipped from `utf8mb4_general_ci` to `utf8mb4_uca1400_ai_ci` in **MariaDB 11.5** (one release earlier).

| Version | character_set_server | collation_server | Default collation of the utf8mb4 charset |
|---|---|---|---|
| 5.5 to 11.4 (incl. 10.6 LTS, 10.11 LTS) | `latin1` | `latin1_swedish_ci` | `utf8mb4_general_ci` |
| 11.5 | `latin1` | `latin1_swedish_ci` | `utf8mb4_uca1400_ai_ci` |
| 11.6 and later | `utf8mb4` | `utf8mb4_uca1400_ai_ci` | `utf8mb4_uca1400_ai_ci` |

The effective `collation_server` value only becomes `utf8mb4_uca1400_ai_ci` in 11.6, because that is when the server charset itself becomes `utf8mb4`. In 11.5 the server charset is still `latin1`, so `collation_server` is still `latin1_swedish_ci` ; only an explicit `CHARSET=utf8mb4` table or column picks up the new `utf8mb4_uca1400_ai_ci` collation default on 11.5.

Distro packages (Debian, Ubuntu) ship `/etc/mysql/mariadb.conf.d/50-server.cnf` overrides that set `character-set-server=utf8mb4` regardless of upstream default. ALWAYS verify with `SELECT @@character_set_server, @@collation_server;`.

`utf8` is an ALIAS in MariaDB. It maps to `utf8mb3` by default. `old_mode` can force `utf8` to mean `utf8mb4` (UTF8_IS_UTF8MB4), but relying on this is fragile. ALWAYS spell out `utf8mb4` in DDL.

- `utf8mb3` : 1 to 3 bytes per character ; CANNOT store anything above U+FFFF (no emoji, no supplementary planes).
- `utf8mb4` : 1 to 4 bytes per character ; full Unicode.

Common collations on `utf8mb4` :

| Collation | Sort order | Accent / case |
|---|---|---|
| `utf8mb4_uca1400_ai_ci` | Unicode CLDR 14.0.0 | accent-insensitive, case-insensitive (default collation of the utf8mb4 charset since 11.5) |
| `utf8mb4_uca1400_as_cs` | Unicode CLDR 14.0.0 | accent-sensitive, case-sensitive |
| `utf8mb4_bin` | binary | byte-for-byte ; fastest, least useful for human text |
| `utf8mb4_general_ci` | legacy MySQL | accent-insensitive ; do NOT use for new schemas (incorrect German sharp s, Turkish dotted i) |
| `utf8mb4_unicode_ci` | UCA 4.0.0 | older Unicode collation algorithm |

## Default Server-System Variables (Verified)

| Variable | Default | Verified Source |
|---|---|---|
| `default_storage_engine` | `InnoDB` (since 10.2) | KB system-variables |
| `binlog_format` | `MIXED` (since 10.2.3) | KB replication-and-binary-log-system-variables |
| `sync_binlog` | `0` | KB replication-and-binary-log-system-variables |
| `innodb_flush_log_at_trx_commit` | `1` | KB innodb-system-variables |
| `innodb_doublewrite` | `1` | KB innodb-system-variables |
| `innodb_default_row_format` | `dynamic` | KB innodb-row-formats |
| `lower_case_table_names` | Linux `0`, Windows `1`, macOS `2` | KB identifier-case-sensitivity |
| `default_authentication_plugin` | `mysql_native_password` | KB authentication-plugin-ed25519 |
| `tx_isolation` / `transaction_isolation` | `REPEATABLE-READ` | KB system-variables |
| `autocommit` | `ON` | KB system-variables |
| `event_scheduler` | `OFF` | KB event-scheduler |

`lower_case_table_names` CANNOT be changed after `mariadb-install-db` has created the system tables. The value is baked into the data dictionary. Changing it post-initdb corrupts table-name resolution.

`sync_binlog=0` is the default for write throughput. On a primary, set `sync_binlog=1` to get crash-safe binlog (matches MySQL group-commit semantics with InnoDB).

## Default Authentication Plug-In

Default plug-in is still `mysql_native_password` (SHA-1 hash). This is intentional for backward-compatibility with older clients and connector libraries.

Recommended alternative since 10.1.21 : `ed25519` (EdDSA, same algorithm as OpenSSH). Stronger than SHA-1, no salt-replay attack surface.

```sql
-- Install plug-in once
INSTALL SONAME 'auth_ed25519';

-- Or load at startup via my.cnf
-- [mysqld]
-- plugin_load_add = auth_ed25519

-- Create user with ed25519
CREATE USER 'alice'@'%' IDENTIFIED VIA ed25519 USING PASSWORD('secret');
```

Other plug-ins available on stock MariaDB :

- `unix_socket` : root login maps to OS uid (default on Debian / Ubuntu for root@localhost).
- `gssapi` : Kerberos for enterprise SSO.
- `pam` : OS PAM stack.
- `named_pipe` : Windows-only IPC.

## my.cnf Option-File Grammar

### Lookup Order (Linux)

mysqld reads in this order ; later files override earlier for the same option key :

1. `/etc/my.cnf`
2. `/etc/mysql/my.cnf`
3. `$MYSQL_HOME/my.cnf` (typically `/var/lib/mysql/my.cnf`)
4. `--defaults-extra-file=<path>` if passed on command line
5. `~/.my.cnf`
6. `~/.mylogin.cnf` (encrypted, mariadb client only)

### Lookup Order (Windows)

1. `%PROGRAMDATA%\MySQL\MySQL Server <version>\my.ini`
2. `%WINDIR%\my.ini` / `%WINDIR%\my.cnf`
3. `C:\my.ini` / `C:\my.cnf`
4. `<installation-dir>\my.ini`
5. `--defaults-extra-file=<path>` if passed

### Includes

```ini
!includedir /etc/mysql/mariadb.conf.d/
!include    /etc/mysql/custom.cnf
```

`!includedir` reads files matching `*.cnf` in ALPHABETICAL order. Higher numeric prefix wins (`99-custom.cnf` overrides `50-server.cnf`).

### Section Groups

| Section | Read by |
|---|---|
| `[mysqld]` | The server daemon (mysqld, mariadbd) |
| `[server]` | All server-side programs (mysqld + mariadb-install-db) |
| `[mariadb]` | MariaDB-specific server options ; MySQL daemons ignore |
| `[mariadb-10.11]` | Only when running 10.11.x ; version-specific |
| `[client]` | All official clients (mariadb, mariadb-dump, mariadb-admin) |
| `[mysql]` / `[mariadb-client]` | Specific client only |
| `[mysqldump]` / `[mariadb-dump]` | Specific tool only |
| `[client-server]` | Both clients and server |

### Option Value Quoting

- Bare token : `sql_mode = STRICT_TRANS_TABLES,NO_ZERO_DATE`
- Quoted (preserves spaces, commas inside quotes treated literally) : `sql_mode = "STRICT_TRANS_TABLES,NO_ZERO_DATE"`
- Multi-line option NOT supported ; one option per line.

### Reload Semantics

Editing an option file changes the on-disk config but NOT the running server. To apply :

- Restart service : `systemctl restart mariadb`
- Persist runtime change to file : `SET PERSIST <var> = <value>;` (10.6.1+ ; writes `/var/lib/mysql/mysqld-auto.cnf`).
- Verify with `SHOW VARIABLES LIKE '<var>';` after restart.

`SET PERSIST_ONLY <var> = <value>;` writes to file without changing the live value (useful when the change requires restart anyway).

## Inspecting Effective Defaults

```sql
-- Live values right now
SELECT @@version, @@version_compile_os,
       @@sql_mode, @@global.sql_mode,
       @@character_set_server, @@collation_server,
       @@binlog_format, @@sync_binlog,
       @@innodb_flush_log_at_trx_commit,
       @@default_storage_engine,
       @@lower_case_table_names,
       @@default_authentication_plugin;

-- Where was a variable last set
SELECT * FROM information_schema.system_variables
WHERE variable_name = 'sql_mode';

-- 10.6+ : per-variable source provenance (option-file path, command-line, runtime)
SELECT variable_name, variable_source, variable_path
FROM performance_schema.variables_info
WHERE variable_name IN ('sql_mode','character_set_server','binlog_format','sync_binlog');
```
