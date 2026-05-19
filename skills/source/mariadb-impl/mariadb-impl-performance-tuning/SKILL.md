---
name: mariadb-impl-performance-tuning
description: >
  Use when tuning a new MariaDB instance, diagnosing slow server performance, sizing innodb_buffer_pool, choosing between one-thread-per-connection and the thread pool, deciding on the durability vs throughput trade-off, or migrating tuning settings from MySQL to MariaDB.
  Prevents the common mistake of setting innodb_buffer_pool_size to 50% of RAM (too low for a dedicated DB host), disabling innodb_flush_log_at_trx_commit on financial data, copying a MySQL 8 my.cnf verbatim (different defaults and removed variables), and inflating per-connection buffers without multiplying by max_connections (OOM).
  Covers innodb_buffer_pool_size (60-80% RAM rule), innodb_buffer_pool_instances (removed 10.6.0+), innodb_buffer_pool_chunk_size (deprecated 10.11.12+ / 11.4.6+ / 11.8.2+), innodb_buffer_pool_size_max, innodb_log_file_size, innodb_log_buffer_size, innodb_flush_log_at_trx_commit (0/1/2/3 modes), innodb_flush_method (deprecated 11.0+), innodb_io_capacity for SSD vs HDD vs NVMe, innodb_doublewrite (fast 11.0.6+), thread_pool, table_open_cache, per-connection buffers (sort, read_rnd, join, thread_stack), sync_binlog, query cache history (default OFF since 10.1.7+, still present but disabled in 11.x), innodb_fill_factor.
  Keywords: performance tuning, my mariadb is slow, slow server, mariadb performance, innodb_buffer_pool_size, buffer pool sizing, 60 percent RAM, 80 percent RAM, innodb_log_file_size, innodb_flush_log_at_trx_commit, ACID durability, innodb_io_capacity, SSD vs HDD, NVMe, innodb_doublewrite, thread_pool, pool of threads, table_open_cache, sort_buffer_size, read_rnd_buffer_size, join_buffer_size, thread_stack, per-connection buffer, max_connections OOM, sync_binlog, query cache deprecated, query_cache_type OFF, innodb_buffer_pool_chunk_size, innodb_buffer_pool_size_max, innodb_fill_factor, my.cnf production, my.cnf OLTP, my.cnf analytics, how to tune mariadb, server is swapping, server is slow, copying mysql 8 my.cnf
license: MIT
compatibility: "Designed for Claude Code. Requires MariaDB 10.6-LTS, 10.11-LTS, 11.x, 12.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# MariaDB Performance Tuning

How to size the InnoDB buffer pool, choose a durability mode, scale IO capacity to the underlying storage, decide between one-thread-per-connection and the thread pool, and avoid the per-connection-buffer multiplication trap. Use this skill on any "the database is slow" or "production my.cnf" task, before changing `innodb_flush_log_at_trx_commit`, and before copying a MySQL 8 config to MariaDB.

## Quick Reference

- `innodb_buffer_pool_size` : set to 60-80% of RAM on a dedicated DB host. KB caps the recommendation at 80% and warns "not too large, because this can cause swapping, which more than undoes the benefits".
- `innodb_flush_log_at_trx_commit=1` is the only full-ACID setting (default). `2` survives process crash but loses up to one second on OS crash. `0` loses up to one second on any crash. `3` exists for legacy 5.5 group-commit emulation.
- `innodb_buffer_pool_instances` is **REMOVED from MariaDB 10.6.0**. NEVER write it into a 10.6+ my.cnf : the server logs an unknown-variable warning. Buffer pool runs as a single instance.
- `innodb_buffer_pool_chunk_size` is **deprecated and ignored from 10.11.12, 11.4.6, 11.8.2**. From those versions on, buffer-pool resize is contiguous in 1 MB steps up to `innodb_buffer_pool_size_max`.
- `innodb_io_capacity=200` (default) targets HDD. SSD baseline is 2000, NVMe 5000-10000. `innodb_io_capacity_max` defaults to 2000 ; set it to ~2x `innodb_io_capacity`. Leaving 200 on NVMe wastes ~95% of the device.
- `innodb_flush_method=O_DIRECT` (default on Unix since 10.6) avoids OS-cache double-buffering on the data files. The variable itself is **deprecated from 11.0** and replaced by four boolean dynamic variables.
- Query cache : `query_cache_type=OFF` is the default since **10.1.7** (KB confirms "disabled by default"). Variable is **NOT removed in 11.x** ; it is still present and tunable, just OFF. NEVER turn it ON on a write-heavy workload : try_lock contention serializes invalidations.
- Per-connection buffers MULTIPLY by `max_connections` : `sort_buffer_size` (default 2 MiB), `read_buffer_size` (128 KiB), `read_rnd_buffer_size` (256 KiB), `join_buffer_size` (256 KiB), `thread_stack` (~292 KiB). Setting `sort_buffer_size=256M` with `max_connections=1000` reserves up to 256 GB of address space.
- Thread pool : `thread_handling=pool-of-threads`. Default OFF on Unix, ON on Windows. `thread_pool_size` defaults to CPU-core count. KB warns "the max value is 100000. However, it is not a good idea to set it that high."
- `sync_binlog=1` is durable but slow. `sync_binlog=0` defers to OS flush cadence ; binlog can lag the data file on crash. `1` is the only correct value for financial systems with binlog replication.
- ALWAYS scale `innodb_log_file_size` up on write-heavy systems. A common starting point is ~25% of buffer-pool size, capped by recovery-time tolerance (larger log = slower crash recovery).

