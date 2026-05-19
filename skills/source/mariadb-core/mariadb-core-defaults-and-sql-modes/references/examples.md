# Examples : MariaDB Defaults and SQL Modes

Working snippets verified against MariaDB 10.6, 10.11, 11.x, 12.x. Every example is version-annotated and assumes a connection as a SUPER-privileged user unless noted.

## Example 1 : Inspect Current sql_mode

```sql
-- 10.2.4+ : both global and session-scope
SELECT @@global.sql_mode  AS global_mode,
       @@session.sql_mode AS session_mode;

-- Also useful : version and origin
SELECT @@version, @@version_compile_os;
```

Expected output on a stock 10.11 install :

```
global_mode  : STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION
session_mode : STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION
```

If global and session diverge, somewhere in the session a `SET SESSION sql_mode = ...` was issued.

## Example 2 : Set sql_mode for Current Session

```sql
-- 10.6+ : add a flag without losing existing ones
SET SESSION sql_mode = CONCAT(@@session.sql_mode, ',ONLY_FULL_GROUP_BY');

-- Replace entirely (loses previous flags)
SET SESSION sql_mode = 'STRICT_TRANS_TABLES,STRICT_ALL_TABLES,NO_ENGINE_SUBSTITUTION,NO_ZERO_DATE,NO_ZERO_IN_DATE,ERROR_FOR_DIVISION_BY_ZERO';

-- Empty for compatibility shimming during migration
SET SESSION sql_mode = '';
```

The CONCAT pattern is the safe way to add ONE flag without resetting unrelated flags that the app might rely on.

## Example 3 : Pin sql_mode Globally and in my.cnf

```sql
-- Live change, all new sessions inherit
SET GLOBAL sql_mode = 'STRICT_TRANS_TABLES,STRICT_ALL_TABLES,NO_ENGINE_SUBSTITUTION,NO_ZERO_DATE,NO_ZERO_IN_DATE,ERROR_FOR_DIVISION_BY_ZERO';

-- 10.6.1+ : persist to disk without manual file edit
SET PERSIST sql_mode = 'STRICT_TRANS_TABLES,STRICT_ALL_TABLES,NO_ENGINE_SUBSTITUTION,NO_ZERO_DATE,NO_ZERO_IN_DATE,ERROR_FOR_DIVISION_BY_ZERO';
```

Equivalent manual `my.cnf` entry :

```ini
[mysqld]
sql_mode = STRICT_TRANS_TABLES,STRICT_ALL_TABLES,NO_ENGINE_SUBSTITUTION,NO_ZERO_DATE,NO_ZERO_IN_DATE,ERROR_FOR_DIVISION_BY_ZERO
```

Then `sudo systemctl restart mariadb` and verify with Example 1.

## Example 4 : Procedure-Creation sql_mode Lock-In

```sql
-- 10.6+ : current session has loose sql_mode
SET SESSION sql_mode = '';

DELIMITER //
CREATE PROCEDURE add_user(IN p_name VARCHAR(100), IN p_age INT)
BEGIN
  INSERT INTO users(name, age) VALUES (p_name, p_age);
END //
DELIMITER ;

-- Verify : procedure has the EMPTY sql_mode locked in
SELECT routine_name, sql_mode
FROM information_schema.routines
WHERE routine_schema = DATABASE() AND routine_name = 'add_user';

-- Even if we now flip global to strict, the procedure still runs under empty mode
SET GLOBAL sql_mode = 'STRICT_TRANS_TABLES,STRICT_ALL_TABLES';

-- This INSERT INSIDE the procedure does NOT raise on out-of-range, because the
-- procedure was created with empty sql_mode
CALL add_user('Alice', 999999999999);  -- silently truncates instead of erroring

-- To fix : drop and recreate under strict mode
DROP PROCEDURE add_user;
SET SESSION sql_mode = 'STRICT_TRANS_TABLES,STRICT_ALL_TABLES';
DELIMITER //
CREATE PROCEDURE add_user(IN p_name VARCHAR(100), IN p_age INT)
BEGIN
  INSERT INTO users(name, age) VALUES (p_name, p_age);
END //
DELIMITER ;
```

This is THE single biggest source of "the procedure works on dev and fails on prod" surprises. ALWAYS check `information_schema.routines.sql_mode` before chasing the bug elsewhere.

## Example 5 : Migration from Charset latin1 to utf8mb4

