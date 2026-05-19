# MariaDB Core Architecture : Methods Reference

Complete inventory of process structures, optimizer flags, storage-engine handler entry points, and introspection variables.

## 1. Process and Thread Model

### Thread types

| Thread role | Spawned by | Controlling variable | Default |
|-------------|-----------|----------------------|---------|
| Connection thread | `mysqld_main()` accept loop | `thread_handling=one-thread-per-connection` | Default mode |
| Thread-pool worker | Pool dispatcher | `thread_pool_size` | Number of CPUs |
| Page cleaner | Buffer pool subsystem | `innodb_page_cleaners` | 4 |
| InnoDB read IO | Storage engine init | `innodb_read_io_threads` | 4 |
| InnoDB write IO | Storage engine init | `innodb_write_io_threads` | 4 |
| Purge thread | Storage engine init | `innodb_purge_threads` | 4 |
| Master thread | Storage engine init | None (one per server) | Always 1 |
| Replica IO thread | `START SLAVE` / `START REPLICA` | None (per source channel) | Always 1 |
| Replica SQL thread | `START SLAVE` | `slave_parallel_threads` | 0 (single-threaded) |
| Galera applier | wsrep provider | `wsrep_slave_threads` | 1 |
| Event scheduler | Event subsystem | `event_scheduler` | OFF by default on most distros |

### Thread-pool variables (10.6+, 11.x, 12.x)

| Variable | Default (Unix) | Role |
|----------|----------------|------|
| `thread_pool_size` | Number of CPUs | Number of thread groups ; equals max statements running simultaneously |
| `thread_pool_max_threads` | 65536 | Hard cap on threads in the pool |
| `thread_pool_stall_limit` | 500 ms | Stall-detection interval ; if a group stalls, a worker is woken or created |
| `thread_pool_idle_timeout` | 60 s | Idle worker exit timeout, shrinks the pool on demand decrease |
| `thread_pool_oversubscribe` | 3 | Concurrent active workers per group, balances CPU-cache vs context-switching |

Source : <https://mariadb.com/kb/en/thread-pool-in-mariadb/>

### Per-thread memory (allocated lazily per query)

| Variable | Default | Role | Multiplies by |
|----------|---------|------|---------------|
| `sort_buffer_size` | 2 MB | Sort buffer for filesort | `max_connections` |
| `read_buffer_size` | 128 KB | Sequential scan buffer | `max_connections` |
| `read_rnd_buffer_size` | 256 KB | Random-read buffer after sort | `max_connections` |
| `join_buffer_size` | 256 KB | Block-nested-loop join buffer | `max_connections` |
| `tmp_table_size` | 16 MB | Max in-memory temp table before disk spill | `max_connections` |
| `max_heap_table_size` | 16 MB | Cap on MEMORY-engine tables and on-disk-spill threshold | Per connection |
| `binlog_cache_size` | 32 KB | Per-transaction binlog buffer | `max_connections` |

## 2. Query Execution Pipeline : Internal Functions

| Stage | Code location | Entry function | Note |
|-------|---------------|----------------|------|
| Receive | `sql_parse.cc` | `dispatch_command()` | One per network command |
| Parse | `sql_yacc.yy`, `sql_lex.cc` | `parse_sql()` | Builds LEX |
| Open tables | `sql_base.cc` | `open_and_lock_tables()` | Acquires metadata locks |
| Optimize | `sql_select.cc` | `JOIN::optimize()` | Cost model, optimizer_switch flags applied |
| Execute | `sql_select.cc` | `JOIN::exec()` | Walks plan, calls handlers |
| Storage-engine access | `sql/handler.h`, engine .cc files | `ha_<op>` family | See handler API below |
| Result | `protocol.cc` | `Protocol::send_*()` | Streams rows to client |

Reference for source layout : <https://github.com/MariaDB/server/tree/main/sql>

## 3. Storage-Engine Handler API (`sql/handler.h`)

The plug-in contract every storage engine implements.

### Table-level operations

| Function | Purpose |
|----------|---------|
| `create()` | Build table on disk (called by CREATE TABLE) |
| `open()` | Open existing table for access |
| `close()` | Release table handle |
| `delete_table()` | Remove table (DROP TABLE) |
| `rename_table()` | Rename (RENAME TABLE) |
| `info()` | Report row count, index stats to the optimizer |

