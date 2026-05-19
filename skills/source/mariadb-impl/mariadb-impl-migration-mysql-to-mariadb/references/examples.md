# Examples : MySQL to MariaDB Migration

10+ end-to-end working examples. All shell-snippets assume root has sudo. All SQL snippets are annotated with the MariaDB version they target.

## Example 1 : MySQL 5.7 in-place upgrade to MariaDB 10.11-LTS (Debian 12)

```bash
# 1. Pre-migration backup
mariabackup --backup --target-dir=/backup/pre-migration --user=root --password=ROOT_SECRET
# (fall back : mysqldump --single-transaction --routines --triggers --events --all-databases > /backup/pre.sql)

# 2. Stop MySQL
systemctl stop mysql

# 3. Remove MySQL packages (datadir survives)
apt-get remove --purge mysql-server mysql-server-core-5.7
apt-get autoremove

# 4. Install MariaDB 10.11 from the official repo
curl -LsS https://r.mariadb.com/downloads/mariadb_repo_setup | bash -s -- --mariadb-server-version="mariadb-10.11"
apt-get update
apt-get install -y mariadb-server

# 5. Start MariaDB
systemctl start mariadb

# 6. Run mariadb-upgrade exactly ONCE
mariadb-upgrade -u root -p

# 7. Verify
mariadb -u root -p -e "SELECT VERSION(), @@version_comment;"
mariadb -u root -p -e "SELECT plugin_name, plugin_status FROM information_schema.plugins WHERE plugin_name LIKE 'auth%' OR plugin_name LIKE '%password%';"
```

## Example 2 : MySQL 8.0 dump-and-restore to MariaDB 11.4-LTS

```bash
# On MySQL 8.0 source
mysqldump \
  --single-transaction \
  --routines --triggers --events \
  --add-drop-database --databases appdb otherdb \
  --column-statistics=0 \
  > /tmp/mysql8-export.sql

# Transfer dump to MariaDB host (rsync, scp, etc.)
scp /tmp/mysql8-export.sql mariadb-host:/tmp/

# On MariaDB 11.4 host : install server first
curl -LsS https://r.mariadb.com/downloads/mariadb_repo_setup | bash -s -- --mariadb-server-version="mariadb-11.4"
apt-get update && apt-get install -y mariadb-server
systemctl start mariadb

# Restore
mariadb -u root -p < /tmp/mysql8-export.sql

# Finalise system tables
mariadb-upgrade -u root -p

# JSON sweep, user migration, INVISIBLE rename : see examples 4, 5, 6
```

## Example 3 : Pre-migration JSON column inventory (dump from MySQL source)

```sql
-- Run on MySQL 5.7+ / 8.x source BEFORE migration
-- 5.7+ / 8.x source SQL
SELECT
  CONCAT(TABLE_SCHEMA, '.', TABLE_NAME, '.', COLUMN_NAME) AS json_col
FROM information_schema.COLUMNS
WHERE DATA_TYPE = 'json'
  AND TABLE_SCHEMA NOT IN ('mysql', 'information_schema', 'performance_schema', 'sys');
```

Save the output ; this is the exact list to add CHECK constraints for on the MariaDB side.

## Example 4 : JSON CHECK constraint sweep (post-restore)

```sql
-- 10.6+ / 10.11+ / 11.x / 12.x
-- Given the column list from Example 3, add CHECK constraints
USE appdb;

ALTER TABLE orders
  ADD CONSTRAINT chk_orders_payload_json CHECK (JSON_VALID(payload));

ALTER TABLE users
  ADD CONSTRAINT chk_users_attributes_json CHECK (JSON_VALID(attributes));

ALTER TABLE audit_log
  ADD CONSTRAINT chk_audit_log_data_json CHECK (JSON_VALID(data));

-- Verify coverage
SELECT TABLE_NAME, COLUMN_NAME, CONSTRAINT_NAME, CHECK_CLAUSE
FROM information_schema.CHECK_CONSTRAINTS
WHERE CONSTRAINT_SCHEMA = 'appdb'
  AND CHECK_CLAUSE LIKE '%json_valid%';
```

## Example 5 : User migration to ed25519

