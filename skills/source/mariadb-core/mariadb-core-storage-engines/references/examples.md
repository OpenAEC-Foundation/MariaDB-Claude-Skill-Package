# MariaDB Storage Engines : Working Examples

Practical, copy-paste-ready CREATE TABLE patterns and engine-swap procedures, version-annotated for MariaDB 10.6+.

---

## Example 1 : InnoDB Transactional Table (the default 99% case)

```sql
-- 10.6+ : production-ready InnoDB transactional table
CREATE TABLE customer (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  email VARCHAR(320) NOT NULL,
  display_name VARCHAR(255) NOT NULL,
  status ENUM('active','suspended','deleted') NOT NULL DEFAULT 'active',
  created_at TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  updated_at TIMESTAMP(6) NOT NULL
    DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),
  UNIQUE KEY uq_customer_email (email),
  KEY ix_customer_status (status, created_at)
) ENGINE=InnoDB
  ROW_FORMAT=DYNAMIC
  DEFAULT CHARSET=utf8mb4
  COLLATE=utf8mb4_uca1400_ai_ci;
```

ALWAYS specify `ENGINE=InnoDB` even though it is the default since 10.2 ; visible-by-default makes code review reliable.

---

## Example 2 : InnoDB With Foreign Key

```sql
-- 10.6+ : InnoDB child table with FK to InnoDB parent
CREATE TABLE invoice (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  customer_id BIGINT UNSIGNED NOT NULL,
  total_cents BIGINT NOT NULL,
  paid_at TIMESTAMP(6) NULL,
  created_at TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  CONSTRAINT fk_invoice_customer
    FOREIGN KEY (customer_id) REFERENCES customer (id)
    ON DELETE RESTRICT ON UPDATE CASCADE,
  KEY ix_invoice_customer (customer_id, created_at)
) ENGINE=InnoDB
  ROW_FORMAT=DYNAMIC
  DEFAULT CHARSET=utf8mb4
  COLLATE=utf8mb4_uca1400_ai_ci;
```

ALWAYS use `ON DELETE RESTRICT` unless cascading deletes are a deliberate business rule ; `CASCADE` on a parent customer can wipe years of invoice history with one DELETE statement.

---

## Example 3 : InnoDB With Generated Column and Functional Index on JSON

```sql
-- 10.6+ : JSON column + virtual generated column + functional index
-- Reminder : MariaDB JSON is a LONGTEXT alias, not native binary
CREATE TABLE event (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  payload JSON NOT NULL CHECK (JSON_VALID(payload)),
  event_type VARCHAR(64) AS
    (JSON_VALUE(payload, '$.type')) VIRTUAL,
  KEY ix_event_type (event_type)
) ENGINE=InnoDB
  ROW_FORMAT=DYNAMIC
  DEFAULT CHARSET=utf8mb4
  COLLATE=utf8mb4_uca1400_ai_ci;
```

ALWAYS pair JSON columns with `CHECK (JSON_VALID(col))` ; without it MariaDB silently accepts non-JSON strings into the LONGTEXT.

---

## Example 4 : Aria System-Like Archive Table

```sql
-- 10.6+ : Aria for append-only audit archive, crash-safe but not transactional
CREATE TABLE audit_archive (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  occurred_at TIMESTAMP(6) NOT NULL,
  actor VARCHAR(255) NOT NULL,
  action VARCHAR(64) NOT NULL,
  payload LONGTEXT NULL,
  KEY ix_audit_occurred (occurred_at)
) ENGINE=Aria
  ROW_FORMAT=PAGE
  TRANSACTIONAL=1
  DEFAULT CHARSET=utf8mb4
  COLLATE=utf8mb4_uca1400_ai_ci;
```

ALWAYS prefer InnoDB even for audit logs if any concurrent writer exists ; Aria's table-level locking serialises every writer. Use Aria only when you control exactly one writer process.

---

## Example 5 : MEMORY Lookup Table

```sql
-- 10.6+ : MEMORY engine for ephemeral session lookup
-- ALL ROWS LOST ON SERVER RESTART
CREATE TABLE session_lookup (
  session_token CHAR(64) NOT NULL,
  user_id BIGINT UNSIGNED NOT NULL,
  csrf_token CHAR(64) NOT NULL,
  expires_at INT UNSIGNED NOT NULL,
  PRIMARY KEY (session_token) USING HASH,
  KEY ix_user (user_id) USING HASH
) ENGINE=MEMORY
  MAX_ROWS=1000000;

-- Populate from durable InnoDB on application startup
INSERT INTO session_lookup
SELECT token, user_id, csrf, UNIX_TIMESTAMP(expires_at)
FROM durable_sessions
WHERE expires_at > NOW();
```

