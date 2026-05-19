---
name: mariadb-core-storage-engines
description: >
  Use when choosing a storage engine for a new table, migrating from MyISAM to InnoDB, understanding engine-specific behaviors, or reasoning about ACID guarantees on MariaDB 10.6+.
  Prevents the common mistake of using MyISAM for transactional data, ColumnStore for OLTP, MEMORY for persistent caches, or mixing engines without understanding cross-engine query limitations.
  Covers InnoDB (default, ACID), Aria (crash-safe MyISAM replacement, system tables), MyISAM (legacy), ColumnStore (analytical), Spider (sharding), CONNECT (federated), MEMORY (in-memory), and a deterministic decision tree per workload.
  Keywords: storage engine, InnoDB, Aria, MyISAM, ColumnStore, Spider, CONNECT, MEMORY, ENGINE=, which engine should I use, no transactions, table locking, row locking, ACID, MVCC, full-text search, columnar analytics, federation, sharding, in-memory cache, foreign keys missing, table corrupted on restart, slow concurrent inserts, why is my table locking, can I run analytics on InnoDB, federated table, CSV table, my cache is gone after restart
license: MIT
compatibility: "Designed for Claude Code. Requires MariaDB 10.6-LTS, 10.11-LTS, 11.x, 12.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# mariadb-core-storage-engines

Deterministic guidance for picking and operating MariaDB storage engines.

## Quick Reference

**InnoDB is the right default 99% of the time.** It is ACID, row-locked, MVCC-backed, foreign-key-aware, crash-safe, and the only engine to pick for new transactional workloads.

**MyISAM lacks transactions, foreign keys, and crash-safety.** NEVER use MyISAM for new tables. Use Aria as the MyISAM replacement when you need crash-safety on system-like data.

**ColumnStore is analytical-only.** It cannot serve OLTP, runs as a separate distribution, and is column-oriented. NEVER point an OLTP application at ColumnStore.

**MEMORY engine loses all rows on server restart.** Use it only for ephemeral lookup tables that can be re-populated from a source of truth.

**Spider federates across MariaDB backends.** Spider's high-availability features were removed in 10.7.5 ; combine Spider with Galera or replication for HA.

**CONNECT federates external data sources** (CSV, XML, JSON files, ODBC, JDBC, MongoDB). NEVER use CONNECT for high-throughput writes ; it is a federation layer, not a storage layer.

## Decision Tree : Which Engine

```
Need ACID + foreign keys + concurrent writes ?
  YES -> InnoDB (default, 99% of cases)
  NO  -> continue

Internal system table or on-disk temporary table ?
  YES -> Aria (managed by MariaDB itself, do NOT pick Aria for user tables)
  NO  -> continue

Analytical OLAP scans over billions of rows ?
  YES -> ColumnStore (separate distribution, never OLTP)
  NO  -> continue

Horizontally shard a single logical table across many MariaDB nodes ?
  YES -> Spider (combine with Galera or replication for HA)
  NO  -> continue

Read external data files or other DBMS as a SQL table ?
  YES -> CONNECT (CSV / XML / JSON / ODBC / JDBC / MongoDB)
  NO  -> continue

Ephemeral lookup table that can be lost on restart ?
  YES -> MEMORY (HEAP) ; no TEXT / BLOB columns ; default HASH index
  NO  -> InnoDB

Legacy MyISAM table found in your schema ?
  YES -> ALTER TABLE x ENGINE=InnoDB; (NEVER leave it on MyISAM)
```

ALWAYS state the engine explicitly in `CREATE TABLE` so the choice is visible in code review.

## Core Patterns

### Pattern 1 : InnoDB as the default

```sql
-- 10.6+ : explicit engine, charset, row format
CREATE TABLE orders (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  customer_id BIGINT UNSIGNED NOT NULL,
  total_cents BIGINT NOT NULL,
  created_at TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6),
  FOREIGN KEY (customer_id) REFERENCES customer (id)
) ENGINE=InnoDB
  ROW_FORMAT=DYNAMIC
  DEFAULT CHARSET=utf8mb4
  COLLATE=utf8mb4_uca1400_ai_ci;
```