## Decision Tree : "Tune a fresh production instance"

1. **Is the host dedicated to MariaDB?** If yes : `innodb_buffer_pool_size` = 60-80% of RAM. If shared (app + DB on same box) : start at 25-50% and benchmark.
2. **What storage?** HDD : `innodb_io_capacity=200`, `innodb_io_capacity_max=400`. SATA SSD : `2000` / `4000`. NVMe : `5000-10000` / `10000-20000`.
3. **Durability tolerance?** Financial / regulated : `innodb_flush_log_at_trx_commit=1` + `sync_binlog=1`. Tolerate up to 1 s loss on OS crash : `=2` + `sync_binlog=0`. Bulk-load only : `=0` temporarily.
4. **Connection model?** Mostly long-lived app connections, low count (<200) : one-thread-per-connection (default). Many short-lived connections (>500) and CPU-bound OLTP : `thread_handling=pool-of-threads`.
5. **Per-connection buffer review** : confirm `sort_buffer_size * max_connections < total_RAM - innodb_buffer_pool_size - 2 GB headroom`. Same check for `read_rnd_buffer_size`, `join_buffer_size`.
6. **Log file sizing** : `innodb_log_file_size` ≈ 25% of buffer-pool for write-heavy ; smaller (256 MB) for read-mostly. Larger value = fewer log flushes but longer crash recovery.
7. **Statistics** : run `ANALYZE TABLE` on the heavy-hit tables after the dataset is loaded. Tuning a server with stale stats produces bad benchmarks. See `mariadb-impl-query-optimization`.
8. **Validate** : check `SHOW ENGINE INNODB STATUS\G` and `SHOW STATUS LIKE 'Innodb_buffer_pool%'` for the hit ratio after 24 h of production load.

## Sizing the Buffer Pool

The buffer pool dominates InnoDB performance because every page read either comes from it (memory) or from disk (orders of magnitude slower). KB rule on a dedicated DB host : "can be set up to 80% of the total memory in these environments".

```ini
# /etc/mysql/mariadb.conf.d/50-server.cnf
# 64 GB RAM dedicated MariaDB host, 10.6+
[mariadb]
innodb_buffer_pool_size = 48G          # 75% of RAM
# innodb_buffer_pool_instances = 8     # REMOVED 10.6.0+ ; do NOT set
# innodb_buffer_pool_chunk_size = 128M # deprecated 10.11.12+ / 11.4.6+ / 11.8.2+
```

Health check :

```sql
-- 10.6+
SHOW STATUS LIKE 'Innodb_buffer_pool_read%';
-- Innodb_buffer_pool_reads          : pages fetched from disk
-- Innodb_buffer_pool_read_requests  : pages requested
-- Target ratio reads/read_requests < 1% on a warm cache.
```

If the ratio is above 1% after 24 h, the buffer pool is undersized OR the working set is bigger than memory. Bump `innodb_buffer_pool_size` and re-measure, or accept that the working set exceeds RAM and budget faster storage.

**Trap** : on a 64 GB host the historic guideline of "50% of RAM" gives 32 GB. That leaves 32 GB unused. On a dedicated DB box, 75-80% is correct.

## Durability vs Throughput : The `innodb_flush_log_at_trx_commit` Matrix