NEVER use TEXT or BLOB columns ; MEMORY engine rejects them. ALWAYS pre-load MEMORY tables from a durable source after every restart.

---

## Example 6 : Engine Swap : MyISAM to InnoDB

```sql
-- Step 1 : audit which tables are still MyISAM
SELECT table_schema, table_name, engine,
       ROUND(data_length / 1024 / 1024) AS data_mb,
       ROUND(index_length / 1024 / 1024) AS index_mb
FROM information_schema.tables
WHERE engine = 'MyISAM'
  AND table_schema NOT IN ('mysql', 'information_schema', 'performance_schema');

-- Step 2 : convert one table, off-peak window
-- NEVER use ALGORITHM=INSTANT ; engine change always copies
ALTER TABLE legacy_orders
  ENGINE=InnoDB,
  ROW_FORMAT=DYNAMIC,
  ALGORITHM=COPY,
  LOCK=SHARED;

-- Step 3 : verify
SHOW TABLE STATUS LIKE 'legacy_orders'\G

-- Step 4 : reclaim disk if a global tablespace was used
-- (only relevant if innodb_file_per_table was OFF historically)
OPTIMIZE TABLE legacy_orders;
```

ALWAYS take a backup before any engine swap. ALWAYS run during a maintenance window for tables over 1 GB ; SHARED lock blocks writes for the full COPY duration.

---

## Example 7 : Spider Sharding Setup

```sql
-- 10.6+ : on the Spider router node
-- Step 1 : install the Spider plugin
INSTALL SONAME 'ha_spider';

-- Step 2 : declare each backend as a federated server
CREATE OR REPLACE SERVER backend1
  FOREIGN DATA WRAPPER mysql
  OPTIONS (
    HOST '10.0.0.11',
    DATABASE 'shop',
    USER 'spider_user',
    PASSWORD 'redacted',
    PORT 3306
  );

CREATE OR REPLACE SERVER backend2
  FOREIGN DATA WRAPPER mysql
  OPTIONS (
    HOST '10.0.0.12',
    DATABASE 'shop',
    USER 'spider_user',
    PASSWORD 'redacted',
    PORT 3306
  );

-- Step 3 : on EACH backend create the physical table
-- (run separately on backend1 and backend2)
-- CREATE TABLE shop.orders_p0 (...) ENGINE=InnoDB ;
-- CREATE TABLE shop.orders_p1 (...) ENGINE=InnoDB ;

-- Step 4 : on the Spider router define the partitioned Spider table
CREATE TABLE order_shard (
  id BIGINT UNSIGNED NOT NULL,
  customer_id BIGINT UNSIGNED NOT NULL,
  total_cents BIGINT NOT NULL,
  created_at TIMESTAMP(6) NOT NULL,
  PRIMARY KEY (id, customer_id)
) ENGINE=Spider
  COMMENT='wrapper "mysql", table "orders"'
  PARTITION BY HASH(customer_id) (
    PARTITION p0 COMMENT='srv "backend1", tbl "orders_p0"',
    PARTITION p1 COMMENT='srv "backend2", tbl "orders_p1"'
  );

-- Step 5 : tune timeouts so a dead backend fails fast
SET GLOBAL spider_connect_timeout = 5;
SET GLOBAL spider_net_read_timeout = 30;
SET GLOBAL spider_net_write_timeout = 30;
```

ALWAYS pair each backend with Galera or replication for HA ; Spider's own HA was removed in 10.7.5. NEVER expose Spider over a public network without TLS on every backend connection.

---

## Example 8 : CONNECT Federating a CSV File

