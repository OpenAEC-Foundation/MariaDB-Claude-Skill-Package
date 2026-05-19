# MariaDB Core Architecture : Anti-Patterns

Real mis-configurations and mistaken mental models that surface in production. Each entry : the bad code, why it fails, the correct alternative.

## 1. innodb_buffer_pool_size = 50% RAM on a Dedicated DB Host

### Bad

```ini
# my.cnf on a dedicated DB host with 64 GB RAM
[mariadb]
innodb_buffer_pool_size=32G    # "Half the RAM, sounds safe."
```

### Why it fails

On a dedicated DB host, the OS and `mysqld` do not need 32 GB of headroom. The buffer pool is the single biggest performance lever ; under-sizing it forces InnoDB to read pages from disk that could otherwise live in RAM. The KB phrases sizing as "starting at several gigabytes" and warns only against oversizing to the point of OS swapping. 50% leaves working set evictions on the table.

### Correct

```ini
# my.cnf on a dedicated DB host with 64 GB RAM (10.6+)
[mariadb]
innodb_buffer_pool_size=48G    # 75% of RAM, room left for OS + per-thread buffers
```

Verify : monitor `Innodb_buffer_pool_reads` vs `Innodb_buffer_pool_read_requests`. If `reads` delta is < 1% of `read_requests` delta, sizing is right. Source : <https://mariadb.com/kb/en/innodb-buffer-pool/>.

## 2. Enabling Every optimizer_switch Flag Blindly

### Bad

```sql
-- "Turn everything on, optimizer will figure it out."
SET GLOBAL optimizer_switch = 'mrr=on,mrr_cost_based=on,mrr_sort_keys=on,not_null_range_scan=on,index_merge_sort_intersection=on';
```

### Why it fails

Flags default to OFF for a reason. `mrr` (Multi-Range Read) helps random-access patterns over HDDs but adds overhead on SSDs and small result sets. `index_merge_sort_intersection` can be slower than picking the best single index. `not_null_range_scan` benefits very specific NOT NULL column scans, not all queries. Turning everything on globally produces worse plans on representative workloads.

### Correct

```sql
-- 10.6+ : enable only what an A/B test of representative queries justifies
-- 1. Identify the query that benefits
-- 2. Test in session
SET SESSION optimizer_switch = 'mrr=on,mrr_cost_based=on';
EXPLAIN <test query>;
ANALYZE FORMAT=JSON <test query>;
-- 3. Compare to default
SET SESSION optimizer_switch = DEFAULT;
EXPLAIN <test query>;
-- 4. Only apply globally if the win is robust across multiple queries
```

ALWAYS test optimizer_switch changes against actual queries. NEVER copy a global recipe from a blog post. Source : <https://mariadb.com/kb/en/optimizer-switch/>.

## 3. query_cache_type=ON In a Write-Heavy Workload

### Bad

```ini
# my.cnf : "Free performance, just turn it on."
[mariadb]
query_cache_type=ON
query_cache_size=256M
```

### Why it fails

Every write to any cached table invalidates all cache entries for that table. On a write-heavy workload, the invalidation rate exceeds the hit rate, and serialization on the query-cache mutex throttles all connections. The KB explicitly states the cache "does not scale well in environments with high throughput on multi-core machines" and is "disabled by default" in modern MariaDB. MySQL 8.0 removed the feature entirely. MariaDB kept it for read-mostly workloads.

### Correct

```ini
# my.cnf for any modern OLTP workload (10.6+)
[mariadb]
query_cache_type=OFF
query_cache_size=0
```

Use a properly designed application cache (Redis, Memcached) or covering indexes for repeated reads. Source : <https://mariadb.com/kb/en/query-cache/>.

## 4. Assuming MySQL Optimizer Behavior Translates 1-to-1

### Bad

```sql
-- "MySQL 8.0 tuning guide says set this flag."
SET GLOBAL optimizer_switch = 'engine_condition_pushdown=on';
-- Or assuming MySQL hash-join behavior applies
SET GLOBAL optimizer_switch = 'block_nested_loop=off';   -- not a MariaDB flag
```