### Row-level operations

| Function | Purpose |
|----------|---------|
| `write_row(uchar* buf)` | Insert a single row |
| `update_row(const uchar* old, const uchar* new)` | Update by primary key |
| `delete_row(const uchar* buf)` | Delete by primary key |
| `rnd_init(bool scan)` | Begin full table scan |
| `rnd_next(uchar* buf)` | Fetch next row in scan |
| `rnd_pos(uchar* buf, uchar* pos)` | Position to specific row id |
| `index_init(uint idx, bool sorted)` | Begin index scan |
| `index_read(uchar* buf, const uchar* key, key_part_map keypart_map, ha_rkey_function find_flag)` | Position to key |
| `index_next(uchar* buf)` | Next row by index |
| `index_first(uchar* buf)` | First row by index |
| `index_last(uchar* buf)` | Last row by index |

### Transaction operations (transactional engines)

| Function | Purpose |
|----------|---------|
| `external_lock(THD*, int lock_type)` | Acquire/release engine-level lock |
| `start_stmt(THD*, thr_lock_type)` | Begin statement context |
| `commit()` | Commit transaction (via `ha_commit_trans`) |
| `rollback()` | Rollback transaction |
| `savepoint_set()` / `savepoint_rollback()` | Savepoint support |

Source : <https://github.com/MariaDB/server/blob/main/sql/handler.h>

## 4. Optimizer Switch : Complete Flag Table

Source : <https://mariadb.com/kb/en/optimizer-switch/>

| Flag | Default | Introduced | Note |
|------|---------|------------|------|
| `condition_pushdown_for_derived` | ON | 10.2.2+ | Push WHERE into derived tables |
| `condition_pushdown_for_subquery` | ON | 10.4+ | Push WHERE into IN-subqueries |
| `condition_pushdown_from_having` | ON | 10.4.3+ | Push HAVING into WHERE where safe |
| `cset_narrowing` | OFF or ON per version | 10.6.16+ | Character-set narrowing |
| `derived_merge` | ON | 5.3+ | Merge derived table into outer query |
| `derived_with_keys` | ON | 5.3+ | Add keys to materialized derived |
| `duplicateweedout` | ON | 12.0+ | Semi-join duplicate weed-out |
| `engine_condition_pushdown` | OFF | 5.5+ | Deprecated 10.1, removed 11.3 |
| `exists_to_in` | ON | 10.0+ | EXISTS to IN transform |
| `extended_keys` | ON | 5.5.21+ | Treat PK columns as part of secondary index |
| `firstmatch` | ON | 5.3+ | Semi-join strategy |
| `hash_join_cardinality` | Version-dependent | 10.6.13+ | Hash-join cost model |
| `index_condition_pushdown` | ON | 5.3+ | Push index condition to engine |
| `index_merge` | ON | 5.1+ | Combine multiple indexes |
| `index_merge_intersection` | ON | 5.1+ | Intersect index-merge |
| `index_merge_sort_intersection` | OFF | 5.3+ | Sorted intersection variant |
| `index_merge_sort_union` | ON | 5.1+ | Sorted union variant |
| `index_merge_union` | ON | 5.1+ | Union variant |
| `in_to_exists` | ON | 5.3+ | IN to EXISTS transform |
| `join_cache_bka` | ON | 5.3+ | Batched key access |
| `join_cache_hashed` | ON | 5.3+ | Hashed join cache |
| `join_cache_incremental` | ON | 5.3+ | Incremental join cache |
| `loosescan` | ON | 5.3+ | Semi-join strategy |
| `materialization` | ON | 5.3+ | Subquery materialization |
| `mrr` | OFF | 5.3+ | Multi-range read |
| `mrr_cost_based` | OFF | 5.3+ | Cost-based MRR |
| `mrr_sort_keys` | OFF | 5.3+ | Sort keys in MRR |
| `not_null_range_scan` | OFF | 10.5+ | NOT NULL range optimization |
| `optimize_join_buffer_size` | ON | 5.3+ | Dynamic join buffer |
| `orderby_uses_equalities` | ON | 10.1.15+ | ORDER BY uses equalities |
| `outer_join_with_cache` | ON | 5.3+ | Cache outer join results |
| `partial_match_rowid_merge` | ON | 5.3+ | Non-semi-join optimization |
| `partial_match_table_scan` | ON | 5.3+ | Table scan partial match |
| `reorder_outer_joins` | Version-dependent | 12.3+ | Outer-join reordering |
| `rowid_filter` | ON | 10.4.3+ | Filter rows by rowid before fetch |
| `sargable_casefold` | ON | 11.3.0+ | Case-folding sargability |
| `semijoin` | ON | 5.3+ | Semi-join optimization |
| `semijoin_with_cache` | ON | 5.3+ | Cached semi-join |
| `split_materialized` | ON | 10.3.4+ | Lateral derived |
| `subquery_cache` | ON | 5.3+ | Cache subquery results |
| `table_elimination` | ON | 5.1+ | Drop unreferenced LEFT JOIN tables |

