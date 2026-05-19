# MariaDB Storage Engines : Anti-Patterns

Real-world engine misuse patterns, why they fail, and the deterministic correct alternative. Each entry includes the broken code, the symptom, the root cause, and the fix.

---

## Anti-Pattern 1 : MyISAM for User Data

### Broken

```sql
CREATE TABLE orders (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  customer_id BIGINT UNSIGNED NOT NULL,
  total_cents BIGINT NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
) ENGINE=MyISAM;
```

### Symptoms

- Every concurrent INSERT serialises on a table-level write lock ; throughput collapses with > 1 writer.
- A power loss corrupts index files ; `mysqld` refuses to open the table on restart and demands `REPAIR TABLE`.
- Foreign keys to `customer` are accepted by the parser but enforced nowhere ; orphan rows accumulate.
- BEGIN / COMMIT silently succeed but never roll anything back ; partial writes are permanent.

### Root cause

MyISAM has no row-level locking, no crash recovery via redo log, no foreign keys, and no transactions. It survives in MariaDB only for compatibility with pre-2010 schemas.

### Fix

```sql
ALTER TABLE orders ENGINE=InnoDB,
  ROW_FORMAT=DYNAMIC,
  ALGORITHM=COPY, LOCK=SHARED;
```

ALWAYS run this audit on every legacy schema and convert all MyISAM tables to InnoDB during a maintenance window.

---

## Anti-Pattern 2 : ColumnStore for OLTP

### Broken

```sql
-- ColumnStore distribution
CREATE TABLE shopping_cart (
  id BIGINT UNSIGNED NOT NULL,
  user_id BIGINT UNSIGNED NOT NULL,
  sku VARCHAR(64) NOT NULL,
  qty INT NOT NULL
) ENGINE=ColumnStore;

-- Application code
UPDATE shopping_cart SET qty = qty + 1 WHERE id = ? AND sku = ?;
```

### Symptoms

- Every UPDATE rewrites a full extent (typically 1M-row block) ; latency climbs from milliseconds to seconds.
- `WHERE id = ?` lookups scan every extent ; ColumnStore has no row-level index.
- Disk usage explodes because of write amplification ; columnar storage is optimised for read-mostly bulk loads.
- Application connection pool exhausts as queries queue behind extent rewrites.

### Root cause

ColumnStore is a columnar OLAP engine. Storage is laid out per column per extent, optimised for sequential scans over many rows. Point updates and lookups require reading and rewriting the full extent containing the row.

### Fix

```sql
-- OLTP path : keep cart on InnoDB
CREATE TABLE shopping_cart (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  user_id BIGINT UNSIGNED NOT NULL,
  sku VARCHAR(64) NOT NULL,
  qty INT NOT NULL,
  KEY ix_cart_user (user_id)
) ENGINE=InnoDB ROW_FORMAT=DYNAMIC;

-- ETL into ColumnStore daily for analytics
-- cpimport -m 1 analytics shopping_cart_history /tmp/cart_2026_05.csv
```

ALWAYS reserve ColumnStore for analytical workloads loaded via `cpimport`. NEVER let an OLTP application talk to ColumnStore directly.

---

## Anti-Pattern 3 : MEMORY for "Persistent" Cache

### Broken

```sql
CREATE TABLE pricing_cache (
  sku VARCHAR(64) NOT NULL PRIMARY KEY,
  computed_price_cents BIGINT NOT NULL,
  updated_at INT UNSIGNED NOT NULL
) ENGINE=MEMORY;

-- Application reads from pricing_cache assuming it survives restarts
```

### Symptoms

- After every MariaDB restart the table is empty ; queries return zero rows.
- Application falls back to recompute every price ; database CPU spikes for hours.
- Replication breaks because the row-based binlog references rows that no longer exist on the replica after replica restart.

### Root cause

The MEMORY engine stores its schema persistently in a definition file, but rows live only in RAM. A clean shutdown, a crash, or a restart wipes every row. There is no on-disk row state.

### Fix

```sql
-- Option A : durable cache on InnoDB with TTL column
CREATE TABLE pricing_cache (
  sku VARCHAR(64) NOT NULL PRIMARY KEY,
  computed_price_cents BIGINT NOT NULL,
  expires_at INT UNSIGNED NOT NULL,
  KEY ix_expires (expires_at)
) ENGINE=InnoDB ROW_FORMAT=DYNAMIC;

-- Option B : keep MEMORY but pre-load on startup from InnoDB source of truth
CREATE TABLE pricing_cache (
  sku VARCHAR(64) NOT NULL PRIMARY KEY USING HASH,
  computed_price_cents BIGINT NOT NULL,
  expires_at INT UNSIGNED NOT NULL
) ENGINE=MEMORY MAX_ROWS=100000;

-- On application boot
INSERT INTO pricing_cache
SELECT sku, computed_price_cents, expires_at
FROM pricing_cache_durable WHERE expires_at > UNIX_TIMESTAMP();
```