```sql
-- 10.6+ : inspect current per-table charset
SELECT t.table_schema, t.table_name, ccsa.character_set_name, t.table_collation
FROM information_schema.tables t
JOIN information_schema.collation_character_set_applicability ccsa
  ON ccsa.collation_name = t.table_collation
WHERE t.table_schema NOT IN ('mysql','information_schema','performance_schema','sys')
  AND ccsa.character_set_name <> 'utf8mb4';

-- Per-table conversion (rewrites all rows ; lock table)
ALTER TABLE customer CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_uca1400_ai_ci;

-- Default for NEW tables in this database
ALTER DATABASE app DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_uca1400_ai_ci;

-- Server-level default (affects only databases created after this point)
SET GLOBAL character_set_server = 'utf8mb4';
SET GLOBAL collation_server     = 'utf8mb4_uca1400_ai_ci';
```

The server-level change is NOT retroactive. Existing tables keep their original charset until ALTER-converted.

## Example 6 : Demonstrating the utf8 vs utf8mb4 Trap

```sql
-- 10.6+ : create with the legacy alias
CREATE TABLE chat_legacy (msg VARCHAR(200) CHARACTER SET utf8) ENGINE=InnoDB;

-- Try to store a 4-byte emoji
INSERT INTO chat_legacy(msg) VALUES ('Hello 😀');
-- ERROR 1366 (HY000): Incorrect string value: '\xF0\x9F\x98\x80...' for column 'msg' at row 1

-- Now the corrected table
CREATE TABLE chat_modern (msg VARCHAR(200) CHARACTER SET utf8mb4) ENGINE=InnoDB;
INSERT INTO chat_modern(msg) VALUES ('Hello 😀');  -- success

-- Verify the actual stored encoding
SELECT column_name, character_set_name, collation_name
FROM information_schema.columns
WHERE table_name IN ('chat_legacy','chat_modern');
```

`utf8` is `utf8mb3` (3-byte max) ; emoji are 4-byte sequences and cannot be stored. This is the most common production charset bug.

## Example 7 : Per-Procedure sql_mode at Definition Time

```sql
-- 10.6+ : route different procedures with different modes
SET SESSION sql_mode = 'STRICT_TRANS_TABLES,STRICT_ALL_TABLES';
DELIMITER //
CREATE PROCEDURE strict_insert(IN p_val INT)
BEGIN
  INSERT INTO numbers(val) VALUES (p_val);
END //
DELIMITER ;

SET SESSION sql_mode = '';
DELIMITER //
CREATE PROCEDURE lax_insert(IN p_val INT)
BEGIN
  INSERT INTO numbers(val) VALUES (p_val);
END //
DELIMITER ;

-- Verify each procedure's locked mode
SELECT routine_name, sql_mode
FROM information_schema.routines
WHERE routine_schema = DATABASE();

-- strict_insert : STRICT_TRANS_TABLES,STRICT_ALL_TABLES
-- lax_insert   : (empty)
```

Each procedure runs under its OWN locked sql_mode regardless of caller.

## Example 8 : Crash-Safe Binlog with sync_binlog and innodb_flush_log_at_trx_commit

```sql
-- 10.6+ : audit current durability settings
SELECT @@sync_binlog, @@innodb_flush_log_at_trx_commit, @@innodb_doublewrite;

-- For a primary that must survive OS crash with zero binlog loss
SET GLOBAL sync_binlog                  = 1;
SET GLOBAL innodb_flush_log_at_trx_commit = 1;

-- Persist (10.6.1+)
SET PERSIST sync_binlog                  = 1;
SET PERSIST innodb_flush_log_at_trx_commit = 1;
```

Equivalent `my.cnf` :

```ini
[mysqld]
sync_binlog                    = 1
innodb_flush_log_at_trx_commit = 1
innodb_doublewrite             = 1
log_bin                        = /var/log/mysql/mariadb-bin
binlog_format                  = MIXED
expire_logs_days               = 14
```

Trade-off : durability vs IOPS. Each commit causes ONE fsync on the redo log and ONE fsync on the binlog. On magnetic disks expect 100-200 commits/sec ceiling ; on NVMe it does not matter.

## Example 9 : Investigating "Default Changed After Upgrade"

