# Examples : MariaDB Performance Tuning

Ten worked configurations and monitoring runbooks. Every snippet annotated with the version it applies to. Verified against `mariadb.com/kb/en/innodb-system-variables/`, `mariadb.com/kb/en/thread-pool-in-mariadb/`, and `mariadb.com/kb/en/query-cache/`.

## Example 1 : 64 GB RAM Dedicated OLTP Host

Web application with ACID requirements, SATA SSD, ~200 active connections.

```ini
# /etc/mysql/mariadb.conf.d/50-server.cnf (MariaDB 10.6+ / 10.11+ / 11.x)
[mariadb]
# Buffer pool : 75% of RAM on a dedicated host
innodb_buffer_pool_size            = 48G
# NOTE : innodb_buffer_pool_instances REMOVED in 10.6.0+, do NOT set.
# NOTE : innodb_buffer_pool_chunk_size deprecated 10.11.12+ / 11.4.6+ / 11.8.2+.

# Redo log : ~16% of buffer-pool, fast recovery on SSD
innodb_log_file_size               = 8G
innodb_log_buffer_size             = 64M

# Durability : full ACID
innodb_flush_log_at_trx_commit     = 1
sync_binlog                        = 1
innodb_doublewrite                 = ON

# IO capacity : SATA SSD
innodb_io_capacity                 = 2000
innodb_io_capacity_max             = 4000

# Concurrency
innodb_thread_concurrency          = 0          # unlimited (recommended)

# Connections
max_connections                    = 300
table_open_cache                   = 4000
table_definition_cache             = 2000

# Per-connection buffers : conservative
sort_buffer_size                   = 2M
read_buffer_size                   = 128K
read_rnd_buffer_size               = 512K
join_buffer_size                   = 512K

# Query cache : OFF (default since 10.1.7)
query_cache_type                   = OFF
query_cache_size                   = 0
```

Per-connection memory budget : `(2M + 128K + 512K + 512K) * 300 conn ≈ 940 MB` -> well within RAM after 48 GB pool.

## Example 2 : 256 GB Analytics Host

Read-heavy reporting, NVMe storage, low connection count, large in-flight scans.

```ini
# 256 GB RAM analytics host, MariaDB 10.11+ / 11.x
[mariadb]
innodb_buffer_pool_size            = 192G                  # 75% of RAM
innodb_log_file_size               = 16G
innodb_log_buffer_size             = 128M
innodb_flush_log_at_trx_commit     = 1                      # durable
sync_binlog                        = 1

# NVMe IO
innodb_io_capacity                 = 10000
innodb_io_capacity_max             = 20000

# Few connections, large sorts
max_connections                    = 100
sort_buffer_size                   = 16M
read_buffer_size                   = 2M
read_rnd_buffer_size               = 16M
join_buffer_size                   = 16M
tmp_table_size                     = 256M
max_heap_table_size                = 256M

# Caches
table_open_cache                   = 8000
table_definition_cache             = 4000

# Stats : engine-independent for richer plans
use_stat_tables                    = PREFERABLY
```

Per-connection memory : `(16M + 2M + 16M + 16M) * 100 conn = 5 GB` -> safe alongside the 192 GB pool.

## Example 3 : Bulk-Load Tuning Profile

ETL or initial migration job. Relax durability for the duration of the load, then restore.

```sql
-- 10.6+
-- BEFORE the load
SET GLOBAL innodb_flush_log_at_trx_commit = 0;
SET GLOBAL sync_binlog                    = 0;
SET GLOBAL innodb_doublewrite             = OFF;   -- only on atomic-write storage
SET GLOBAL innodb_io_capacity             = 20000; -- saturate the device
SET GLOBAL innodb_io_capacity_max         = 40000;

-- run the load (mariadb-import, LOAD DATA, bench restore, ...)

-- AFTER the load : RESTORE
SET GLOBAL innodb_doublewrite             = ON;
SET GLOBAL sync_binlog                    = 1;
SET GLOBAL innodb_flush_log_at_trx_commit = 1;
SET GLOBAL innodb_io_capacity             = 2000;  -- production baseline
SET GLOBAL innodb_io_capacity_max         = 4000;
```

NEVER leave the relaxed values in production. Wrap the load in a deployment runbook so the restore step cannot be skipped.

## Example 4 : Thread Pool for High-Connection OLTP

Application with 2000 concurrent connections, mostly short queries.

```ini
# Unix, MariaDB 10.6+
[mariadb]
thread_handling                    = pool-of-threads
thread_pool_size                   = 16            # = CPU cores (default)
thread_pool_max_threads            = 1000          # cap below 65536 default
thread_pool_oversubscribe          = 3             # KB default
thread_pool_idle_timeout           = 60
thread_pool_stall_limit            = 500           # ms

# Connections : the pool serializes ; max_connections can be high
max_connections                    = 2000

# Per-connection buffers MUST stay conservative
sort_buffer_size                   = 1M
read_buffer_size                   = 128K
read_rnd_buffer_size               = 256K
join_buffer_size                   = 256K
```

Pool serializes work onto `thread_pool_size` workers. Maximum active OS threads ≈ `thread_pool_size * thread_pool_oversubscribe`.