ALWAYS treat MEMORY as a read-through cache layered above a durable source. NEVER rely on MEMORY across a restart boundary.

---

## Anti-Pattern 4 : Mixing Engines Inside a Transaction

### Broken

```sql
-- audit_log is MyISAM ; orders is InnoDB
BEGIN;
  INSERT INTO orders (customer_id, total_cents) VALUES (42, 9900);
  INSERT INTO audit_log (event) VALUES ('order created');
  -- ... business logic detects a problem ...
ROLLBACK;
```

### Symptoms

- The `orders` row is correctly rolled back ; InnoDB participates in the transaction.
- The `audit_log` row is NOT rolled back ; the MyISAM INSERT committed instantly outside of any transaction.
- The audit trail and the actual data drift apart.
- A warning is emitted (`Some non-transactional changed tables couldn't be rolled back`) and silently ignored by most clients.

### Root cause

Only transactional engines (InnoDB and partially ColumnStore) honour `BEGIN` and `ROLLBACK`. MyISAM, Aria, MEMORY, CONNECT, and CSV writes are committed immediately when the statement executes.

### Fix

```sql
-- Move audit_log to InnoDB so it participates in transactions
ALTER TABLE audit_log ENGINE=InnoDB, ROW_FORMAT=DYNAMIC,
  ALGORITHM=COPY, LOCK=SHARED;
```

ALWAYS keep all tables that need transactional all-or-nothing semantics on the same transactional engine (InnoDB). NEVER mix non-transactional engines into a transaction expected to roll back.

---

## Anti-Pattern 5 : Spider Over Slow WAN Without Pooling

### Broken

```sql
-- Spider router pointing to a backend over a WAN with default timeouts
CREATE OR REPLACE SERVER us_backend
  FOREIGN DATA WRAPPER mysql
  OPTIONS (HOST 'us-east-1.example.com', DATABASE 'shop',
           USER 'spider', PASSWORD 'redacted', PORT 3306);
-- Default spider_connect_timeout = 30, spider_net_read_timeout = 600
```

### Symptoms

- Every Spider query opens a fresh TCP connection across the WAN ; latency adds 100-200 ms per query.
- A backend brown-out hangs Spider queries for 600 seconds (10-minute net-read default).
- The Spider router exhausts its file descriptors during a backend outage.
- `SHOW PROCESSLIST` on the router shows hundreds of `Spider: connecting` threads.

### Root cause

Spider's default network timeouts are conservative and assume LAN deployments. Without connection pooling and aggressive timeout tuning, every client request can wait minutes for a dead backend. Spider has no built-in connection pool ; each session holds its own backend connection.

### Fix

```sql
-- Tune timeouts for WAN
SET GLOBAL spider_connect_timeout = 5;
SET GLOBAL spider_net_read_timeout = 30;
SET GLOBAL spider_net_write_timeout = 30;
SET GLOBAL spider_quick_mode = 3;

-- Use a single Spider user with connection multiplexing on the backend
-- (configure max_user_connections on each backend MariaDB)
```

ALWAYS tune Spider timeouts before deploying across a WAN. ALWAYS pair Spider with Galera or replication for HA, since Spider's own HA was removed in 10.7.5 (MDEV-28479).

---

## Anti-Pattern 6 : Aria for High-Concurrency Writes

### Broken

```sql
-- Web application click tracking, 1000 writers/sec
CREATE TABLE click_event (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  user_id BIGINT UNSIGNED NOT NULL,
  url VARCHAR(512) NOT NULL,
  clicked_at TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
) ENGINE=Aria ROW_FORMAT=PAGE;
```

### Symptoms

- Throughput caps around 50-100 INSERTs/sec ; the table-level lock serialises every writer.
- `SHOW PROCESSLIST` shows many sessions in `Waiting for table level lock`.
- Read queries against `click_event` block until current writers finish.
- CPU is largely idle ; the bottleneck is the lock, not the disk or the engine.

### Root cause

Aria uses table-level locking. Crash-safety on PAGE row format only protects against power loss ; it does not enable row-level concurrency. Aria is intended for system tables and append-only workloads with a single writer, not for high-concurrency user workloads.

### Fix

```sql
-- Convert to InnoDB for row-level locking and MVCC
ALTER TABLE click_event ENGINE=InnoDB, ROW_FORMAT=DYNAMIC,
  ALGORITHM=COPY, LOCK=SHARED;
```

ALWAYS use InnoDB for any table with concurrent writers. NEVER use Aria for user-facing high-throughput workloads.

---

## Anti-Pattern 7 : InnoDB With innodb_file_per_table=OFF

### Broken

```ini
# Legacy config from a MySQL 5.5 era migration
[mysqld]
innodb_file_per_table = OFF
```

