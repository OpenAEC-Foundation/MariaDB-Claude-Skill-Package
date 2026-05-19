# Anti-Patterns : MariaDB Performance Tuning

Eight real-world tuning anti-patterns, with the broken code, why it fails, and the fix.

## AP-01 : innodb_buffer_pool_size = 50% of RAM on a dedicated DB host

### Broken

```ini
# 64 GB RAM dedicated MariaDB host
[mariadb]
innodb_buffer_pool_size = 32G       # 50% of RAM ; under-sized
```

### Why it fails

On a dedicated DB host the OS plus mariadbd process baseline is ~2-3 GB. Leaving 32 GB unused is leaving 50% of the page-cache budget on the floor. KB explicitly recommends "up to 80%". Hit-ratio stays in the 95-98% range when it could be 99.9%, every cache miss adds disk-read latency.

### Fix

```ini
# 64 GB RAM dedicated MariaDB host
[mariadb]
innodb_buffer_pool_size = 48G       # 75% of RAM
# Leave 16 GB for : OS page cache, mariadbd thread stacks, per-connection
# buffers, monitoring agents, OS reservation.
```

Measure after 24 h with the buffer-pool hit-ratio query from `examples.md` §5.

## AP-02 : innodb_flush_log_at_trx_commit = 0 on financial data

### Broken

```ini
# "Make it faster"
[mariadb]
innodb_flush_log_at_trx_commit = 0
sync_binlog                    = 0
```

### Why it fails

`innodb_flush_log_at_trx_commit=0` writes and flushes the redo log only once per second. ANY crash (process, OS, power) loses up to 1 second of committed transactions. On financial / ERP / regulated workloads this loses paid orders, processed payments, audit-trail entries. Recovery cannot synthesize "what the user did in the last 800 ms". This is a regulatory violation in PCI, SOX, GDPR-record contexts.

### Fix

```ini
# Financial workload : full ACID
[mariadb]
innodb_flush_log_at_trx_commit = 1
sync_binlog                    = 1
innodb_doublewrite             = ON
```

If throughput is the actual concern, profile the bottleneck with `SHOW ENGINE INNODB STATUS\G` before weakening durability. Storage IOPS is usually the issue, not commit-flush cost. Move to NVMe before touching durability.

## AP-03 : query_cache_type = ON on a write-heavy workload

### Broken

```ini
# 10.6+ on an OLTP order-management system
[mariadb]
query_cache_type = ON
query_cache_size = 512M
```

### Why it fails

The query cache uses a global mutex around the cache table. Every INSERT, UPDATE, DELETE on a cached table invalidates entries under that mutex. On a multi-core OLTP system every write serializes through this global lock. KB confirms : "It does not scale well in environments with high throughput on multi-core machines, so it is disabled by default." Throughput drops, latencies spike, mutex contention dominates `SHOW ENGINE INNODB STATUS\G`.

### Fix

```ini
# Default, since 10.1.7
[mariadb]
query_cache_type = OFF
query_cache_size = 0
```

The query cache is only justified on a >99% read workload against a mostly-static dataset (see `examples.md` §10). On any OLTP system it MUST be OFF.

## AP-04 : sort_buffer_size = 256M with max_connections = 1000

### Broken

```ini
# "Sorts are slow, raise sort_buffer_size"
[mariadb]
sort_buffer_size = 256M
max_connections  = 1000
```

### Why it fails

`sort_buffer_size` is a per-session allocation. With 1000 connections potentially all sorting, the peak commitment is `256 MB * 1000 = 256 GB` of address space on top of `innodb_buffer_pool_size` and the OS. On a 64 GB host this triggers OOM-kill the moment a burst of sorting queries lands. The KB documents the per-connection allocation in the `Server System Variables` page ; vooronderzoek §4 paragraph on per-connection buffers calls this out as the top tuning trap.

### Fix

```ini
# Keep global default small ; raise per-session for known heavy queries
[mariadb]
sort_buffer_size = 2M               # default
max_connections  = 1000
```

```sql
-- 10.6+ : for the one nightly report that needs a big sort
SET SESSION sort_buffer_size = 64 * 1024 * 1024;
SELECT ... ORDER BY ... ;
```

Verify the per-connection memory ceiling with the calculation query in `examples.md` §7.

## AP-05 : Leaving innodb_io_capacity = 200 on NVMe SSD

### Broken

```ini
# Migrated from spinning disk to NVMe, kept the old config
[mariadb]
innodb_io_capacity     = 200        # HDD default
innodb_io_capacity_max = 400
```

### Why it fails

