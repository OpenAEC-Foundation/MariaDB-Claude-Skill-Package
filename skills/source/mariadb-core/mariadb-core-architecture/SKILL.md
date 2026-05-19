---
name: mariadb-core-architecture
description: >
  Use when explaining MariaDB process model, threading, the query execution pipeline, plug-in architecture, in-memory caches, or when reasoning about why a query behaves as it does at the engine layer.
  Prevents the mistake of treating MariaDB as a black box ; surfaces the buffer-pool, log-buffer, thread-pool, and storage-engine boundaries that drive performance.
  Covers mysqld process model, connection threads vs thread-pool plug-in, parser-optimizer-executor stages, storage-engine plug-in API, buffer pool, redo log, undo log, binlog, query cache history, MariaDB vs MySQL optimizer divergence.
  Keywords: mariadb architecture, mysqld process, thread pool, buffer pool, query execution pipeline, storage engine plug-in, why is my query slow, what does mariadb do internally, how does mariadb work, redo log, undo log, binlog, mariadb internals, mariadb architecture diagram, innodb buffer pool, optimizer switch, handler interface, connection lifecycle
license: MIT
compatibility: "Designed for Claude Code. Requires MariaDB 10.6-LTS, 10.11-LTS, 11.x, 12.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# MariaDB Core Architecture

How `mysqld` accepts a connection, parses a query, picks a plan, and reads pages from disk. Read this skill before tuning, before diagnosing slow queries, and before assuming MySQL behavior applies to MariaDB.

## Quick Reference

- MariaDB = single multi-threaded `mysqld` process with per-connection threads or a built-in thread-pool plug-in.
- Query pipeline : parser (lex + yacc) : optimizer (cost-based, `optimizer_switch` flags) : executor : storage-engine handler API.
- Storage engines are plug-ins ; choice happens at `CREATE TABLE ... ENGINE=` time. InnoDB is the default ; Aria, MyISAM, MEMORY, MyRocks, Spider, S3, Archive, CSV, Sequence, MERGE are pluggable.
- In-memory caches per engine ; biggest is the InnoDB buffer pool, sized 60-80% of RAM on a dedicated DB host.
- Diverges from MySQL : different `optimizer_switch` flags (e.g. `condition_pushdown_for_subquery`, `rowid_filter`, `split_materialized`), built-in thread-pool (not Enterprise-only), query cache kept (deprecated, off by default), GTID format incompatible, JSON is LONGTEXT alias.
- ALWAYS run `EXPLAIN` before tuning : optimizer choices are visible in the plan, not in the schema.
- NEVER set `innodb_buffer_pool_size` blindly to 50% RAM on a dedicated DB host. Target 60-80%.
- NEVER enable every flag in `optimizer_switch`. Several are OFF by default (e.g. `mrr`, `mrr_cost_based`, `index_merge_sort_intersection`) because the cost model rejects them in common cases.

## Process Model

`mysqld` is a single OS process with these thread roles :

| Thread role | Purpose | Configurable via |
|-------------|---------|------------------|
| Connection thread (one-thread-per-connection) | Handles one client session end-to-end | Default in `thread_handling=one-thread-per-connection` |
| Thread-pool worker | Shared workers across many sessions, grouped by `thread_pool_size` (default = CPU count) | `thread_handling=pool-of-threads` |
| Page cleaner | Flushes dirty pages from buffer pool to disk | `innodb_page_cleaners` (default 4) |
| InnoDB IO threads | Read-ahead, async write, log writes | `innodb_read_io_threads`, `innodb_write_io_threads` (default 4 each) |
| Purge threads | Remove unused undo records | `innodb_purge_threads` (default 4) |
| Master thread | Background flushing schedule, doublewrite buffer | Internal |
| Replication threads | IO thread (fetches binlog from primary), SQL thread or workers (applies events) | `slave_parallel_threads`, `slave_parallel_mode` |

**Thread-pool decision rule** :
- ALWAYS use one-thread-per-connection for <100 concurrent connections (lower latency, default).
- ALWAYS switch to `thread_handling=pool-of-threads` when serving 1000+ concurrent connections in an OLTP workload. Set `thread_pool_size = number of CPU cores`.
- NEVER use thread-pool for analytics workloads with long-running queries ; the stall-detection (`thread_pool_stall_limit` default 500ms) does not fully prevent head-of-line blocking.

## Query Execution Pipeline

A statement traverses these stages, in order :

