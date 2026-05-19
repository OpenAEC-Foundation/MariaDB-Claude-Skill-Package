# Anti-Patterns : MariaDB Defaults and SQL Modes

Production failure modes pulled from MariaDB JIRA, KB pitfalls section, and real upgrade post-mortems. Each entry has the broken code, the failure cause, and the correct alternative.

## AP-01 : Relying on Implicit Default sql_mode

### Broken

```sql
-- App code over years, never specifies sql_mode in my.cnf
-- Runs fine on 10.1, breaks after 10.2.4 upgrade
INSERT INTO orders(amount, ordered_at) VALUES (12345.6789, '0000-00-00');
-- 10.1 : silently inserts 12345.68 and zero-date
-- 10.2.4+ : ERROR 1264 Out of range value AND ERROR 1292 Incorrect date value
```

### Why it fails

The 10.2.4 release flipped the default `sql_mode` from `NO_ENGINE_SUBSTITUTION,NO_AUTO_CREATE_USER` to include `STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO`. Code that silently truncated values now raises. The application config never pinned `sql_mode`, so the upgrade silently changed behaviour.

### Correct

Pin `sql_mode` EXPLICITLY in `my.cnf` to whatever the app was tested against. THEN plan the migration to strict mode as a deliberate, version-controlled change with data cleanup.

```ini
[mysqld]
sql_mode = NO_ENGINE_SUBSTITUTION,NO_AUTO_CREATE_USER
```

After the upgrade is stable, tighten one flag at a time with a deploy + bug-fix cycle.

## AP-02 : Using `utf8` Charset and Hitting Emoji Failure

### Broken

```sql
-- 10.6+ : default install with stock latin1 character_set_server
CREATE TABLE messages (
  id INT PRIMARY KEY AUTO_INCREMENT,
  body VARCHAR(500) CHARACTER SET utf8
) ENGINE=InnoDB;

INSERT INTO messages(body) VALUES ('Hello 👋');
-- ERROR 1366 (HY000): Incorrect string value: '\xF0\x9F\x91\x8B'
```

### Why it fails

`utf8` is an ALIAS for `utf8mb3` in MariaDB by default. `utf8mb3` is the 3-byte subset of UTF-8 covering U+0000 to U+FFFF (the Basic Multilingual Plane). Emoji and most supplementary-plane characters need 4 bytes. The column rejects them.

### Correct

ALWAYS spell out `utf8mb4` in DDL. Never rely on the `utf8` alias.

```sql
CREATE TABLE messages (
  id INT PRIMARY KEY AUTO_INCREMENT,
  body VARCHAR(500) CHARACTER SET utf8mb4 COLLATE utf8mb4_uca1400_ai_ci
) ENGINE=InnoDB;

INSERT INTO messages(body) VALUES ('Hello 👋');  -- success
```

Or set the server default in `my.cnf` and verify with `SHOW CREATE TABLE` after creation.

## AP-03 : Setting `sql_mode='ALL'`

### Broken

```sql
-- Engineer wants "everything strict"
SET GLOBAL sql_mode = 'ALL';
-- ERROR 1231 (42000): Variable 'sql_mode' can't be set to the value of 'ALL'
```

### Why it fails

There is NO `ALL` value for `sql_mode`. The flag list is finite and explicit. `STRICT_ALL_TABLES` is a NAMED flag (applies strict mode to non-transactional engines too), not a wildcard for all flags.

### Correct

Use the composite `TRADITIONAL` for maximum strictness, then add anything else as needed :

```sql
SET GLOBAL sql_mode = 'TRADITIONAL,NO_AUTO_VALUE_ON_ZERO,NO_ZERO_IN_DATE';
-- TRADITIONAL expands to : STRICT_TRANS_TABLES,STRICT_ALL_TABLES,NO_ZERO_IN_DATE,
--                          NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,
--                          NO_ENGINE_SUBSTITUTION,NO_AUTO_CREATE_USER
```

Check the expansion immediately :

```sql
SELECT @@sql_mode;
```

## AP-04 : Expecting `INSERT IGNORE` to Suppress Warnings Under Strict Mode

### Broken

```sql
-- Pre-strict thinking
SET sql_mode = 'STRICT_TRANS_TABLES';
INSERT IGNORE INTO users(age) VALUES (300);
-- 0 rows affected, 1 warning : Out of range value for column 'age'
SHOW WARNINGS;  -- still flagged
```

### Why it fails