| Value | Behaviour | Survives | Loses | Use when |
|-------|-----------|----------|-------|----------|
| `1` (default) | Flush redo log to disk on every commit | Process crash, OS crash, power loss | Nothing committed | ACID required, financial, regulated |
| `2` | Write to OS cache on every commit, flush once per second | Process crash | Up to 1 s on OS crash | Tolerate small loss for higher throughput |
| `0` | Write and flush once per second | Nothing extra | Up to 1 s on any crash | Bulk loads, ETL, where data is regenerable |
| `3` | Emulate MariaDB 5.5 group commit | Legacy only | See KB | NEVER on a fresh setup |

```ini
# Strong durability (default, financial)
innodb_flush_log_at_trx_commit = 1
sync_binlog                    = 1

# Speed-leaning, OS-crash-tolerant (web app, can tolerate 1 s loss)
innodb_flush_log_at_trx_commit = 2
sync_binlog                    = 0
```

NEVER pair `innodb_flush_log_at_trx_commit=0` with binlog replication unless you accept that the binlog can outlast the data file on crash. NEVER weaken durability for a perceived speed problem before profiling the actual bottleneck (storage IOPS, CPU, lock contention) with `SHOW ENGINE INNODB STATUS\G`.

## InnoDB Log Sizing

`innodb_log_file_size` controls the redo-log capacity. Larger value = fewer log-buffer flushes and less write amplification, BUT longer crash recovery.

```ini
# Write-heavy OLTP (48 G buffer pool), 10.6+
innodb_log_file_size   = 8G          # ~16% of buffer pool
innodb_log_buffer_size = 64M
```

Rule of thumb : start at 25% of `innodb_buffer_pool_size`, cap at the largest size whose crash recovery completes inside your RTO. On a dedicated NVMe host, 8 GB log files recover in seconds.

## IO Capacity for Modern Storage

`innodb_io_capacity` controls background-flush throughput.

| Storage class | `innodb_io_capacity` | `innodb_io_capacity_max` |
|---------------|----------------------|--------------------------|
| 7200 RPM HDD (default) | 200 | 400 |
| 15000 RPM HDD | 400 | 800 |
| SATA SSD | 2000 | 4000 |
| NVMe SSD | 5000 - 10000 | 10000 - 20000 |
| Cloud burst-IOPS (gp3, Premium SSD) | match provisioned IOPS | 2x |

```ini
# NVMe SSD baseline, 10.6+
innodb_io_capacity     = 8000
innodb_io_capacity_max = 16000
```

Leaving `200` on NVMe wastes ~95% of the device's IO budget : InnoDB throttles itself to HDD-class flush rates.

## Thread Pool : When and How

Default on Unix is `thread_handling=one-thread-per-connection` : one OS thread per client. Cheap up to a few hundred connections, expensive above that (context-switching overhead).

Switch to the pool when : many short-lived connections (>500), CPU-bound OLTP, or memory pressure from thousands of thread stacks.

```ini
# Pool of threads, Unix (must restart, 10.6+)
[mariadb]
thread_handling          = pool-of-threads
thread_pool_size         = 16              # default = CPU count
thread_pool_max_threads  = 65536           # KB default ; rarely raised
thread_pool_oversubscribe = 3              # default ; KB calls it "primarily for internal use"
thread_pool_idle_timeout = 60              # seconds
```

KB warning : `thread_pool_size` "max value is 100000. However, it is not a good idea to set it that high." Set it to physical-core count (not vcpu count) on bare metal, vcpu count on virtualised hosts.

NEVER enable the thread pool on workloads with long-running, non-yielding queries (large analytics SELECT, big LOCK TABLES) ; they pin worker threads and starve the rest.

## Table Cache

```ini
# 10.6+ defaults are usually fine ; raise only when COM_OPEN_TABLES is hot
table_open_cache        = 4000     # default 2000
table_definition_cache  = 2000     # default 400
```

Check `Opened_tables` over time : `SHOW GLOBAL STATUS LIKE 'Opened_tables'`. A monotonically rising value above ~1000/min means cache is too small.

## The Per-Connection-Buffer Trap

These buffers are allocated **per session, on demand**. They multiply by `max_connections`.

| Variable | Default | Safe max (with `max_connections=200`) | OOM example |
|----------|---------|---------------------------------------|-------------|
| `sort_buffer_size`     | 2 MiB    | 16 MiB  | `256M * 1000 conn = 256 GB` |
| `read_buffer_size`     | 128 KiB  | 1 MiB   | `4M * 1000 conn = 4 GB`     |
| `read_rnd_buffer_size` | 256 KiB  | 4 MiB   | `16M * 1000 conn = 16 GB`   |
| `join_buffer_size`     | 256 KiB  | 4 MiB   | `64M * 1000 conn = 64 GB`   |
| `thread_stack`         | ~292 KiB | leave default | rarely raised |