```sql
-- 10.6+ / 10.11+ / 11.x / 12.x
-- One-off : install plug-in
INSTALL SONAME 'auth_ed25519';

-- Persist availability across restarts (add to my.cnf [mariadb] section)
-- plugin_load_add = auth_ed25519

-- Re-create application user with ed25519
DROP USER IF EXISTS 'app'@'%';
CREATE USER 'app'@'%'
  IDENTIFIED VIA ed25519 USING PASSWORD('NEW_STRONG_SECRET');
GRANT SELECT, INSERT, UPDATE, DELETE ON appdb.* TO 'app'@'%';

-- For a legacy driver that does not speak ed25519, use mysql_native_password
DROP USER IF EXISTS 'legacy'@'10.0.%';
CREATE USER 'legacy'@'10.0.%'
  IDENTIFIED WITH mysql_native_password BY 'TRANSITIONAL_SECRET';
GRANT SELECT ON appdb.* TO 'legacy'@'10.0.%';

-- Verify
SELECT user, host, plugin FROM mysql.global_priv WHERE user IN ('app', 'legacy');
```

## Example 6 : INVISIBLE INDEX to IGNORED INDEX rewrite

```sql
-- WRONG (MySQL 8 syntax ; fails in MariaDB with a syntax error)
-- ALTER TABLE orders ALTER INDEX idx_orders_buyer INVISIBLE;

-- RIGHT (MariaDB 10.6+ / 10.11+ / 11.x / 12.x)
ALTER TABLE orders ALTER INDEX idx_orders_buyer IGNORED;

-- Re-enable
ALTER TABLE orders ALTER INDEX idx_orders_buyer NOT IGNORED;

-- Confirm via SHOW INDEX
SHOW INDEX FROM orders WHERE Key_name = 'idx_orders_buyer'\G
-- Look for "Ignored : YES" or "Ignored : NO" in the output
```

## Example 7 : Cut-over with positional replication (MariaDB-replica-of-MySQL)

```sql
-- On MySQL 8 source : create replication user with mysql_native_password (MariaDB cannot speak caching_sha2_password)
-- 8.x source SQL
CREATE USER 'repl'@'%' IDENTIFIED WITH mysql_native_password BY 'REPL_SECRET';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';

-- Capture position
SHOW MASTER STATUS\G
-- note File and Position

-- On MariaDB 10.11+ replica
-- 10.11+ / 11.x / 12.x
CHANGE MASTER TO
  MASTER_HOST = 'mysql-source.internal',
  MASTER_USER = 'repl',
  MASTER_PASSWORD = 'REPL_SECRET',
  MASTER_LOG_FILE = 'mysql-bin.000123',
  MASTER_LOG_POS = 4567,
  MASTER_USE_GTID = NO;    -- positional only ; GTID is INCOMPATIBLE
START SLAVE;

SHOW SLAVE STATUS\G
-- Wait until Seconds_Behind_Master = 0

-- Cut-over moment :
-- 1. Stop apps (drain writes on MySQL)
-- 2. Wait Seconds_Behind_Master = 0 on MariaDB
-- 3. STOP SLAVE; on MariaDB
-- 4. Repoint apps at MariaDB
-- 5. mariadb-upgrade -u root -p
-- 6. Sweep JSON CHECKs, re-create users with ed25519
```

## Example 8 : One-shot export back to MySQL (rollback or testing)

MariaDB-to-MySQL replication is NOT supported. A one-shot export is the only path :

```bash
# On MariaDB
mariadb-dump \
  --single-transaction \
  --routines --triggers --events \
  --skip-comments \
  --compatible=mysql8023 \
  --databases appdb > /tmp/mariadb-export.sql

# Manual remediation before importing to MySQL :
# - Replace IGNORED INDEX with INVISIBLE INDEX
# - Replace VIA ed25519 USING PASSWORD with WITH caching_sha2_password BY
# - Replace SET DEFAULT ROLE ... FOR ... with ALTER USER ... DEFAULT ROLE ...
# - Replace CREATE SEQUENCE ... with rewritten AUTO_INCREMENT tables

# On MySQL target
mysql -u root -p < /tmp/mariadb-export-edited.sql
```

## Example 9 : Sequence introduction post-migration (deliberate, MariaDB-only)

After the migration is stable, you may CHOOSE to replace certain AUTO_INCREMENT columns with sequences for cross-table id-allocation. NEVER do this during migration ; only after.

