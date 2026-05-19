# MariaDB Storage Engines : Complete Feature Matrix

Per-engine reference of capabilities, locking model, indexing, row formats, and authoritative documentation links.

---

## InnoDB

Default storage engine since MariaDB 10.2. ALWAYS the right choice for transactional user data.

| Capability | Value |
|------------|-------|
| ACID | Full (atomicity, consistency, isolation, durability) |
| Locking granularity | Row-level (via record-level + gap locks) |
| Foreign keys | Yes (enforced) |
| MVCC | Yes (undo log) |
| Crash recovery | Yes (redo log + double-write buffer) |
| Default page size | 16 KB (4K / 8K / 32K / 64K configurable at instance init) |
| Row formats | DYNAMIC (default since 10.2), COMPACT, COMPRESSED, REDUNDANT |
| Max row size | ~half page size for non-DYNAMIC ; long columns off-page on DYNAMIC |
| Full-text index | Yes (since 10.0.5) |
| Spatial index | Yes (R-tree on GEOMETRY) |
| Hash index | No (adaptive hash index is an internal optimisation, not user-controllable) |
| Encryption at rest | Yes (innodb_encrypt_tables, innodb_encrypt_log) |
| Online DDL | INSTANT (10.3+ end-of-table, 10.4+ any position), INPLACE, COPY |
| Compression | Yes (ROW_FORMAT=COMPRESSED, zlib) ; incompatible with INSTANT ALTER |
| Replication | Row-based, statement-based, mixed ; binlog-aware |
| Galera compatibility | Full |
| Tablespace | innodb_file_per_table=ON (default since 5.6 era) |
| Auto-increment | Yes ; persists across restart since 10.2.4 (innodb_autoinc_persist) |
| Generated columns | Yes (VIRTUAL and PERSISTENT) |

ALWAYS keep `innodb_file_per_table=ON` so each table lives in its own `.ibd` file. NEVER fall back to the shared tablespace for new tables ; it cannot be reclaimed without a full dump and reload.

KB : https://mariadb.com/kb/en/innodb/  
Row formats : https://mariadb.com/kb/en/innodb-row-formats/  
System variables : https://mariadb.com/kb/en/innodb-system-variables/  
Online DDL : https://mariadb.com/kb/en/innodb-online-ddl-overview/

---

## Aria

Crash-safe MyISAM replacement. ALWAYS used internally for `mysql.*` system tables and on-disk temporary tables.

| Capability | Value |
|------------|-------|
| ACID | No (transactional flag means "crash-safe", not transactions) |
| Locking granularity | Table-level |
| Foreign keys | No |
| MVCC | No |
| Crash recovery | Yes (write-ahead log when ROW_FORMAT=PAGE, default) |
| Page size | 8 KB (configurable via aria_block_size) |
| Row formats | PAGE (default), FIXED, DYNAMIC |
| Full-text index | Yes (legacy MyISAM-style implementation) |
| Spatial index | No |
| Hash index | No |
| Encryption at rest | Yes (aria_encrypt_tables) |
| Online DDL | COPY only ; no INPLACE or INSTANT |
| Compression | No |
| Replication | Statement-based ; row-based works but Aria is rare in user schemas |
| Galera compatibility | Yes for system tables (replicated via DDL) ; user-data Aria tables are NOT replicated by Galera as a default |
| Generated columns | Yes |

ALWAYS know that Aria's `TRANSACTIONAL=1` only enables crash-safety on the write-ahead log ; there is no BEGIN / COMMIT semantics. NEVER place concurrent user-write workloads on Aria ; table-level locking will serialise every writer.

KB : https://mariadb.com/kb/en/aria-storage-engine/  
Aria storage formats : https://mariadb.com/kb/en/aria-storage-formats/

---

## MyISAM

Legacy engine. NEVER choose MyISAM for new tables.

| Capability | Value |
|------------|-------|
| ACID | No |
| Locking granularity | Table-level (read locks are shared, write locks exclusive) |
| Foreign keys | No |
| MVCC | No |
| Crash recovery | No (manual REPAIR TABLE on corruption) |
| Row formats | FIXED, DYNAMIC, COMPRESSED, PACKED |
| Full-text index | Yes (legacy) |
| Spatial index | Yes |
| Hash index | No |
| Encryption at rest | No |
| Online DDL | COPY only |
| Replication | Yes (statement-based safer than row-based) |
| Galera compatibility | No (Galera certifies only InnoDB writes by default) |
| Auto-increment | Yes (per table, lost on crash if uncommitted) |