```sql
-- 10.6+ : federate a CSV file as a read-mostly SQL table
CREATE TABLE product_feed (
  sku VARCHAR(64) NOT NULL,
  name VARCHAR(255) NOT NULL,
  price_cents BIGINT NOT NULL,
  in_stock TINYINT(1) NOT NULL
) ENGINE=CONNECT
  TABLE_TYPE=CSV
  FILE_NAME='/var/feeds/products.csv'
  SEP_CHAR=','
  QUOTED=1
  HEADER=1;

-- Query like any SQL table
SELECT sku, name FROM product_feed WHERE in_stock = 1 LIMIT 100;

-- ETL into a durable InnoDB table
INSERT INTO product (sku, name, price_cents, in_stock)
SELECT sku, name, price_cents, in_stock FROM product_feed
ON DUPLICATE KEY UPDATE
  name = VALUES(name),
  price_cents = VALUES(price_cents),
  in_stock = VALUES(in_stock);
```

NEVER let untrusted user input control `FILE_NAME` ; that field is a file-system read primitive. ALWAYS read CONNECT once and ETL into InnoDB rather than letting application traffic hit CONNECT directly.

---

## Example 9 : CONNECT Federating ODBC to External DBMS

```sql
-- 10.6+ : federate an external PostgreSQL table via ODBC
CREATE TABLE external_orders (
  id BIGINT NOT NULL,
  customer_email VARCHAR(320) NOT NULL,
  total_cents BIGINT NOT NULL
) ENGINE=CONNECT
  TABLE_TYPE=ODBC
  CONNECTION='DSN=external_pg;UID=reader;PWD=redacted'
  TABNAME='public.orders'
  QUOTED=2;
```

ALWAYS pin the DSN credentials to a read-only role on the external system ; CONNECT inherits whatever privileges the DSN credentials hold. NEVER assume the ODBC bridge is performant for high-volume queries ; row pulls are sequential.

---

## Example 10 : ColumnStore Analytical Sidecar

```sql
-- ColumnStore Enterprise / Community : star-schema fact table
CREATE TABLE sales_fact (
  sale_date DATE NOT NULL,
  store_id INT NOT NULL,
  sku VARCHAR(64) NOT NULL,
  qty INT NOT NULL,
  total_cents BIGINT NOT NULL,
  channel VARCHAR(32) NOT NULL
) ENGINE=ColumnStore;

-- Bulk load via cpimport (shell, NOT SQL)
-- cpimport -m 1 analytics sales_fact /var/data/sales_2026_05.csv

-- Analytical query : full-extent scan, columnar pruning
SELECT sale_date, channel, SUM(total_cents) / 100 AS revenue
FROM sales_fact
WHERE sale_date BETWEEN '2026-01-01' AND '2026-05-31'
GROUP BY sale_date, channel
ORDER BY sale_date;
```

ALWAYS load ColumnStore via `cpimport` for bulk ingestion ; row-by-row INSERT is by design slow. NEVER point an OLTP application at ColumnStore ; columnar storage has no row-level index for `WHERE id = ?` lookups.

---

## Example 11 : Inspecting Engine Use in a Schema

```sql
-- Full engine inventory
SELECT engine, COUNT(*) AS n_tables,
       ROUND(SUM(data_length) / 1024 / 1024) AS total_data_mb,
       ROUND(SUM(index_length) / 1024 / 1024) AS total_index_mb
FROM information_schema.tables
WHERE table_schema NOT IN ('mysql', 'information_schema',
                            'performance_schema', 'sys')
GROUP BY engine
ORDER BY total_data_mb DESC;

-- Available engines on this server
SELECT engine, support, transactions, savepoints
FROM information_schema.engines
WHERE support IN ('YES', 'DEFAULT')
ORDER BY support DESC, engine;
```

---

## Example 12 : Setting the Server Default Engine

```ini
# /etc/mysql/mariadb.conf.d/50-server.cnf
[mysqld]
default_storage_engine = InnoDB
innodb_default_row_format = dynamic
innodb_file_per_table = ON
character-set-server = utf8mb4
collation-server = utf8mb4_uca1400_ai_ci
```

ALWAYS audit `default_storage_engine` and `innodb_default_row_format` on legacy servers before migration ; an old `default_storage_engine = MyISAM` will silently create MyISAM tables for every `CREATE TABLE` that omits `ENGINE=`.

---

## See Also

- Per-engine feature matrix : `methods.md`
- Engine anti-patterns and recovery : `anti-patterns.md`
- Schema design patterns : `mariadb-impl-schema-design` skill (when created)
- DDL algorithms (INSTANT / INPLACE / COPY) : `mariadb-syntax-ddl-alter` skill (when created)