`innodb_io_capacity` caps background-flush throughput. On NVMe capable of 500000+ IOPS, capping at 200 IOPS throttles InnoDB to HDD-class flushing. Symptoms : dirty-page percentage in the buffer pool climbs and never drops, checkpoint age grows, redo log fills, eventually the server stalls on `Innodb_buffer_pool_wait_free`. Storage is idle ; mariadbd refuses to use it.

### Fix

```ini
# NVMe baseline
[mariadb]
innodb_io_capacity     = 8000
innodb_io_capacity_max = 16000
```

Numbers depend on the device. `fio` benchmark the disk first ; set `innodb_io_capacity` to a sustainable IOPS the device handles for hours, `_max` to roughly 2x that for burst.

## AP-06 : thread_pool_size = 1000 (oversubscribed)

### Broken

```ini
# "We have 1000 connections, give the pool 1000 workers"
[mariadb]
thread_handling  = pool-of-threads
thread_pool_size = 1000
```

### Why it fails

`thread_pool_size` is the number of thread GROUPS, each running a small set of worker threads. KB warns explicitly : "the max value is 100000. However, it is not a good idea to set it that high." With 1000 thread groups on a 16-core host, context-switching overhead dominates ; the OS scheduler thrashes. The whole point of the thread pool is to keep concurrent active threads near the CPU-core count.

### Fix

```ini
# Match CPU cores ; let oversubscribe expand under load
[mariadb]
thread_handling           = pool-of-threads
thread_pool_size          = 16          # = physical CPU cores
thread_pool_oversubscribe = 3           # default ; allows burst above 16
thread_pool_max_threads   = 1000        # cap, not target
```

The pool serializes incoming connections onto `thread_pool_size` workers. 2000 client connections still works with 16 workers ; that is the design.

## AP-07 : innodb_log_file_size = 64 MiB on a write-heavy production

### Broken

```ini
# Tiny log on a write-heavy OLTP system
[mariadb]
innodb_log_file_size   = 64M
innodb_log_buffer_size = 16M
```

### Why it fails

Under sustained write load the redo log fills in seconds. Each fill forces a synchronous checkpoint flush of all dirty pages in the buffer pool. The server stalls (sometimes for seconds) every time. `SHOW ENGINE INNODB STATUS\G` shows the `LOG` section with `Last checkpoint at` lagging far behind `Log sequence number`, and `Innodb_buffer_pool_wait_free` climbs in `SHOW GLOBAL STATUS`.

### Fix

```ini
# Write-heavy, 48 G buffer pool
[mariadb]
innodb_log_file_size   = 8G          # ~16% of buffer pool
innodb_log_buffer_size = 64M
```

Trade-off : larger redo log means slower crash recovery. On NVMe, 8 GB recovers in seconds. Size up to the largest value whose recovery time fits inside the RTO.

## AP-08 : Copying a MySQL 8 my.cnf verbatim to MariaDB

### Broken

```ini
# Pasted from a MySQL 8 production tutorial
[mysqld]
innodb_buffer_pool_instances = 8                 # REMOVED in MariaDB 10.6.0+
innodb_dedicated_server      = ON                # MySQL-only, does NOT exist in MariaDB
default_authentication_plugin = caching_sha2_password   # MySQL 8-only ; MariaDB uses different plugins
innodb_redo_log_capacity     = 8G                # MySQL 8.0.30+ ; replaced innodb_log_file_size
```

### Why it fails

- `innodb_buffer_pool_instances` is REMOVED in MariaDB 10.6.0+. The server logs an unknown-variable warning and ignores it.
- `innodb_dedicated_server` does not exist in MariaDB ; the auto-tuning logic is MySQL-only.
- `caching_sha2_password` is MySQL 8's default plugin ; MariaDB uses `mysql_native_password`, `ed25519`, or `unix_socket`. Setting an unknown plugin can prevent the server starting.
- `innodb_redo_log_capacity` (single variable replacing log-file-size in MySQL 8.0.30+) does NOT exist in MariaDB ; MariaDB still uses `innodb_log_file_size`.

This anti-pattern wastes hours of "why won't the server start" debugging.

### Fix

```ini
# MariaDB 10.6+ / 10.11+ / 11.x equivalent
[mariadb]
# NO innodb_buffer_pool_instances (removed)
# NO innodb_dedicated_server (does not exist)
innodb_buffer_pool_size       = 48G
innodb_log_file_size          = 8G          # NOT innodb_redo_log_capacity
innodb_log_buffer_size        = 64M

# Authentication : use MariaDB-native plugins (see mariadb-syntax-grants-and-roles)
# CREATE USER 'alice'@'%' IDENTIFIED VIA ed25519 USING PASSWORD('secret');
```

ALWAYS verify each variable against the MariaDB KB (`https://mariadb.com/kb/en/innodb-system-variables/`) before copying a MySQL config. The variable namespaces diverge increasingly with every MySQL major release.