ALWAYS migrate MyISAM tables to InnoDB via `ALTER TABLE x ENGINE=InnoDB`. NEVER trust MyISAM to survive a power loss ; index files corrupt silently and require offline `myisamchk`.

KB : https://mariadb.com/kb/en/myisam-storage-engine/

---

## ColumnStore

Columnar analytical engine. Lives in a separate MariaDB ColumnStore distribution.

| Capability | Value |
|------------|-------|
| ACID | Partial (snapshot isolation, durable bulk inserts) |
| Locking granularity | Extent-level (1M-row blocks) |
| Foreign keys | No |
| MVCC | Snapshot-based |
| Crash recovery | Yes (versioned extents) |
| Storage layout | Columnar (one file per column per extent) |
| Indexing | None ; full extent scans with extent-level min/max pruning |
| Full-text index | No |
| Spatial index | No |
| Encryption at rest | Filesystem-level only |
| Online DDL | Limited ; bulk loads via cpimport |
| Compression | Yes (per-column adaptive) |
| Replication | None (use ETL from InnoDB master) |
| Galera compatibility | No |

ALWAYS load ColumnStore via `cpimport` for bulk ingestion ; row-by-row INSERT is by design slow. NEVER point an OLTP application at ColumnStore for `WHERE id = ?` lookups ; columnar storage has no row-level index.

KB : https://mariadb.com/docs/columnstore/  
ColumnStore is exclusive to MariaDB Enterprise Server and the community ColumnStore distribution. It is NOT included in the standard MariaDB Server binaries.

---

## Spider

Sharding / federation engine. Built into MariaDB since 10.0.4.

| Capability | Value |
|------------|-------|
| ACID | Depends on backend engine (typically InnoDB) |
| Locking granularity | Depends on backend |
| Foreign keys | No (cannot be enforced across backends) |
| MVCC | Depends on backend |
| Crash recovery | Depends on backend |
| Storage | None on the Spider node ; data lives on backends |
| Indexing | Mirrors backend ; no condition push-down for complex predicates |
| Full-text index | No |
| Spatial index | No |
| Encryption at rest | Depends on backend |
| Online DDL | DDL must be replicated to every backend ; no atomic guarantee |
| Replication | Combined with Galera or master-slave per backend |
| HA built-in | Removed in 10.7.5 (MDEV-28479) |
| Partitioning | Native ; one partition per backend |

ALWAYS pair Spider with Galera or replication on each backend for HA. NEVER deploy Spider without configuring `spider_connect_timeout`, `spider_net_read_timeout`, and `spider_net_write_timeout` ; default values can mask backend failures for minutes. NEVER expect transparent transaction semantics across shards ; Spider supports XA but it is best-effort.

KB : https://mariadb.com/kb/en/spider-storage-engine-overview/  
Spider installation : https://mariadb.com/kb/en/installing-spider/  
Removed HA : https://jira.mariadb.org/browse/MDEV-28479

---

## CONNECT

Federation engine for external data. Available since MariaDB 10.0.

