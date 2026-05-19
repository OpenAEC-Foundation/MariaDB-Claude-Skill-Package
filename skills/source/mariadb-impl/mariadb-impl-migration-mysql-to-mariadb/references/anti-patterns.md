# Anti-Patterns : MySQL to MariaDB Migration

Eight documented anti-patterns, each with the symptom, the root cause, and the correct alternative. All applicable to MariaDB 10.6-LTS, 10.11-LTS, 11.x, 12.x unless stated.

## AP-1 : Skip mariadb-upgrade after binary swap

### Symptom
- Apps connect, basic SELECTs work, but `SHOW GRANTS FOR 'user'@'%'` returns stale rows.
- New users created on 10.4+ are invisible to legacy tooling that reads `mysql.user`.
- Replication breaks at the first DDL because the system tables still have MySQL field layout.

### Wrong
```bash
systemctl stop mysql
apt-get install mariadb-server
systemctl start mariadb
# Apps connect, "looks fine", skip mariadb-upgrade entirely
```

### Why it fails
MariaDB 10.4+ stores privileges in `mysql.global_priv`, not `mysql.user`. The binary swap puts the new server binary in place but does NOT rewrite the system tables. Without `mariadb-upgrade`, the privilege store is in a transitional state : the new server reads `mysql.user` as a compatibility view, but writes to `mysql.global_priv`. Every privilege change widens the drift until the next restart fails or silently mis-applies grants.

### Right
```bash
systemctl stop mysql
apt-get install mariadb-server
systemctl start mariadb
mariadb-upgrade -u root -p     # MANDATORY, exactly once
```

## AP-2 : Expect MySQL GTID continuity into MariaDB (L-004)

### Symptom
- `START SLAVE` with `MASTER_AUTO_POSITION=1` or `MASTER_USE_GTID=current_pos` errors out or stops applying after a few transactions.
- `SHOW SLAVE STATUS\G` shows `Last_SQL_Error : Could not find GTID state requested by slave`.

### Wrong
```sql
-- On MariaDB replica trying to follow a MySQL primary
-- 10.6+ / 10.11+ / 11.x / 12.x
CHANGE MASTER TO
  MASTER_HOST = 'mysql-source',
  MASTER_USER = 'repl',
  MASTER_PASSWORD = 'x',
  MASTER_USE_GTID = current_pos;   -- WRONG
START SLAVE;
```

### Why it fails
MariaDB GTID is `domain-server-sequence` (e.g. `0-1-1234`). MySQL GTID is `uuid:seqno` (e.g. `3e11fa47-...:23`). The schemes are not interconvertible. MariaDB cannot translate an incoming MySQL GTID into its own scheme, so the applier has no way to track position.

### Right
```sql
-- 10.6+ / 10.11+ / 11.x / 12.x
CHANGE MASTER TO
  MASTER_HOST = 'mysql-source',
  MASTER_USER = 'repl',
  MASTER_PASSWORD = 'x',
  MASTER_LOG_FILE = 'mysql-bin.000123',
  MASTER_LOG_POS = 4567,
  MASTER_USE_GTID = NO;            -- positional only ; cut-over bridge only
START SLAVE;
```

Continuous GTID-tracked replication between MySQL and MariaDB is IMPOSSIBLE. The MariaDB-replica-of-MySQL path is binlog-positional ONLY, intended as a cut-over bridge. After cut-over, do not maintain a reverse MariaDB-to-MySQL replica : MySQL cannot replicate from MariaDB at all.

## AP-3 : Assume JSON storage is identical (L-005, D-010)

### Symptom
- JSON columns survive the migration.
- A few weeks later, an UPDATE writes non-JSON text into a JSON column without error.
- Application code that does `JSON_EXTRACT` fails at read time with `Invalid JSON text` somewhere downstream.

### Wrong
```sql
-- 10.6+ / 10.11+ / 11.x / 12.x
-- Migrate the schema, restore data, declare done
CREATE TABLE orders (
  id BIGINT PRIMARY KEY,
  payload JSON     -- LONGTEXT alias ; NO insert-time validation
);

-- Later, somewhere in app code or a bad migration script
UPDATE orders SET payload = 'not json' WHERE id = 1;   -- accepted silently
```