1. **Receive** : connection thread reads bytes, identifies command type (COM_QUERY, COM_STMT_PREPARE, etc.).
2. **Parse** : `sql_yacc.yy` + `sql_lex.cc` build a parse tree (`LEX` struct). Syntax errors raised here.
3. **Resolve names** : tables, columns, functions resolved against the data dictionary and grants.
4. **Optimize** : cost-based optimizer evaluates join order, access methods (full scan, index, index-merge), uses `optimizer_switch` flags to enable or disable transformations. Trace via `optimizer_trace=enabled=on`.
5. **Execute** : executor walks the plan, calls into the storage-engine handler API (`sql/handler.h`) for each table access : `rnd_init`, `rnd_next`, `index_read`, `index_next`, `write_row`, `update_row`, `delete_row`.
6. **Return** : rows streamed to the client over the connection thread.

**Diagnostic chain** :
- `EXPLAIN <stmt>` : returns the plan without running the query.
- `EXPLAIN FORMAT=JSON <stmt>` : richer plan with cost estimates (10.1+).
- `ANALYZE SELECT <stmt>` : actually runs and adds observed `r_rows`, `r_filtered`, `r_total_time_ms` (10.1+).
- ALWAYS use `ANALYZE SELECT`, NEVER `ANALYZE UPDATE` or `ANALYZE DELETE` for diagnostic purposes : those execute the mutation.

## Storage-Engine Plug-In Architecture

Storage engines are loadable plug-ins implementing `class handler` defined in `sql/handler.h`. Choice happens per table.

```sql
-- 10.6+
CREATE TABLE inventory (
   id  INT PRIMARY KEY,
   sku VARCHAR(64)
) ENGINE = InnoDB;

CREATE TABLE log_ingest (
   ts   TIMESTAMP,
   line LONGTEXT
) ENGINE = Aria;
```

| Engine | Strength | Default | Loaded by default |
|--------|----------|---------|-------------------|
| InnoDB | Transactional, row-locking, foreign keys, crash-safe | Yes | Yes |
| Aria | Crash-safe non-transactional, used for internal temp tables | No | Yes |
| MyISAM | Read-heavy non-transactional, legacy | No | Yes |
| MEMORY | In-RAM, lost on restart | No | Yes |
| MyRocks | LSM-tree, write-heavy, high compression | No | Plug-in install |
| Spider | Sharded across nodes | No | Plug-in install |
| S3 | Read-only Amazon S3 backing | No | Plug-in install |
| Archive | Compressed insert-only | No | Plug-in install |
| CSV | Plain-text CSV | No | Yes |
| Sequence | Virtual number sequences | No | Yes |
| MERGE | Logical union of identical MyISAM tables | No | Yes |
| ColumnStore | Columnar analytics, separate distribution | No | Separate install |

ALWAYS use InnoDB unless a documented constraint forces another engine (e.g. MyRocks for petabyte write-heavy logs, Aria for crash-safe non-transactional staging, MEMORY for short-lived lookup tables).

## In-Memory Caches

| Cache | Engine | Sizing variable | Role |
|-------|--------|-----------------|------|
| Buffer pool | InnoDB | `innodb_buffer_pool_size` (default 128 MiB) | Caches data and index pages ; target 60-80% RAM on dedicated host |
| Log buffer | InnoDB | `innodb_log_buffer_size` (default 16 MiB) | Buffers redo log records before flush |
| Adaptive hash index | InnoDB | `innodb_adaptive_hash_index` (default ON) | Auto-built hash on hot pages |
| Pagecache | Aria | `aria_pagecache_buffer_size` (default 128 MiB) | Aria data and index pages |
| Key cache | MyISAM | `key_buffer_size` (default 128 MiB) | MyISAM index blocks only ; data goes through OS cache |
| Query cache | Server-level | `query_cache_size` (default 0, type OFF) | Deprecated, do NOT enable in write-heavy workloads |

**Buffer-pool hit-ratio rule** : monitor `Innodb_buffer_pool_reads` (physical disk reads) versus `Innodb_buffer_pool_read_requests` (total). If the delta of `reads` over time is < 1% of the delta of `read_requests`, the pool is sized correctly. If `Innodb_buffer_pool_wait_free` is increasing, the pool is too small.

**10.11.12+, 11.4.6+, 11.8.2+ change** : `innodb_buffer_pool_chunk_size` is deprecated and ignored ; the pool resizes in 1 MB increments up to `innodb_buffer_pool_size_max` (set at startup). `SET GLOBAL innodb_buffer_pool_size` now blocks until the resize completes.