Under strict mode `INSERT IGNORE` demotes the ERROR to a WARNING but does NOT make the data valid. The row may be skipped or the value silently coerced (depending on the violation). Application code that checks `affected_rows == 1` and assumes the insert succeeded will be wrong.

### Correct

ALWAYS check `SHOW WARNINGS` after `INSERT IGNORE`, or use explicit validation BEFORE the INSERT :

```sql
-- Pre-validate
SELECT COUNT(*) FROM users WHERE age = 300;  -- 0 means safe to insert

-- Or use ON DUPLICATE KEY UPDATE for the upsert case instead of IGNORE
INSERT INTO users(id, age) VALUES (1, 300)
  ON DUPLICATE KEY UPDATE age = VALUES(age);

-- Or check the warning count
INSERT IGNORE INTO users(age) VALUES (300);
SELECT @@warning_count;  -- if > 0, the insert had problems
```

## AP-05 : Forgetting Procedure-Creation sql_mode Lock-In

### Broken

```sql
-- DBA sets strict mode at runtime, expects it to apply everywhere
SET GLOBAL sql_mode = 'STRICT_TRANS_TABLES,STRICT_ALL_TABLES,NO_ZERO_DATE';

-- A procedure created MONTHS AGO under loose sql_mode still runs lax
CALL legacy_insert_with_zero_date();  -- still silently accepts '0000-00-00'
```

### Why it fails

Stored procedures, functions, triggers, events, and views capture the active `@@session.sql_mode` at CREATE time and store it in routine metadata. Changing GLOBAL sql_mode after CREATE does NOT affect already-defined routines. Verify with :

```sql
SELECT routine_name, sql_mode
FROM information_schema.routines
WHERE routine_schema = DATABASE();
```

### Correct

Audit all routines, drop and recreate any with stale sql_mode, under the new desired session sql_mode :

```sql
SET SESSION sql_mode = 'STRICT_TRANS_TABLES,STRICT_ALL_TABLES,NO_ZERO_DATE';
DROP PROCEDURE legacy_insert_with_zero_date;
DELIMITER //
CREATE PROCEDURE legacy_insert_with_zero_date(...)
BEGIN
  -- body
END //
DELIMITER ;

-- Re-verify
SELECT routine_name, sql_mode FROM information_schema.routines WHERE routine_name = 'legacy_insert_with_zero_date';
```

Automate this in deploy scripts : DROP + CREATE every routine under a known sql_mode at every release.

## AP-06 : Trusting Default `sync_binlog=0` on a Primary

### Broken

```ini
# /etc/mysql/mariadb.conf.d/50-server.cnf : no sync_binlog line
[mysqld]
log_bin = /var/log/mysql/mariadb-bin
# sync_binlog implicit = 0
```

### Why it fails

`sync_binlog=0` means "let the OS decide when to fsync the binlog". On a clean process crash, recent commits may not be in the binlog. On an OS crash or power loss, up to several seconds of committed transactions can vanish from the binlog, breaking replica catch-up and PITR.

### Correct

On any primary serving real traffic, pin `sync_binlog=1` :

```ini
[mysqld]
log_bin                          = /var/log/mysql/mariadb-bin
binlog_format                    = MIXED
sync_binlog                      = 1
innodb_flush_log_at_trx_commit   = 1
expire_logs_days                 = 14
```

`sync_binlog=1` adds one fsync per commit. On NVMe storage this is invisible ; on magnetic disk plan for ~200 commit/sec ceiling and use group-commit (default behaviour).

## AP-07 : Editing my.cnf and Assuming the Running Server Picked It Up

### Broken

```bash
# Engineer edits config
sudo vim /etc/mysql/mariadb.conf.d/50-server.cnf
# Adds : sql_mode = STRICT_TRANS_TABLES,STRICT_ALL_TABLES

# Engineer "verifies" with a SELECT
mariadb -e "SELECT @@sql_mode;"
# Returns the OLD value -- engineer thinks the change failed
```

### Why it fails

mysqld reads option files at STARTUP only. Editing a `.cnf` file changes the persisted config but NOT the live process. Until a restart (or `SET GLOBAL` + `SET PERSIST`), the running value stays at whatever was loaded at boot.

### Correct

After editing the option file, restart the service AND verify :

```bash
sudo systemctl restart mariadb

mariadb -e "SELECT @@sql_mode;"
# Returns the new value

# 10.6+ : also verify WHERE the value came from
mariadb -e "
  SELECT variable_name, variable_source, variable_path
  FROM performance_schema.variables_info
  WHERE variable_name = 'sql_mode';
"
# variable_source : EXPLICIT
# variable_path   : /etc/mysql/mariadb.conf.d/50-server.cnf
```