## 5. Memory Structures (server-level)

| Struct | Defined in | Role |
|--------|-----------|------|
| `THD` | `sql/sql_class.h` | One per connection, holds session state, transaction context, errors, user variables |
| `LEX` | `sql/sql_lex.h` | One per parsed statement, lexer state, parse tree |
| `TABLE_SHARE` | `sql/table.h` | Per-table-definition cached metadata, shared across all `TABLE` instances |
| `TABLE` | `sql/table.h` | Per-thread runtime handle for one table, references a `TABLE_SHARE` |
| `JOIN` | `sql/sql_select.h` | Plan and execution state for one SELECT |
| `Field` | `sql/field.h` | Column representation, type, length, current value |
| `Item` | `sql/item.h` | Expression tree node (function call, constant, column ref) |

## 6. Status Variables for Architecture Introspection

| Variable | Source | Meaning |
|----------|--------|---------|
| `Threads_connected` | `SHOW STATUS` | Currently open connections |
| `Threads_running` | `SHOW STATUS` | Threads not idle (actively processing) |
| `Threads_created` | `SHOW STATUS` | Total threads spawned since start ; rising rapidly means thread-cache too small |
| `Innodb_buffer_pool_reads` | `SHOW STATUS` | Physical disk reads into buffer pool |
| `Innodb_buffer_pool_read_requests` | `SHOW STATUS` | Total logical read requests |
| `Innodb_buffer_pool_wait_free` | `SHOW STATUS` | Waits for a free page ; non-zero growth = pool too small |
| `Innodb_buffer_pool_pages_dirty` | `SHOW STATUS` | Pages waiting to be flushed |
| `Innodb_checkpoint_age` | `SHOW STATUS` (reintroduced 10.5+) | Distance between LSN and last checkpoint |
| `Innodb_log_waits` | `SHOW STATUS` | Waits for the log buffer ; raise `innodb_log_buffer_size` if non-zero |
| `Qcache_hits`, `Qcache_inserts` | `SHOW STATUS` | Query cache effectiveness ; both meaningless if cache OFF |
| `Created_tmp_disk_tables` | `SHOW STATUS` | Tmp tables that spilled to disk ; raise `tmp_table_size` if growing |

Source : <https://mariadb.com/kb/en/server-status-variables/>

## 7. Performance Schema Tables for Architecture

| Table | Use |
|-------|-----|
| `performance_schema.threads` | One row per server thread, with type, state, last statement |
| `performance_schema.events_statements_current` | Last statement per thread, with parsing + execution timings |
| `performance_schema.events_waits_current` | Active wait events (locks, IO, mutexes) |
| `performance_schema.file_summary_by_event_name` | Aggregated IO per file event class |
| `information_schema.OPTIMIZER_TRACE` | Optimizer trace when `optimizer_trace=enabled=on` |
| `information_schema.INNODB_BUFFER_POOL_STATS` | Buffer pool metrics broken down per instance |

## 8. Authentication Plug-Ins (Connection Lifecycle)

| Plug-in | Algorithm | Default in | Note |
|---------|-----------|-----------|------|
| `mysql_native_password` | SHA-1 | Legacy default | Deprecated, weak hash |
| `ed25519` | Edwards-curve DSA | Recommended from 10.1.21+ | Same algorithm as OpenSSH |
| `unix_socket` | OS uid mapping | Debian/Ubuntu root by default | No password needed |
| `gssapi` | Kerberos | Optional | Enterprise SSO |
| `pam` | OS PAM stack | Optional | LDAP, RADIUS via PAM modules |

Source : <https://mariadb.com/kb/en/authentication-plugin-ed25519/>