### Symptoms

- Every InnoDB table writes into the shared `ibdata1` tablespace.
- `ibdata1` grows monotonically ; dropping a table reclaims no disk space.
- A 1 TB shared tablespace cannot be shrunk without a full dump-and-reload.
- Online backup tools cannot do per-table partial backups effectively.

### Root cause

The shared tablespace was the InnoDB default in MySQL 5.5 and earlier. It bundles every table's pages into one file, making per-table operations (truncate, drop, partial backup, file-system migration) impossible. MariaDB 10.x and modern MySQL ship with `innodb_file_per_table=ON` by default.

### Fix

```ini
# /etc/mysql/mariadb.conf.d/50-server.cnf
[mysqld]
innodb_file_per_table = ON
```

```sql
-- Force every existing table to its own .ibd file
ALTER TABLE my_table ENGINE=InnoDB, ALGORITHM=COPY, LOCK=SHARED;
-- Repeat for every table ; then a dump-and-reload to shrink ibdata1
```

ALWAYS verify `SHOW VARIABLES LIKE 'innodb_file_per_table'` returns `ON` on every new server. NEVER inherit a shared-tablespace config from a legacy migration without a full dump-and-reload.

---

## Anti-Pattern 8 : ENGINE Without ROW_FORMAT on Large-Column Tables

### Broken

```sql
-- 10.6+ : default config still uses latin1 on stock builds and may pick non-DYNAMIC row format
CREATE TABLE document (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  title VARCHAR(1000) NOT NULL,
  body LONGTEXT NOT NULL,
  tags VARCHAR(2000) NOT NULL,
  KEY ix_title (title)
) ENGINE=InnoDB;
```

### Symptoms

- On a server where `innodb_default_row_format` was set to `COMPACT` (legacy), `CREATE INDEX` on `title VARCHAR(1000)` fails with `Specified key was too long ; max key length is 767 bytes`.
- COMPACT row format stores variable-length columns inline up to 767 bytes ; longer columns truncate or fail.
- Migration scripts succeed on developer machines (DYNAMIC) and fail in production (COMPACT).

### Root cause

InnoDB's COMPACT row format limits indexed prefix length to 767 bytes per column. DYNAMIC row format stores long variable-length columns off-page and supports the 3072-byte limit (or larger with `innodb_large_prefix` historically, now permanent). Different servers may have different `innodb_default_row_format` settings.

### Fix

```sql
-- 10.6+ : explicit ROW_FORMAT=DYNAMIC, always
CREATE TABLE document (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  title VARCHAR(1000) NOT NULL,
  body LONGTEXT NOT NULL,
  tags VARCHAR(2000) NOT NULL,
  KEY ix_title (title)
) ENGINE=InnoDB ROW_FORMAT=DYNAMIC
  DEFAULT CHARSET=utf8mb4
  COLLATE=utf8mb4_uca1400_ai_ci;
```

ALWAYS specify `ROW_FORMAT=DYNAMIC` on every InnoDB table that contains a column longer than VARCHAR(255). ALWAYS audit `innodb_default_row_format` on every server.

---

## Anti-Pattern 9 : CONNECT With User-Controlled FILE_NAME

### Broken

```sql
-- Application takes user input and creates a CONNECT table over the chosen file
PREPARE stmt FROM
  'CREATE TABLE feed_import ENGINE=CONNECT TABLE_TYPE=CSV FILE_NAME=?';
EXECUTE stmt USING @user_supplied_path;
```

### Symptoms

- An attacker supplies `/etc/passwd` and SELECTs the contents through the federated table.
- An attacker supplies `/var/lib/mysql/mysql/user.MYD` and reads the MariaDB credential store.
- The CONNECT engine has the file-system privileges of the `mysqld` user, which usually means read access to many sensitive files.

### Root cause

CONNECT's `FILE_NAME`, `CONNECTION`, and `TABNAME` parameters are file-system and credential primitives. They do not sandbox the path. A user-controlled FILE_NAME is equivalent to a file-read vulnerability in the database engine.

### Fix

```sql
-- ALWAYS hard-code paths under a controlled directory
CREATE TABLE feed_import (
  sku VARCHAR(64), name VARCHAR(255)
) ENGINE=CONNECT
  TABLE_TYPE=CSV
  FILE_NAME='/var/feeds/incoming/products.csv'
  SEP_CHAR=','
  HEADER=1;
```

ALWAYS hard-code CONNECT file paths in DDL. NEVER let user input flow into a CONNECT table definition. ALWAYS run `mysqld` under a dedicated user with minimal file-system permissions.

---

## See Also

- Engine feature matrix : `methods.md`
- Working examples : `examples.md`
- Architecture overview : `mariadb-core-architecture` skill
- DDL algorithms : `mariadb-syntax-ddl-alter` skill (when created)