ALWAYS verify : `per_conn_buffers_sum * max_connections + innodb_buffer_pool_size + 2 GB OS headroom < total RAM`.

If a single rare query needs `sort_buffer_size=64M`, raise it for that session only :

```sql
-- 10.6+ : session scope, no global impact
SET SESSION sort_buffer_size = 64 * 1024 * 1024;
SELECT ...;
```

## Query Cache : Default OFF Since 10.1.7

```sql
-- 10.6+
SELECT @@global.query_cache_type, @@global.query_cache_size;
-- Expected : OFF, 1048576
```

KB confirms "It does not scale well in environments with high throughput on multi-core machines, so it is disabled by default." The variable is **still present in 11.x** ; it is NOT removed.

NEVER turn the query cache ON on a write-heavy workload : every write to a cached table invalidates the cache entry under a global mutex. A read-heavy workload (>99% SELECT) with a small, mostly-static dataset is the only place it ever helps.

## Doublewrite Buffer

`innodb_doublewrite=ON` (default) protects against torn-page writes by writing every page twice (once to the doublewrite buffer, once to the data file). The `fast` value (10.6.0+ ; default `ON` again since 11.0.6) skips the sync between the two writes.

```ini
# Default, 10.6+ : safe
innodb_doublewrite = ON
```

NEVER set `innodb_doublewrite=OFF` on production storage that does not guarantee atomic page writes (most consumer SSD, most cloud block storage).

## Bulk-Load Tuning Profile

For one-off bulk loads (initial migration, dimensional rebuild), relax durability temporarily :

```sql
-- 10.6+ : session scope
SET GLOBAL innodb_flush_log_at_trx_commit = 0;
SET GLOBAL sync_binlog                    = 0;
SET GLOBAL innodb_doublewrite             = OFF;   -- only if storage is atomic-write-safe
-- run the load
SET GLOBAL innodb_doublewrite             = ON;
SET GLOBAL sync_binlog                    = 1;
SET GLOBAL innodb_flush_log_at_trx_commit = 1;
```

ALWAYS restore the production values when the load finishes.

## Quality Rules

- ALWAYS set `innodb_buffer_pool_size` to 60-80% of RAM on a dedicated DB host. Defaults (128 MiB) are tuned for tiny instances, not servers.
- ALWAYS keep `innodb_flush_log_at_trx_commit=1` (and `sync_binlog=1`) on systems that own money or legally-relevant records.
- ALWAYS multiply per-connection buffer sizes by `max_connections` before changing them. If the product approaches free RAM, lower one of the two.
- ALWAYS scale `innodb_io_capacity` to the underlying storage. 200 (default) is HDD-class and starves SSD or NVMe.
- ALWAYS read the variable's KB page for deprecation status before adding it to a fresh 10.6+ / 11.x my.cnf. `innodb_buffer_pool_instances` is removed ; `innodb_buffer_pool_chunk_size` is deprecated ; `innodb_flush_method` is deprecated from 11.0.
- NEVER copy a MySQL 8 my.cnf verbatim to MariaDB. Defaults differ ; some MySQL variables (e.g. `innodb_dedicated_server`) do not exist in MariaDB.
- NEVER enable the query cache on a write-heavy workload. Try_lock contention serializes invalidations under a global mutex.
- NEVER weaken durability without an instrumented benchmark proving the current setting is the bottleneck.
- NEVER set `thread_pool_size` to thousands. KB explicitly warns it is "not a good idea".

## Reference Links

- `references/methods.md` : full system-variable reference for InnoDB and thread pool, defaults per version, ranges, monitoring queries, deprecation matrix.
- `references/examples.md` : ten worked configurations (64 GB OLTP, 256 GB analytics, NVMe my.cnf, write-heavy vs read-heavy, bulk-load profile, monitoring queries, thread-pool sizing).
- `references/anti-patterns.md` : eight real-world anti-patterns with code and corrected alternatives.

## Sources

- KB `InnoDB System Variables` : https://mariadb.com/kb/en/innodb-system-variables/
- KB `Server System Variables` : https://mariadb.com/docs/server/server-management/variables-and-modes/server-system-variables.md
- KB `Thread Pool in MariaDB` : https://mariadb.com/kb/en/thread-pool-in-mariadb/
- KB `Query Cache` : https://mariadb.com/kb/en/query-cache/
- KB `InnoDB Buffer Pool` : https://mariadb.com/kb/en/innodb-buffer-pool/
- KB `mariadb-upgrade` (for migration tuning context) : https://mariadb.com/kb/en/upgrading-from-mysql-to-mariadb/
