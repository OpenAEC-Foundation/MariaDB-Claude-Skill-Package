# Methods : MariaDB Performance-Tuning System Variables

Complete reference of every system variable used in `mariadb-impl-performance-tuning`. All defaults, ranges, scope, dynamism, and version flags verified against `mariadb.com/kb/en/innodb-system-variables/`, `mariadb.com/docs/server/server-management/variables-and-modes/server-system-variables.md`, `mariadb.com/kb/en/thread-pool-in-mariadb/`, and `mariadb.com/kb/en/query-cache/` on 2026-05-19.

## InnoDB Buffer Pool

### innodb_buffer_pool_size

- **Scope** : Global, Dynamic
- **Default** : `134217728` (128 MiB)
- **Range** : Minimum 2 MiB (4k pages) up to 20 MiB (64k pages) ; maximum 9223372036854775807 (8 EiB)
- **Recommendation** : "can be set up to 80% of the total memory in these environments" (dedicated DB host)
- **Versions** : All targeted versions (10.6+)

### innodb_buffer_pool_instances

- **Status** : **REMOVED in MariaDB 10.6.0**. Deprecated and ignored from 10.5.1.
- **Note** : Buffer pool runs as a single instance regardless of size. Do NOT set this variable in 10.6+ my.cnf.
- **Versions** : Not applicable to any targeted version.

### innodb_buffer_pool_chunk_size

- **Status** : **Deprecated and ignored from MariaDB 10.11.12, 11.4.6, 11.8.2**.
- **Default (pre-10.8.0)** : 134217728 (128 MiB). **Default (10.8.0+)** : autosize (0), buffer_pool_size / 64.
- **Versions** : Still functional in 10.6.x, 10.11.0-11. Ignored from 10.11.12, 11.4.6, 11.8.2.

### innodb_buffer_pool_size_max

- **Status** : **Introduced in MariaDB 10.11.12, 11.4.6, 11.8.2** (replaces chunk-based resize).
- **Scope** : Global, Read-only (startup-only)
- **Default** : Initial `innodb_buffer_pool_size` rounded up to 8 MB.
- **Versions** : 10.11.12+, 11.4.6+, 11.8.2+, 12.x.

## InnoDB Log

### innodb_log_file_size

- **Scope** : Global, Read-only (startup)
- **Default** : 100663296 (96 MiB) in older versions ; varies by version.
- **Recommendation** : Start at ~25% of `innodb_buffer_pool_size`. Cap at recovery-time tolerance.

### innodb_log_buffer_size

- **Scope** : Global, Read-only
- **Default** : 16 MiB
- **Range** : 2 MiB to 4 GiB
- **Recommendation** : Raise to 64-128 MiB on write-heavy workloads.

### innodb_flush_log_at_trx_commit

- **Scope** : Global, Dynamic
- **Default** : `1`
- **Values** :
  - `1` : flush redo log to disk on every commit (full ACID).
  - `2` : write to OS cache on every commit, flush per `innodb_flush_log_at_timeout` (default 1 s).
  - `0` : write and flush once per second only.
  - `3` : MariaDB 5.5 group-commit emulation. Legacy only.

### innodb_flush_method

- **Status** : Default `O_DIRECT` on Unix from MariaDB 10.6. Previous default `fsync`. **Deprecated from MariaDB 11.0** ; replaced by four boolean dynamic variables.
- **Valid Unix values** : `fsync`, `O_DSYNC`, `O_DIRECT`, `O_DIRECT_NO_FSYNC`, `littlesync`, `nosync`.
- **Valid Windows values** : `unbuffered`, `async_unbuffered`, `normal`.

## InnoDB IO

### innodb_io_capacity

- **Scope** : Global, Dynamic
- **Default** : `200`
- **Range** : 100 to 18446744073709551615
- **Purpose** : "The number of I/O operations per second (IOPS) InnoDB is assumed to be able to do."

### innodb_io_capacity_max

- **Scope** : Global, Dynamic
- **Default** : `2000`
- **Range** : 100 to 18446744073709551615
- **Recommendation** : Roughly 2x `innodb_io_capacity`.

### innodb_doublewrite

- **Scope** : Global, Dynamic
- **Default** : `ON` (default `ON` again from 11.0.6)
- **Values** : `OFF`, `ON`, `fast` (10.6.0+, default again 11.0.6+)
- **Purpose** : Doublewrite buffer protects against torn-page writes.

## InnoDB Fill Factor

### innodb_fill_factor

- **Scope** : Global, Dynamic
- **Default** : `100`
- **Range** : 10 to 100
- **Purpose** : "Percentage of B-tree page filled during bulk insert. Setting to 70 reserves 30% of space on each page for index growth."

### innodb_thread_concurrency

- **Scope** : Global, Dynamic
- **Default** : `0` (unlimited)
- **Recommendation** : Leave at 0 unless KB-documented evidence shows benefit on the specific workload.

## Server : Per-Connection Buffers

These are **per-session** allocations. They multiply by `max_connections`.

### sort_buffer_size

- **Scope** : Global, Session, Dynamic
- **Default** : 2 MiB (2097152 bytes)
- **Allocation** : per session, on demand.

### read_buffer_size

- **Scope** : Global, Session, Dynamic
- **Default** : 128 KiB (131072 bytes)
- **Allocation** : per session, on demand.

### read_rnd_buffer_size