### Why it fails

MariaDB's optimizer is a separate code path with MariaDB-specific flags (`condition_pushdown_for_subquery`, `rowid_filter`, `split_materialized`, `sargable_casefold`, `hash_join_cardinality`) and missing MySQL flags (`block_nested_loop`, `derived_condition_pushdown`). `engine_condition_pushdown` was deprecated in 10.1 and removed in 11.3, while still recommended in some MySQL guides. Window-function GROUPS frame, JSON storage, and GTID format also differ.

### Correct

ALWAYS consult `mariadb.com/kb/en/optimizer-switch/` for the version-specific defaults. NEVER apply a MySQL tuning recipe without verifying each flag against the MariaDB KB. Run :

```sql
-- 10.6+
SHOW VARIABLES LIKE 'optimizer_switch';
```

and compare to the MariaDB-version default before changing anything. Source : <https://mariadb.com/kb/en/optimizer-switch/>.

## 5. Ignoring Thread-Pool When Serving 1000+ Concurrent Connections

### Bad

```ini
# my.cnf serving 2000 concurrent OLTP connections (default thread handling)
[mariadb]
max_connections=2000
# thread_handling defaults to one-thread-per-connection
sort_buffer_size=8M
join_buffer_size=8M
```

### Why it fails

One-thread-per-connection allocates a kernel thread per session. At 2000 concurrent connections, the kernel scheduler spends measurable time context-switching, and per-thread buffer multiplication explodes : `sort_buffer_size=8M * 2000 = 16 GB` ceiling per allocator class, on top of the buffer pool. The OS thrashes, latency degrades, and tail percentiles collapse under load.

### Correct

```ini
# my.cnf serving 2000 concurrent OLTP connections (MariaDB 10.6+)
[mariadb]
max_connections=2000
thread_handling=pool-of-threads
thread_pool_size=16              # = CPU cores
thread_pool_max_threads=2000
thread_pool_stall_limit=500
thread_pool_idle_timeout=60
sort_buffer_size=2M              # default, do not inflate
join_buffer_size=256K            # default, do not inflate
```

The pool caps concurrent active workers at `thread_pool_size * thread_pool_oversubscribe`, keeping CPU caches warm and bounding kernel scheduling overhead. NEVER use thread-pool for analytics workloads with long-running queries ; stall detection mitigates but does not eliminate head-of-line blocking. Source : <https://mariadb.com/kb/en/thread-pool-in-mariadb/>.

## 6. Inflating Per-Thread Buffers Hoping For Speed

### Bad

```ini
# "Bigger buffers, faster queries."
[mariadb]
sort_buffer_size=256M
read_rnd_buffer_size=64M
join_buffer_size=128M
tmp_table_size=512M
max_heap_table_size=512M
max_connections=500
```

### Why it fails

Per-thread buffers allocate lazily per query, but each value multiplies by `max_connections` as a worst-case ceiling. The configuration above silently approves `sort + join + read_rnd + tmp = (256+128+64+512) * 500 MB = 480 GB` worst-case. Additionally, the `sort_buffer_size` allocator switches to a different (slower) algorithm beyond 256 KB ; the KB and source code note significant gains above default are rare.

### Correct

```ini
# Keep per-thread buffers near defaults ; only raise when EXPLAIN proves the need
[mariadb]
sort_buffer_size=2M              # default
read_rnd_buffer_size=256K        # default
join_buffer_size=256K            # default
tmp_table_size=32M
max_heap_table_size=32M
max_connections=500
```

ALWAYS raise the InnoDB buffer pool first. NEVER raise per-thread buffers globally without `EXPLAIN`-driven evidence on specific queries.

## 7. Treating MariaDB JSON Like MySQL Binary JSON

### Bad

```sql
-- "MariaDB JSON is the same as MySQL JSON, just store it."
CREATE TABLE doc (
   id   INT PRIMARY KEY,
   body JSON
);
CREATE INDEX idx_status ON doc((body->>'$.status'));   -- MySQL-style functional index
```

### Why it fails