## Disk Structures

| Structure | File pattern | Purpose | Controlled by |
|-----------|--------------|---------|---------------|
| System tablespace | `ibdata1` | Doublewrite buffer, internal metadata, undo (legacy) | `innodb_data_file_path` |
| Per-table tablespace | `<schema>/<table>.ibd` | Table data + indexes | `innodb_file_per_table=ON` (default since 5.6) |
| Undo tablespace | `undo001`, `undo002`, ... | MVCC undo records (separated 10.6+) | `innodb_undo_tablespaces` |
| Redo log | `ib_logfile0` (single file 10.5+) | Crash recovery, write-ahead log | `innodb_log_file_size` |
| Binary log | `<basename>.000001`, ... | Replication, point-in-time recovery | `log_bin`, `binlog_format` |
| Aria log | `aria_log.<n>` | Aria crash recovery | `aria_log_file_size` |

**10.5+ change** : redo log is a single file (`ib_logfile0`), `innodb_log_files_in_group` deprecated. From 10.9+, `innodb_log_file_size` can be set dynamically without restart.

## Connection Lifecycle

```
client connect (TCP/socket/named-pipe)
   :
   authentication plug-in (ed25519, mysql_native_password, unix_socket, gssapi, pam)
   :
   THD struct allocated (one per connection, holds session state)
   :
   per-query : parse : resolve : optimize : execute : handler-API : result-set
   :
   close : THD destroyed, per-thread buffers (sort_buffer, read_rnd_buffer, join_buffer, tmp_table) freed
```

Per-thread buffers multiply by `max_connections`. NEVER set `sort_buffer_size=256M` and `max_connections=1000` ; that's a 256 GB worst-case ceiling on top of the buffer pool.

## Optimizer Divergence From MySQL

MariaDB has its own optimizer code path with flags MySQL does not have. Examples :

| Flag | Default | Introduced | MariaDB-only |
|------|---------|------------|--------------|
| `condition_pushdown_for_derived` | ON | 10.2.2+ | Yes |
| `condition_pushdown_for_subquery` | ON | 10.4+ | Yes |
| `condition_pushdown_from_having` | ON | 10.4.3+ | Yes |
| `rowid_filter` | ON | 10.4.3+ | Yes |
| `split_materialized` | ON | 10.3.4+ | Yes |
| `hash_join_cardinality` | ON/OFF | 10.6.13+ | Yes |
| `not_null_range_scan` | OFF | 10.5+ | Yes |
| `sargable_casefold` | ON | 11.3.0+ | Yes |
| `mrr`, `mrr_cost_based`, `mrr_sort_keys` | OFF | 5.3+ | Both, but defaults differ |

ALWAYS check MariaDB KB `optimizer-switch` for the per-version default before assuming a MySQL tuning recipe applies. See `references/methods.md` for the full table.

## Query Cache : History

- Introduced in MySQL 4.0, inherited by MariaDB.
- Disabled by default in MariaDB 10.1.7+ (`query_cache_type=OFF`, `query_cache_size=0`).
- Kept in MariaDB after MySQL 8.0 removed it.
- NEVER enable on write-heavy workloads : every write to a cached table invalidates all related entries, causing lock contention on the cache mutex.

## When Not To Use This Skill

- For tuning specific subsystems beyond the architecture surface : use `mariadb-impl-performance-tuning` or `mariadb-impl-query-optimization`.
- For storage-engine selection deep-dive : use `mariadb-core-storage-engines`.
- For replication architecture : use `mariadb-core-replication-model`.

## References

- `references/methods.md` : full thread roles, optimizer-switch table, handler API entry points, status variables for introspection.
- `references/examples.md` : SHOW PROCESSLIST, SHOW ENGINE INNODB STATUS, EXPLAIN walkthrough, optimizer_switch toggling, log file paths.
- `references/anti-patterns.md` : sizing mistakes, optimizer mis-tuning, query-cache misuse, thread-pool misuse, MySQL-tuning carry-over.
- Authoritative sources : <https://mariadb.com/kb/en/thread-pool-in-mariadb/>, <https://mariadb.com/kb/en/optimizer-switch/>, <https://mariadb.com/kb/en/innodb-buffer-pool/>, <https://mariadb.com/kb/en/innodb-system-variables/>, <https://mariadb.com/kb/en/storage-engines/>, <https://mariadb.com/kb/en/explain/>, <https://mariadb.com/kb/en/innodb-redo-log/>, <https://mariadb.com/kb/en/query-cache/>.