- **Scope** : Global, Session, Dynamic
- **Default** : 256 KiB (262144 bytes)
- **Allocation** : per session, on demand.

### join_buffer_size

- **Scope** : Global, Session, Dynamic
- **Default** : 256 KiB (262144 bytes)
- **Allocation** : per session, on demand (when no index can be used for the join).

### thread_stack

- **Scope** : Global, Read-only (startup)
- **Default** : ~292 KiB (varies by build, 299008 on common builds)
- **Allocation** : per thread.

## Server : Caches

### table_open_cache

- **Scope** : Global, Dynamic
- **Default** : 2000

### table_definition_cache

- **Scope** : Global, Dynamic
- **Default** : 400

### tmp_table_size

- **Scope** : Global, Session, Dynamic
- **Default** : 16 MiB (16777216 bytes)

### max_heap_table_size

- **Scope** : Global, Session, Dynamic
- **Default** : 16 MiB (16777216 bytes)

## Server : Connections and Binlog

### max_connections

- **Scope** : Global, Dynamic
- **Default** : 151

### sync_binlog

- **Scope** : Global, Dynamic
- **Default** : 0
- **Values** : `0` = OS-flush cadence ; `1` = fsync on every binlog write. Use `1` for crash-safe binlog replay.

## Thread Pool (Unix)

### thread_handling

- **Scope** : Global, Read-only (startup)
- **Default** : `one-thread-per-connection` (Unix). `pool-of-threads` is opt-in on Unix, default on Windows.
- **Values** : `one-thread-per-connection`, `pool-of-threads`, `no-threads` (DEBUG builds only).

### thread_pool_size

- **Scope** : Global, Dynamic
- **Default** : Number of CPUs on the system.
- **KB warning** : "the max value is 100000. However, it is not a good idea to set it that high."

### thread_pool_max_threads

- **Scope** : Global, Dynamic
- **Default** : 65536
- **Purpose** : "Once this limit is reached, no new threads will be created in most cases."

### thread_pool_oversubscribe

- **Scope** : Global, Dynamic
- **Default** : 3
- **Note** : KB describes as "primarily for internal use."

### thread_pool_stall_limit

- **Scope** : Global, Dynamic
- **Default** : 500 ms
- **Purpose** : Interval between stall checks by timer thread.

### thread_pool_idle_timeout

- **Scope** : Global, Dynamic
- **Default** : 60 seconds
- **Purpose** : Duration before idle worker thread exits.

### thread_pool_prio_kickup_timer

- **Purpose** : Timeout after which low-priority connections move to the high-priority queue.

## Thread Pool (Windows)

### thread_pool_min_threads

- **Default** : 1

### thread_pool_max_threads (Windows)

- **Default** : 1000

## Query Cache

### query_cache_type

- **Scope** : Global, Session, Dynamic
- **Default** : `OFF` (since MariaDB 10.1.7)
- **Status in 11.x** : Still present, NOT removed. Default remains `OFF`.

### query_cache_size

- **Scope** : Global, Dynamic
- **Default** : 1 MiB
- **Status in 11.x** : Still present.

### query_cache_limit

- **Scope** : Global, Dynamic
- **Default** : 1 MiB
- **Purpose** : Maximum result size to cache.

### query_cache_min_res_unit

- **Scope** : Global, Dynamic
- **Default** : 4 KiB
- **Purpose** : Smallest allocation block in the cache.

## Monitoring Queries

```sql
-- 10.6+ : buffer-pool hit ratio (after warm-up)
SELECT
   ROUND(100 * (1 -
     (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS
       WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') /
     (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS
       WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests')
   ), 4) AS hit_ratio_pct;
-- Target > 99.0

-- 10.6+ : opened-tables growth (run twice, 60 s apart, subtract)
SHOW GLOBAL STATUS LIKE 'Opened_tables';

-- 10.6+ : thread-pool group activity (when pool-of-threads)
SHOW STATUS LIKE 'Threadpool%';

-- 10.6+ : full InnoDB engine status
SHOW ENGINE INNODB STATUS\G
```

## Performance Schema (10.6+)

```sql
-- Enable specific instruments
UPDATE performance_schema.setup_instruments
   SET ENABLED = 'YES', TIMED = 'YES'
 WHERE NAME LIKE 'wait/io/file%';

-- Top 10 wait events by total time
SELECT EVENT_NAME, COUNT_STAR, SUM_TIMER_WAIT/1e12 AS total_seconds
  FROM performance_schema.events_waits_summary_global_by_event_name
 ORDER BY SUM_TIMER_WAIT DESC
 LIMIT 10;
```

## Deprecation Matrix (Quick Lookup)

| Variable | Status | Versions |
|----------|--------|----------|
| `innodb_buffer_pool_instances` | REMOVED | 10.6.0+ |
| `innodb_buffer_pool_chunk_size` | Deprecated, ignored | 10.11.12+, 11.4.6+, 11.8.2+ |
| `innodb_flush_method` | Deprecated | 11.0+ |
| `query_cache_type`, `query_cache_size` | Default OFF | Since 10.1.7 (NOT removed in 11.x) |
| `innodb_buffer_pool_size_max` | Introduced | 10.11.12+, 11.4.6+, 11.8.2+ |
| `innodb_doublewrite=fast` | Added | 10.6.0+, default ON again 11.0.6+ |