ALWAYS specify `ENGINE=InnoDB` even though it is the default. ALWAYS specify `ROW_FORMAT=DYNAMIC` so off-page storage of long variable-length columns is unambiguous. ALWAYS specify `DEFAULT CHARSET=utf8mb4` ; stock 10.6 and 10.11 still ship with `latin1` as the global default.

### Pattern 2 : Engine swap from MyISAM to InnoDB

```sql
-- Audit a legacy schema
SELECT table_schema, table_name, engine
FROM information_schema.tables
WHERE table_schema = DATABASE()
  AND engine NOT IN ('InnoDB', 'Aria')
ORDER BY table_schema, table_name;

-- Convert one table, version-safe
ALTER TABLE legacy_orders ENGINE=InnoDB,
  ALGORITHM=COPY, LOCK=SHARED;
```

NEVER use `ALGORITHM=INSTANT` for an engine swap ; engine change always requires a full table rewrite. NEVER swap an engine on a busy table during peak hours ; the table is fully locked for SHARED access during the COPY rewrite.

### Pattern 3 : MEMORY for ephemeral lookup

```sql
-- 10.6+ : ephemeral session-state, lost on restart
CREATE TABLE session_lookup (
  session_token CHAR(64) NOT NULL PRIMARY KEY,
  user_id BIGINT UNSIGNED NOT NULL,
  expires_at INT UNSIGNED NOT NULL,
  INDEX ix_user (user_id) USING HASH
) ENGINE=MEMORY
  MAX_ROWS=100000;
```

NEVER store any value of record in MEMORY tables ; a restart wipes them. NEVER use TEXT or BLOB columns ; MEMORY rejects them. ALWAYS size `MAX_ROWS` to bound memory usage.

### Pattern 4 : Aria for system-like crash-safe storage

```sql
-- 10.6+ : crash-safe but non-transactional ; use only for system-like tables
CREATE TABLE audit_archive (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  event_at TIMESTAMP(6) NOT NULL,
  payload TEXT NOT NULL
) ENGINE=Aria
  ROW_FORMAT=PAGE
  TRANSACTIONAL=1;
```

ALWAYS know that `TRANSACTIONAL=1` on Aria means "crash-safe", NOT "supports BEGIN / COMMIT". Aria has no MVCC and uses table-level locking. NEVER pick Aria for concurrent user data.

### Pattern 5 : Spider sharding shell

```sql
-- 10.6+ : on the shard-router node
INSTALL SONAME 'ha_spider';

CREATE SERVER backend1
  FOREIGN DATA WRAPPER mysql
  OPTIONS (HOST '10.0.0.11', DATABASE 'shop', USER 'spider', PASSWORD '...', PORT 3306);

CREATE TABLE order_shard (
  id BIGINT UNSIGNED NOT NULL,
  customer_id BIGINT UNSIGNED NOT NULL,
  total_cents BIGINT NOT NULL,
  PRIMARY KEY (id, customer_id)
) ENGINE=Spider
  PARTITION BY HASH(customer_id) (
    PARTITION p0 COMMENT = 'srv "backend1", table "orders_p0"',
    PARTITION p1 COMMENT = 'srv "backend2", table "orders_p1"'
  );
```

ALWAYS pair Spider with Galera or replication on each backend for HA ; Spider's own HA was removed in 10.7.5. NEVER deploy Spider over a slow WAN without `spider_connect_timeout` and `spider_net_read_timeout` tuned.

### Pattern 6 : CONNECT to an external CSV

```sql
-- 10.6+ : federate a CSV file as a SQL table
CREATE TABLE product_feed (
  sku VARCHAR(64) NOT NULL,
  name VARCHAR(255) NOT NULL,
  price_cents BIGINT NOT NULL
) ENGINE=CONNECT
  TABLE_TYPE=CSV
  FILE_NAME='/var/feeds/products.csv'
  SEP_CHAR=','
  QUOTED=1
  HEADER=1;
```

NEVER write heavy transactional traffic to a CONNECT table ; it has no real storage engine of its own. NEVER let untrusted user input control `FILE_NAME` ; that is a file-system read primitive.

### Pattern 7 : ColumnStore as an analytical sidecar