NEVER raise `thread_pool_size` to thousands ; KB explicitly warns this is "not a good idea".

## Example 5 : Monitoring Buffer-Pool Hit Ratio

```sql
-- 10.6+ : single-shot hit ratio (after 24 h warm-up)
SELECT
   ROUND(100 * (1 -
     (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS
       WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') /
     (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS
       WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests')
   ), 4) AS hit_ratio_pct;
-- Healthy : > 99.0
-- Warning : 95.0 - 99.0 (working set exceeds pool, or pool undersized)
-- Critical : < 95.0
```

## Example 6 : Reading SHOW ENGINE INNODB STATUS

```sql
-- 10.6+
SHOW ENGINE INNODB STATUS\G
```

Sections to read first :

- `BUFFER POOL AND MEMORY` : `Buffer pool hit rate XXXX / 1000` (close to 1000/1000 is healthy).
- `LATEST DETECTED DEADLOCK` : presence indicates application transaction logic needs review (NOT a tuning fix).
- `TRANSACTIONS` : long-running transactions hold undo and prevent purge. Watch for transaction-id gap.
- `LOG` : `Log sequence number` vs `Last checkpoint at` gap. Large gap means redo log undersized OR write storm.
- `ROW OPERATIONS` : per-second insert/update/delete rates.

NEVER tune off a single sample. Run twice, 60 s apart, compare deltas.

## Example 7 : Per-Connection Memory Calculation

```sql
-- 10.6+ : approximate per-connection memory ceiling
SELECT
  @@sort_buffer_size      AS sort_buf,
  @@read_buffer_size      AS read_buf,
  @@read_rnd_buffer_size  AS read_rnd_buf,
  @@join_buffer_size      AS join_buf,
  @@thread_stack          AS stack,
  (@@sort_buffer_size + @@read_buffer_size + @@read_rnd_buffer_size
    + @@join_buffer_size + @@thread_stack) AS per_conn_bytes,
  @@max_connections                              AS max_conn,
  (@@sort_buffer_size + @@read_buffer_size + @@read_rnd_buffer_size
    + @@join_buffer_size + @@thread_stack) * @@max_connections / (1024*1024*1024)
                                                  AS total_gb_if_all_full;
```

Compare `total_gb_if_all_full + innodb_buffer_pool_size_gb + 2 GB OS headroom` against physical RAM. Adjust either `sort_buffer_size` family OR `max_connections` until it fits.

## Example 8 : Session-Scoped Sort Buffer for One Heavy Report

When a single nightly report needs a large sort, keep the global default small and bump per-session.

```sql
-- 10.6+
SET SESSION sort_buffer_size       = 64 * 1024 * 1024;   -- 64 MB
SET SESSION tmp_table_size         = 256 * 1024 * 1024;
SET SESSION max_heap_table_size    = 256 * 1024 * 1024;

SELECT customer_id, COUNT(*) AS orders, SUM(total) AS revenue
  FROM orders
 WHERE created_at BETWEEN '2026-01-01' AND '2026-04-30'
 GROUP BY customer_id
 ORDER BY revenue DESC;

-- Session ends, defaults restored. No global impact, no OOM risk on small queries.
```

## Example 9 : Performance Schema Top Wait Events

```sql
-- 10.6+ : enable IO instruments
UPDATE performance_schema.setup_instruments
   SET ENABLED = 'YES', TIMED = 'YES'
 WHERE NAME LIKE 'wait/io/file%';

-- Top 10 wait events by total time across the entire server uptime
SELECT EVENT_NAME,
       COUNT_STAR,
       ROUND(SUM_TIMER_WAIT / 1e12, 2) AS total_seconds,
       ROUND(AVG_TIMER_WAIT / 1e6, 2)  AS avg_microseconds
  FROM performance_schema.events_waits_summary_global_by_event_name
 ORDER BY SUM_TIMER_WAIT DESC
 LIMIT 10;
```

`wait/io/file/innodb/innodb_data_file` dominating = storage-bound. `wait/synch/mutex/sql/LOCK_open` dominating = `table_open_cache` too small.

## Example 10 : Read-Heavy Static Workload (the ONE Place Query Cache Helps)

A documentation site with 99% reads against a 1 GB dataset that changes once per day.

```ini
# MariaDB 10.6+
[mariadb]
query_cache_type = ON
query_cache_size = 256M
query_cache_limit = 2M

# everything else still applies : buffer pool, log file, etc.
```

After 24 h, verify the cache is actually helping :

```sql
SHOW GLOBAL STATUS LIKE 'Qcache%';
-- Qcache_hits           : should be high relative to Com_select
-- Qcache_inserts        : should be MUCH lower than hits
-- Qcache_lowmem_prunes  : 0 ideally ; > 0 means cache is undersized

SHOW GLOBAL STATUS LIKE 'Com_select';
-- Compute hit ratio : Qcache_hits / (Qcache_hits + Com_select)
```

If the ratio is below ~60% : the cache is not paying for itself, turn it OFF.

NEVER turn the query cache ON on a write-heavy workload. KB explicitly warns "It does not scale well in environments with high throughput on multi-core machines."