Alternatively use `SET GLOBAL sql_mode = ...; SET PERSIST sql_mode = ...;` (10.6.1+) to apply live AND persist without manual file editing.

## AP-08 : Hardcoding `ANSI_QUOTES` in App-Layer SQL Without Setting It Server-Side

### Broken

```python
# Python app builds SQL with double-quoted identifiers
cursor.execute('SELECT "customer_id", "name" FROM "customers" WHERE "id" = %s', (cid,))
# Default sql_mode : "customer_id" is a STRING LITERAL, not an identifier
# Result : SELECT 'customer_id', 'name' FROM 'customers' WHERE 'id' = ...
# Syntax error : FROM 'customers' is invalid
```

### Why it fails

By default MariaDB treats `"text"` as a string literal. Only when `ANSI_QUOTES` is in `sql_mode` does `"text"` become an identifier quote (interchangeable with backtick). Stock default sql_mode does NOT include `ANSI_QUOTES`. The app code assumes a non-default mode is active.

### Correct

Pick ONE convention and enforce it server-side :

```ini
# my.cnf : enable ANSI_QUOTES globally
[mysqld]
sql_mode = STRICT_TRANS_TABLES,STRICT_ALL_TABLES,NO_ENGINE_SUBSTITUTION,ANSI_QUOTES
```

OR use backtick in app code (no sql_mode dependency) :

```python
cursor.execute('SELECT `customer_id`, `name` FROM `customers` WHERE `id` = %s', (cid,))
```

Backtick works regardless of sql_mode. ANSI_QUOTES is portable across databases (PostgreSQL, Oracle) but requires the server-side flag to be ON.

## AP-09 : Assuming `default_authentication_plugin` Is ed25519

### Broken

```sql
-- Engineer thinks new users automatically get ed25519
CREATE USER 'app'@'%' IDENTIFIED BY 'secret';

-- Verify
SELECT user, host, plugin FROM mysql.global_priv WHERE user = 'app';
-- plugin : mysql_native_password (SHA-1)
```

### Why it fails

The default `default_authentication_plugin` is still `mysql_native_password` for backward compatibility. The MariaDB project recommends ed25519 in documentation but does NOT flip the default, to avoid breaking older connector libraries. `IDENTIFIED BY 'secret'` uses the default plug-in, which is the legacy SHA-1 hash.

### Correct

ALWAYS specify `IDENTIFIED VIA <plugin>` explicitly for new users :

```sql
-- Load plug-in at startup
-- [mysqld]
-- plugin_load_add = auth_ed25519

INSTALL SONAME 'auth_ed25519';

CREATE USER 'app'@'%' IDENTIFIED VIA ed25519 USING PASSWORD('secret');

-- Migrate existing users
ALTER USER 'old_user'@'%' IDENTIFIED VIA ed25519 USING PASSWORD('new-secret');
```

For audit, list users still on the legacy plug-in :

```sql
SELECT user, host FROM mysql.global_priv
WHERE JSON_EXTRACT(priv, '$.plugin') = '"mysql_native_password"';
```

## AP-10 : Changing `lower_case_table_names` After initdb

### Broken

```ini
# /etc/mysql/my.cnf on an existing Linux server initialized with lower_case_table_names=0
[mysqld]
lower_case_table_names = 1
```

```bash
sudo systemctl restart mariadb
# Service fails to start
# Error log : Different lower_case_table_names settings for server (1) and data dictionary (0).
```

### Why it fails

`lower_case_table_names` is baked into the data dictionary by `mariadb-install-db`. Changing the option-file value after initdb causes a mismatch detected at startup. mysqld refuses to start to prevent corruption.

### Correct

There is NO in-place change. The migration path :

```bash
# 1. Dump everything
mariadb-dump --all-databases --routines --events --triggers > /backup/full.sql

# 2. Stop server, move datadir aside
sudo systemctl stop mariadb
sudo mv /var/lib/mysql /var/lib/mysql-old

# 3. Reinitialize with new value in my.cnf
sudo mariadb-install-db --user=mysql --datadir=/var/lib/mysql

# 4. Start server, restore dump
sudo systemctl start mariadb
mariadb < /backup/full.sql
```

PLAN this BEFORE deciding the OS or naming convention. The decision is locked at install time.