```sql
-- ColumnStore Enterprise / Community : analytical-only
CREATE TABLE sales_fact (
  sale_date DATE NOT NULL,
  store_id INT NOT NULL,
  sku VARCHAR(64) NOT NULL,
  qty INT NOT NULL,
  total_cents BIGINT NOT NULL
) ENGINE=ColumnStore;
```

ALWAYS treat ColumnStore as a separate distribution that is loaded from InnoDB via batch ETL or change-data-capture. NEVER point an OLTP application at ColumnStore for row-by-row UPDATE / DELETE ; performance is by design slow for that pattern.

## Engine Feature Matrix (Quick Reference)

| Engine | ACID | Locking | FK | MVCC | Crash-safe | Full-text | Spatial | Default |
|--------|------|---------|----|------|------------|-----------|---------|---------|
| InnoDB | yes | row | yes | yes | yes (redo log) | yes (10.0.5+) | yes | YES (default since 10.2) |
| Aria | no | table | no | no | yes (PAGE row format) | yes | no | no |
| MyISAM | no | table | no | no | no | yes | yes | no (legacy) |
| ColumnStore | no | block | no | no | yes (snapshot) | no | no | no (separate dist) |
| Spider | depends on backend | depends | depends | depends | depends | depends | depends | no |
| CONNECT | no | none (read-mostly) | no | no | no | no | no | no |
| MEMORY | no | table | no | no | no (lost on restart) | no | no | no |

Full method-level matrix in `references/methods.md`.

## Cross-Engine Query Limits

ALWAYS know that a multi-statement transaction touching both InnoDB and MyISAM rows commits the MyISAM writes immediately and only the InnoDB writes participate in the transaction. NEVER mix engines inside a transactional boundary that needs all-or-nothing semantics.

ALWAYS know that foreign keys are forbidden in partitioned tables and forbidden across engines ; a FK can only point from one InnoDB table to another InnoDB table.

ALWAYS know that JOINing an InnoDB table to a CONNECT or Spider table forces a row-by-row pull through the federation layer ; the optimiser has no statistics on the remote side.

## Anti-Patterns (See references/anti-patterns.md)

1. MyISAM for orders / users (no transactions, crash-loss)
2. ColumnStore for OLTP (write amplification, latency)
3. MEMORY for "cache" without considering restart-loss
4. Mixing engines in a JOIN that crosses transaction boundaries
5. Spider over slow WAN without connection pooling
6. Aria for high-concurrency writes (still table-locking semantics)
7. InnoDB with innodb_file_per_table=OFF (shared tablespace bloats)
8. ENGINE= without ROW_FORMAT specification on tables with long columns

## Version Notes

- MariaDB 10.2 : InnoDB became the global default ; DYNAMIC became the default `ROW_FORMAT`.
- MariaDB 10.3 : InnoDB instant ADD COLUMN at the end of the table.
- MariaDB 10.4 : InnoDB instant ADD COLUMN at any position, instant DROP COLUMN.
- MariaDB 10.6 : atomic DDL via binary log ; ignored indexes ; Aria continues as system-table storage.
- MariaDB 10.7.5 : Spider's built-in high-availability features removed (MDEV-28479).
- MariaDB 11.6 : default charset migrated from `latin1` to `utf8mb4` ; affects new-table defaults across all engines.

## Reference Links

- InnoDB overview : https://mariadb.com/kb/en/innodb/
- InnoDB row formats : https://mariadb.com/kb/en/innodb-row-formats/
- Aria overview : https://mariadb.com/kb/en/aria-storage-engine/
- MyISAM overview : https://mariadb.com/kb/en/myisam-storage-engine/
- ColumnStore : https://mariadb.com/docs/columnstore/
- Spider overview : https://mariadb.com/kb/en/spider-storage-engine-overview/
- CONNECT overview : https://mariadb.com/kb/en/connect/
- MEMORY overview : https://mariadb.com/kb/en/memory-storage-engine/

See `references/methods.md` for the complete per-engine feature matrix, `references/examples.md` for full working examples per engine, and `references/anti-patterns.md` for in-depth anti-pattern walkthroughs.