```sql
-- 10.6+ / 10.11+ / 11.x / 12.x
CREATE SEQUENCE order_seq
  START WITH 1000000
  INCREMENT BY 1
  MINVALUE 1000000
  MAXVALUE 9999999999
  CACHE 100
  NOCYCLE;

ALTER TABLE orders MODIFY id BIGINT NOT NULL;
ALTER TABLE orders DROP PRIMARY KEY, ADD PRIMARY KEY (id);
ALTER TABLE orders MODIFY id BIGINT NOT NULL DEFAULT (NEXT VALUE FOR order_seq);

-- Allocate next id explicitly
INSERT INTO orders (id, payload)
  VALUES (NEXT VALUE FOR order_seq, '{"k":"v"}');
```

## Example 10 : my.cnf migration diff

```ini
# /etc/mysql/my.cnf  on MariaDB 11.4 (post-migration from MySQL 8.0)
[mariadb]
# --- KEEP from MySQL ---
character_set_server = utf8mb4
collation_server = utf8mb4_uca1400_ai_ci   # 10.10+ ; on older keep utf8mb4_unicode_ci
innodb_buffer_pool_size = 16G              # tune ; on 10.11.12+ chunk_size deprecated
innodb_default_row_format = dynamic
max_allowed_packet = 256M
binlog_format = MIXED                      # MariaDB default since 10.2.4
log_bin = mariadb-bin
server_id = 11

# --- REMOVE (MySQL-only) ---
# default_authentication_plugin = caching_sha2_password
# query_cache_type = ON                    # REMOVED in MariaDB 11.0+
# query_cache_size = 0                     # REMOVED in MariaDB 11.0+
# gtid_mode = ON
# enforce_gtid_consistency = ON
# caching_sha2_password_auto_generate_rsa_keys = OFF

# --- ADD (MariaDB) ---
plugin_load_add = auth_ed25519             # ed25519 available without INSTALL SONAME
gtid_strict_mode = ON                      # MariaDB-style GTID safety

# --- REVIEW (defaults differ) ---
sql_mode = STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION
# do NOT carry ONLY_FULL_GROUP_BY across blindly ; verify app compliance first
```

## Example 11 : Post-migration validation suite

```sql
-- 10.6+ / 10.11+ / 11.x / 12.x
-- Run as root after mariadb-upgrade and after user re-creation.

-- A. Server identity
SELECT VERSION() AS server_version, @@version_comment AS edition;

-- B. Privilege store (10.4+)
SELECT user, host, plugin
FROM mysql.global_priv
WHERE user NOT IN ('mariadb.sys', 'mysql', '');

-- C. No caching_sha2_password leftovers
SELECT user, host, plugin
FROM mysql.global_priv
WHERE plugin = 'caching_sha2_password';
-- expect 0 rows

-- D. JSON columns are CHECK-protected
SELECT TABLE_NAME, COLUMN_NAME, IFNULL(cc.CONSTRAINT_NAME, '*** MISSING JSON_VALID CHECK ***') AS status
FROM information_schema.COLUMNS c
LEFT JOIN information_schema.CHECK_CONSTRAINTS cc
  ON cc.CONSTRAINT_SCHEMA = c.TABLE_SCHEMA
 AND cc.CHECK_CLAUSE LIKE CONCAT('%', c.COLUMN_NAME, '%')
 AND cc.CHECK_CLAUSE LIKE '%json_valid%'
WHERE c.TABLE_SCHEMA = DATABASE()
  AND c.COLUMN_COMMENT LIKE '%was JSON in MySQL%'   -- or other heuristic
ORDER BY c.TABLE_NAME, c.COLUMN_NAME;

-- E. No INVISIBLE leftovers anywhere in DDL (search dumped DDL)
-- (run outside mariadb : grep -i 'INVISIBLE' dump.sql ; expect 0 hits in DDL)

-- F. Replication state (if replica)
SHOW SLAVE STATUS\G
-- Slave_IO_Running and Slave_SQL_Running both Yes
```

## Example 12 : Roll-back plan

Before cut-over, document this plan :

1. Keep MySQL host running, read-only, for N days (`SET GLOBAL super_read_only = ON;` on MySQL).
2. Take a fresh MariaDB-side mariabackup AFTER cut-over (post-cut-over delta is captured).
3. If rollback needed within window : stop apps, repoint to MySQL, accept data loss for the window (since MySQL cannot replicate from MariaDB).
4. NEVER attempt to "re-sync MariaDB writes back to MySQL" : the schemes (JSON, GTID, IGNORED-INDEX DDL, ed25519 users) require manual remediation in reverse.