MariaDB's `JSON` type is a LONGTEXT alias, not a native binary format. There is no in-place binary representation, no `JSON_EXTRACT` index acceleration like MySQL 5.7.8+. Functional indexes on JSON paths require explicit virtual columns. Carrying MySQL recipes verbatim leads to missing indexes, table scans, and slow `->>` lookups.

### Correct

```sql
-- 10.6+ : MariaDB-idiomatic JSON storage with CHECK + virtual column index
CREATE TABLE doc (
   id   INT PRIMARY KEY,
   body LONGTEXT,                                              -- explicit, makes the alias visible
   CONSTRAINT body_valid CHECK (JSON_VALID(body)),
   status VARCHAR(32) AS (JSON_VALUE(body, '$.status')) VIRTUAL,
   INDEX idx_status (status)
);
```

ALWAYS use `CHECK (JSON_VALID(col))` plus virtual columns with functional indexes. NEVER assume MySQL JSON indexing recipes translate. See `mariadb-syntax-json` skill and L-005 in LESSONS.md.

## 8. Setting innodb_flush_log_at_trx_commit=0 In Production

### Bad

```ini
# "Faster commits, who needs durability."
[mariadb]
innodb_flush_log_at_trx_commit=0
```

### Why it fails

`innodb_flush_log_at_trx_commit=0` writes and flushes the redo log only once per second, accepting up to one second of committed data loss on any crash (`mysqld` crash or OS crash). The KB lists this explicitly. Combined with semi-sync replication, the primary can lose committed transactions that were already acknowledged to clients.

### Correct

```ini
# my.cnf for OLTP requiring durability (10.6+)
[mariadb]
innodb_flush_log_at_trx_commit=1     # Default. Full ACID. Flush every commit.
sync_binlog=1                         # Crash-safe binlog
```

Only deviate when the workload is bulk-ingest or write-cache and the application tolerates loss. Document the tradeoff explicitly. Source : <https://mariadb.com/kb/en/innodb-system-variables/>.

## 9. Forgetting To Enable event_scheduler

### Bad

```sql
-- 10.6+
CREATE EVENT nightly_purge
ON SCHEDULE EVERY 1 DAY STARTS '2026-01-01 03:00:00'
DO DELETE FROM audit_log WHERE created_at < NOW() - INTERVAL 90 DAY;
-- Wait three months, audit_log unchanged
```

### Why it fails

The event scheduler is OFF by default on most distributions. Created events compile and are visible in `information_schema.EVENTS`, but never fire. The default catches every new install.

### Correct

```sql
-- 10.6+
SET GLOBAL event_scheduler = ON;
SHOW VARIABLES LIKE 'event_scheduler';   -- verify ON
```

Persist in `my.cnf` so a restart preserves the setting :

```ini
[mariadb]
event_scheduler=ON
```

## 10. Killing a Long Transaction Without Reading TRANSACTIONS Section First

### Bad

```sql
-- "This query is slow, just kill it."
KILL 12345;
```

### Why it fails

If the connection is in the middle of a large `UPDATE` or `DELETE`, MariaDB rolls back the work, holding row locks for the duration of the rollback. Long transactions block undo purge ; the buffer pool fills with undo records, and `Innodb_history_list_length` grows. A naive `KILL` can lengthen the recovery window rather than shorten it.

### Correct

```sql
-- 10.6+ : assess before killing
SHOW ENGINE INNODB STATUS \G
-- Inspect TRANSACTIONS section : find trx id, statement, rows modified

SELECT * FROM information_schema.INNODB_TRX
WHERE trx_mysql_thread_id = 12345 \G

-- If the transaction modified millions of rows, KILL triggers full rollback.
-- Decide : wait for natural commit, or accept the rollback cost.
KILL 12345;

-- Monitor recovery
SHOW ENGINE INNODB STATUS \G   -- watch ROLLING BACK section drain
```

ALWAYS read `SHOW ENGINE INNODB STATUS` before killing a long-running transaction. NEVER assume `KILL` is instant. Source : <https://mariadb.com/kb/en/innodb-system-variables/>, <https://mariadb.com/kb/en/show-engine/>.