### Why it fails
MariaDB `JSON` is a LONGTEXT alias. MySQL 5.7.8+ stores JSON in a binary format that validates on every write. MariaDB does NOT validate unless you add a CHECK constraint. The migration carries the column type over but loses the validation invariant. Silent corruption follows.

### Right
```sql
-- 10.6+ / 10.11+ / 11.x / 12.x
ALTER TABLE orders
  ADD CONSTRAINT chk_orders_payload_json CHECK (JSON_VALID(payload));

-- Now the bad UPDATE fails immediately :
UPDATE orders SET payload = 'not json' WHERE id = 1;
-- ERROR 4025 (23000): CONSTRAINT `chk_orders_payload_json` failed for `orders`
```

ALWAYS sweep every JSON column after migration. The pre-migration inventory (Example 3 in `references/examples.md`) is the authoritative target list.

## AP-4 : Copy MySQL 8 INVISIBLE INDEX syntax (L-006)

### Symptom
- `ALTER TABLE` statements that worked on MySQL 8 fail in MariaDB with a syntax error.
- ORM-generated migration scripts that emit `INVISIBLE` break at `db migrate`.

### Wrong
```sql
-- WRONG (MySQL 8 syntax in MariaDB)
-- 10.6+ / 10.11+ / 11.x / 12.x
ALTER TABLE orders ALTER INDEX idx_orders_buyer INVISIBLE;
-- ERROR 1064 (42000): You have an error in your SQL syntax
```

### Why it fails
MariaDB chose the keyword `IGNORED` for the same concept (an index the optimiser is told to ignore for testing). MySQL 8 chose `INVISIBLE`. The features are semantically equivalent but the keywords differ.

### Right
```sql
-- 10.6+ / 10.11+ / 11.x / 12.x
ALTER TABLE orders ALTER INDEX idx_orders_buyer IGNORED;
ALTER TABLE orders ALTER INDEX idx_orders_buyer NOT IGNORED;
```

Grep all dumped DDL and migration scripts for the keyword `INVISIBLE` BEFORE running them on MariaDB. Translate every hit.

## AP-5 : Keep caching_sha2_password users

### Symptom
- After dump-restore, app connects with `ERROR 2059 (HY000): Authentication plugin 'caching_sha2_password' cannot be loaded`.
- Or : the GRANT statement restores but the user can never log in.

### Wrong
```sql
-- Dump from MySQL 8 contains
CREATE USER 'app'@'%' IDENTIFIED WITH 'caching_sha2_password' BY 'secret';
-- Restored verbatim on MariaDB : user is created with an unsupported plug-in reference
```

### Why it fails
MariaDB does NOT ship the `caching_sha2_password` plug-in. The CREATE USER may succeed (or warn) but the user is unusable : authentication requests cannot be processed because the plug-in is not loaded.

### Right
```sql
-- 10.6+ / 10.11+ / 11.x / 12.x
INSTALL SONAME 'auth_ed25519';      -- one-off
DROP USER IF EXISTS 'app'@'%';
CREATE USER 'app'@'%'
  IDENTIFIED VIA ed25519 USING PASSWORD('NEW_STRONG_SECRET');
GRANT SELECT, INSERT, UPDATE, DELETE ON appdb.* TO 'app'@'%';
```

Driver-compatibility fallback : `mysql_native_password` works with every MySQL driver and is supported on every MariaDB version this skill targets.

## AP-6 : Copy MySQL my.cnf verbatim

### Symptom
- MariaDB fails to start with `unknown variable 'default_authentication_plugin'` or `unknown variable 'query_cache_type'` (on 11.0+).
- Or : starts but with unexpected sql_mode, missing GTID, or wrong plug-in loaded.