| Capability | Value |
|------------|-------|
| ACID | No |
| Locking granularity | File-level (when applicable) |
| Foreign keys | No |
| MVCC | No |
| Crash recovery | No (data lives outside MariaDB) |
| Storage | None native ; reads files or external DBMS |
| Supported TABLE_TYPE | CSV, FMT, XML, JSON, INI, DBF, DOS, FIX, BIN, XCOL, OCCUR, PIVOT, VEC, ODBC, JDBC, MYSQL, MONGO, PROXY, TBL, VIR, DIR, WMI, MAC, ZIP |
| Indexing | XINDEX (CONNECT's own index files), but typically full-scan |
| Full-text index | No |
| Spatial index | No |
| Encryption at rest | Depends on backing format |
| Online DDL | N/A (CONNECT defines metadata only) |
| Replication | Statement-based, but federated data drifts unless ETL coordinates |
| Galera compatibility | No |

ALWAYS treat CONNECT as a read-mostly federation layer. NEVER allow user input to control `FILE_NAME`, `CONNECTION`, or `TABNAME` ; these are file-system and credential primitives. NEVER expect transactional semantics on a CONNECT table.

KB : https://mariadb.com/kb/en/connect/  
CONNECT table types : https://mariadb.com/kb/en/connect-table-types/

---

## MEMORY (HEAP)

In-memory engine. Data is LOST on every server restart.

| Capability | Value |
|------------|-------|
| ACID | No |
| Locking granularity | Table-level |
| Foreign keys | No |
| MVCC | No |
| Crash recovery | No (rows lost on shutdown / crash / restart) |
| Storage | RAM only ; table definition file persisted but rows empty after restart |
| Indexing | HASH (default), BTREE (optional via `USING BTREE`) |
| Max indexes per table | 64 |
| Column type restrictions | NO TEXT / BLOB columns ; VARCHAR allowed |
| Full-text index | No |
| Spatial index | No |
| Encryption at rest | N/A |
| Online DDL | COPY |
| Replication | Statement-based ; row-based fails because replica memory is empty after restart |
| Galera compatibility | Limited ; MEMORY tables do not replicate row state |

ALWAYS know that MEMORY tables hold their schema across restart but are populated empty. NEVER store any value of record in MEMORY. ALWAYS use `MAX_ROWS` and `MAX_HEAP_TABLE_SIZE` to bound memory consumption.

KB : https://mariadb.com/kb/en/memory-storage-engine/

---

## Cross-Engine Feature Matrix

| Feature | InnoDB | Aria | MyISAM | ColumnStore | Spider | CONNECT | MEMORY |
|---------|--------|------|--------|-------------|--------|---------|--------|
| Transactions | yes | no | no | partial | depends | no | no |
| Row-level locking | yes | no | no | no | depends | no | no |
| Foreign keys | yes | no | no | no | no | no | no |
| MVCC | yes | no | no | snapshot | depends | no | no |
| Crash-safe | yes | yes | no | yes | depends | no | no |
| Full-text | yes (10.0.5+) | yes | yes | no | no | no | no |
| Spatial | yes | no | yes | no | no | no | no |
| Persistent | yes | yes | yes | yes | yes | yes | NO |
| Encrypted at rest | yes | yes | no | filesystem | depends | depends | no |
| Replication-safe | yes | partial | partial | no | depends | no | partial |
| Galera-friendly | yes | system tables only | no | no | with backends | no | no |
| INSTANT ALTER | 10.3+ | no | no | no | no | no | no |
| Generated columns | yes | yes | yes | no | depends | no | yes |

## INFORMATION_SCHEMA Engine Inventory

```sql
-- List all available engines on this server
SELECT engine, support, transactions, xa, savepoints
FROM information_schema.engines
ORDER BY support DESC, engine;
```

`support` values : `DEFAULT` (the chosen default), `YES` (available), `NO` (compiled but disabled), `DISABLED`.

```sql
-- List every table that is NOT InnoDB or Aria in the current database
SELECT table_schema, table_name, engine, table_rows, data_length, index_length
FROM information_schema.tables
WHERE table_schema = DATABASE()
  AND engine NOT IN ('InnoDB', 'Aria')
ORDER BY data_length DESC;
```

ALWAYS run this audit on every legacy schema before migration.

## Engine Conversion Commands

```sql
-- Convert MyISAM to InnoDB ; full rewrite, table is SHARED-locked
ALTER TABLE legacy_orders ENGINE=InnoDB,
  ALGORITHM=COPY, LOCK=SHARED;

-- Convert Aria to InnoDB ; full rewrite
ALTER TABLE legacy_audit ENGINE=InnoDB,
  ALGORITHM=COPY, LOCK=SHARED;

-- Convert to MEMORY ; FAILS if TEXT/BLOB columns are present
ALTER TABLE ephemeral_lookup ENGINE=MEMORY;
```

NEVER convert to MEMORY without first auditing the column types ; the ALTER will fail mid-COPY if TEXT or BLOB columns exist. ALWAYS run the conversion during a maintenance window for any table over 1 GB ; the COPY algorithm writes a full new copy on disk.

## See Also

- Examples per engine : `examples.md`
- Anti-patterns and recovery procedures : `anti-patterns.md`
- Architecture overview : `mariadb-core-architecture` skill