```sql
-- 10.6+ : full default snapshot for a config drift audit
SELECT @@version, @@version_compile_os, NOW();

SELECT 'sql_mode'                       AS variable, @@sql_mode                       AS value UNION ALL
SELECT 'character_set_server'           , @@character_set_server                                  UNION ALL
SELECT 'collation_server'               , @@collation_server                                      UNION ALL
SELECT 'binlog_format'                  , @@binlog_format                                         UNION ALL
SELECT 'sync_binlog'                    , @@sync_binlog                                           UNION ALL
SELECT 'innodb_flush_log_at_trx_commit' , @@innodb_flush_log_at_trx_commit                        UNION ALL
SELECT 'default_storage_engine'         , @@default_storage_engine                                UNION ALL
SELECT 'lower_case_table_names'         , @@lower_case_table_names                                UNION ALL
SELECT 'default_authentication_plugin'  , @@default_authentication_plugin                         UNION ALL
SELECT 'innodb_default_row_format'      , @@innodb_default_row_format                             UNION ALL
SELECT 'transaction_isolation'          , @@transaction_isolation                                 UNION ALL
SELECT 'autocommit'                     , @@autocommit;

-- 10.6+ : trace WHERE each variable was set (file path, command-line, runtime)
SELECT variable_name, variable_source, variable_path, set_time
FROM performance_schema.variables_info
WHERE variable_name IN (
  'sql_mode','character_set_server','binlog_format','sync_binlog',
  'innodb_flush_log_at_trx_commit','default_storage_engine'
);
```

Run this on the OLD server before upgrading, save the output, then run again on the NEW server and diff. Defaults that differ are the candidates for the regression.

## Example 10 : Forcing a Strict Production my.cnf

```ini
# /etc/mysql/mariadb.conf.d/99-production.cnf
# Loaded last in alphabetical order, overrides distro defaults

[mysqld]
# Identity
character-set-server               = utf8mb4
collation-server                   = utf8mb4_uca1400_ai_ci

# Strictness
sql_mode                           = STRICT_TRANS_TABLES,STRICT_ALL_TABLES,NO_ENGINE_SUBSTITUTION,NO_ZERO_DATE,NO_ZERO_IN_DATE,ERROR_FOR_DIVISION_BY_ZERO
default_storage_engine             = InnoDB
innodb_default_row_format          = dynamic

# Durability (primary)
sync_binlog                        = 1
innodb_flush_log_at_trx_commit     = 1
innodb_doublewrite                 = 1
innodb_file_per_table              = 1

# Binlog
log_bin                            = /var/log/mysql/mariadb-bin
binlog_format                      = MIXED
expire_logs_days                   = 14
max_binlog_size                    = 256M

# Replication-friendly
gtid_strict_mode                   = 1
log_slave_updates                  = 1

# Auth
plugin_load_add                    = auth_ed25519

# Case-sensitivity (locked at initdb on Linux already, but explicit is good)
lower_case_table_names             = 0

[client]
default-character-set              = utf8mb4

[mariadb-dump]
default-character-set              = utf8mb4
```

After `sudo systemctl restart mariadb`, run Example 9 to verify the live values match the file.

## Example 11 : Creating ed25519 Users (Recommended Default)

```sql
-- 10.1.21+ : load plug-in once
INSTALL SONAME 'auth_ed25519';

-- 10.4+ : new user with ed25519 instead of mysql_native_password
CREATE USER 'app'@'%' IDENTIFIED VIA ed25519 USING PASSWORD('strong-secret');

-- Verify
SELECT user, host, plugin FROM mysql.global_priv;

-- Migrate an existing mysql_native_password user
ALTER USER 'app'@'%' IDENTIFIED VIA ed25519 USING PASSWORD('strong-secret');
```

Persist the plug-in load in `my.cnf` so it survives restart :

```ini
[mysqld]
plugin_load_add = auth_ed25519
```

## Example 12 : Verifying lower_case_table_names Cannot Be Changed

```sql
-- Linux default
SELECT @@lower_case_table_names;  -- 0

-- Attempt to change at runtime
SET GLOBAL lower_case_table_names = 1;
-- ERROR 1238 (HY000): Variable 'lower_case_table_names' is a read only variable

-- Even via my.cnf restart : the data dictionary was initialized with 0
-- Changing the option file value after initdb makes the SERVER REFUSE TO START
-- with : 'Different lower_case_table_names settings for server and data dictionary'
```

The fix : dump everything with `mariadb-dump`, reinitialize the data directory with the new value, restore the dump. There is no in-place toggle.