### Wrong
```ini
# Verbatim MySQL 8.0 my.cnf moved to MariaDB host
[mysqld]
default_authentication_plugin = caching_sha2_password   # NOT a MariaDB variable
query_cache_type = ON                                   # REMOVED in MariaDB 11.0+
query_cache_size = 64M                                  # REMOVED in MariaDB 11.0+
gtid_mode = ON                                          # NOT a MariaDB variable
enforce_gtid_consistency = ON                           # NOT a MariaDB variable
```

### Why it fails
MariaDB and MySQL share a large variable namespace but each side has variables and defaults the other does not recognise. Loading an unknown variable causes startup failure or silent ignore depending on version. The query cache was removed in MariaDB 11.0+ : referring to its variables in 11.0+ is a startup error.

### Right
```ini
# 10.11+ / 11.x / 12.x : start from a MariaDB-native baseline, port one setting at a time
[mariadb]
character_set_server = utf8mb4
collation_server = utf8mb4_uca1400_ai_ci      # 10.10+ ; older use utf8mb4_unicode_ci
innodb_buffer_pool_size = 16G
innodb_default_row_format = dynamic
binlog_format = MIXED
log_bin = mariadb-bin
server_id = 11
plugin_load_add = auth_ed25519
gtid_strict_mode = ON
sql_mode = STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION
```

Use the my.cnf migration checklist in `references/methods.md` §5 as the verbatim port-from-MySQL diff.

## AP-7 : Skip CHECK (JSON_VALID(col)) sweep on the basis that "it migrated cleanly"

### Symptom
- Migration completes with zero errors.
- Six months later, a bug-fix release writes a malformed string into a JSON column.
- Reporting jobs that JSON_EXTRACT silently return NULL on bad rows ; dashboards show plausible-looking but incomplete data.

### Wrong
```sql
-- 10.6+ / 10.11+ / 11.x / 12.x
-- "All rows passed JSON_VALID at restore time, so we are done."
SELECT COUNT(*) FROM orders WHERE NOT JSON_VALID(payload);
-- returns 0 ; declared safe
-- NO CHECK constraint added
```

### Why it fails
A point-in-time validation does NOT establish an ongoing invariant. Without a CHECK constraint, the very next non-JSON UPDATE will corrupt a row. The MySQL JSON type enforces the invariant on every write ; MariaDB only enforces it if you ASK with a CHECK.

### Right
```sql
-- 10.6+ / 10.11+ / 11.x / 12.x
-- Establish the ongoing invariant as part of the migration, not as an optional follow-up
ALTER TABLE orders
  ADD CONSTRAINT chk_orders_payload_json CHECK (JSON_VALID(payload));
```

The CHECK pays for itself on the first malformed UPDATE that would otherwise have corrupted the row.

## AP-8 : Run mariadb-upgrade without a backup

### Symptom
- `mariadb-upgrade` hits a CHECK TABLE failure on a corrupt InnoDB table and runs ALTER TABLE.
- The ALTER fails mid-way due to disk full or unrelated error.
- The system table state is now partially migrated and the user is faced with a broken catalogue and no rollback path.

### Wrong
```bash
systemctl stop mysql
apt-get install mariadb-server
systemctl start mariadb
mariadb-upgrade -u root -p     # No backup taken
```

### Why it fails
`mariadb-upgrade` performs DDL (ALTER TABLE, system table rewrite). DDL is not transactional across multiple tables. A failure mid-stream leaves the catalogue in an unknown state with no rollback. Recovery without a backup is at best a long manual repair, at worst data loss.

### Right
```bash
# Take a full physical or logical backup first
mariabackup --backup --target-dir=/backup/pre-upgrade --user=root --password=ROOT_SECRET
# OR
mariadb-dump --single-transaction --routines --triggers --events --all-databases > /backup/pre-upgrade.sql

# Stop, swap binaries, start
systemctl stop mysql
apt-get install mariadb-server
systemctl start mariadb

# Now run mariadb-upgrade with a known-good restore point
mariadb-upgrade -u root -p
```

If the upgrade fails, restore the backup, address the root cause (often a corrupt table from the MySQL side that needed repair BEFORE the swap), and retry.
