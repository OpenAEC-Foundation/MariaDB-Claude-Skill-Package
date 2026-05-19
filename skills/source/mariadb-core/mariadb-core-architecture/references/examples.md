# MariaDB Core Architecture : Examples

Working snippets for inspecting the running server, reading diagnostics, and validating optimizer behavior. All examples version-annotated.

## 1. SHOW PROCESSLIST : Read the Connection Roster

```sql
-- 10.6+, 11.x, 12.x
SHOW PROCESSLIST;
```

Typical output columns :

```
+----+-------------+-----------+------+---------+------+--------------------------+------------------------------+
| Id | User        | Host      | db   | Command | Time | State                    | Info                         |
+----+-------------+-----------+------+---------+------+--------------------------+------------------------------+
|  1 | system user |           | NULL | Daemon  |   54 | InnoDB shutdown handler  | NULL                         |
| 12 | app         | 10.0.0.5  | erp  | Query   |    3 | Sending data             | SELECT ... FROM tabInvoice   |
| 17 | repl        | 10.0.0.7  | NULL | Binlog Dump | 600 | Master has sent all binlog to slave; waiting for binlog to be updated | NULL |
+----+-------------+-----------+------+---------+------+--------------------------+------------------------------+
```

Reading rules :
- `Command=Query` + `Time>30` : long-running statement, candidate for `EXPLAIN` review.
- `Command=Sleep` + `Time>wait_timeout/2` : idle client holding a connection, candidate for `KILL`.
- `Command=Binlog Dump` : replica IO thread connected to this primary.
- `State=Waiting for table metadata lock` : blocked by ongoing DDL on another connection.

Full-detail version :

```sql
-- 10.6+
SELECT id, user, host, db, command, time, state, info_binary
FROM information_schema.processlist
WHERE command != 'Sleep'
ORDER BY time DESC;
```

## 2. SHOW ENGINE INNODB STATUS : Interpret Key Sections

```sql
-- 10.6+
SHOW ENGINE INNODB STATUS \G
```

Sections to read first :

```
=====================================
TRANSACTIONS
------------
Trx id counter 1234567
Purge done for trx's n:o < 1234500 undo n:o < 0 state: running but idle
...
---TRANSACTION 1234560, ACTIVE 412 sec
mysql tables in use 1, locked 1
LOCK WAIT 3 lock struct(s), heap size 1136, 2 row lock(s)
```

- `ACTIVE 412 sec` : long transaction, blocking purge ; investigate the session.
- `LOCK WAIT` : connection is waiting on a row lock held by another transaction.
- `Purge done for trx's n:o` lagging behind current LSN : indicates undo bloat.

```
----------------------
BUFFER POOL AND MEMORY
----------------------
Total large memory allocated 25769803776
Buffer pool size   1572864
Free buffers       12345
Database pages     1456789
Pages read 9876543, created 12345, written 65432
Buffer pool hit rate 998 / 1000, young-making rate 5 / 1000 not 12 / 1000
```

- `Buffer pool hit rate 998 / 1000` : 99.8% hit ratio, healthy.
- `Free buffers` close to zero with rising `Innodb_buffer_pool_wait_free` : pool too small.

## 3. Inspecting Memory and Thread Variables

```sql
-- 10.6+, 11.x, 12.x
SHOW VARIABLES LIKE 'thread%';
SHOW VARIABLES LIKE 'innodb_buffer_pool%';
SHOW VARIABLES LIKE 'innodb_log%';
SHOW VARIABLES LIKE 'max_connections';
SHOW VARIABLES LIKE 'optimizer_switch';
```

Sample output for `optimizer_switch` (10.11 LTS) :

```
optimizer_switch  index_merge=on,index_merge_union=on,index_merge_sort_union=on,
                  index_merge_intersection=on,index_merge_sort_intersection=off,
                  engine_condition_pushdown=off,index_condition_pushdown=on,
                  derived_merge=on,derived_with_keys=on,firstmatch=on,loosescan=on,
                  materialization=on,in_to_exists=on,semijoin=on,partial_match_rowid_merge=on,
                  partial_match_table_scan=on,subquery_cache=on,mrr=off,mrr_cost_based=off,
                  mrr_sort_keys=off,outer_join_with_cache=on,semijoin_with_cache=on,
                  join_cache_incremental=on,join_cache_hashed=on,join_cache_bka=on,
                  optimize_join_buffer_size=on,table_elimination=on,extended_keys=on,
                  exists_to_in=on,orderby_uses_equalities=on,condition_pushdown_for_derived=on,
                  split_materialized=on,condition_pushdown_for_subquery=on,rowid_filter=on,
                  condition_pushdown_from_having=on,not_null_range_scan=off
```

## 4. EXPLAIN : Walk a Query Through the Pipeline

```sql
-- 10.6+
EXPLAIN
SELECT i.id, i.customer, SUM(d.amount)
FROM tabInvoice i
JOIN tabInvoiceDetail d ON d.parent = i.name
WHERE i.posting_date BETWEEN '2026-01-01' AND '2026-12-31'
  AND i.docstatus = 1
GROUP BY i.id, i.customer;
```

Typical plan :

```
+----+-------------+-------+-------+---------------+---------+---------+--------------+-------+----------------------------------+
| id | select_type | table | type  | possible_keys | key     | key_len | ref          | rows  | Extra                            |
+----+-------------+-------+-------+---------------+---------+---------+--------------+-------+----------------------------------+
|  1 | SIMPLE      | i     | range | posting_date  | posting_date | 4 | NULL         | 18234 | Using where; Using temporary;    |
|                                                                                                  Using filesort                  |
|  1 | SIMPLE      | d     | ref   | parent_idx    | parent_idx | 767 | erp.i.name |    14 | Using index                      |
+----+-------------+-------+-------+---------------+---------+---------+--------------+-------+----------------------------------+
```

Reading rules :
- `type=range` on `i` : good, used the `posting_date` index.
- `type=ref` on `d` : excellent, indexed join.
- `Using temporary; Using filesort` : the optimizer materializes a temp table for GROUP BY. Add a covering composite index `(posting_date, docstatus, customer)` to remove.
- `type=ALL` on a large table : never ship.

## 5. ANALYZE SELECT : Actual vs Estimated Rows

```sql
-- 10.6+
ANALYZE FORMAT=JSON
SELECT i.id, i.customer
FROM tabInvoice i
WHERE i.posting_date BETWEEN '2026-01-01' AND '2026-01-31';
```

Look for the `r_rows` versus `rows` divergence : if the optimizer estimated 1000 rows and `r_rows=120000`, statistics are stale. Fix with `ANALYZE TABLE tabInvoice`.

ALWAYS use `ANALYZE SELECT`. NEVER use `ANALYZE UPDATE` or `ANALYZE DELETE` as a diagnostic : both execute the mutation.

## 6. Toggling optimizer_switch

```sql
-- 10.6+ : enable Multi-Range Read for a session test
SET SESSION optimizer_switch = 'mrr=on,mrr_cost_based=on';

EXPLAIN SELECT * FROM big_table WHERE indexed_col IN (1, 2, 3, /* ... thousands */);

-- Reset
SET SESSION optimizer_switch = DEFAULT;
```

NEVER apply `SET GLOBAL optimizer_switch = ...` in production without A/B testing on representative queries. Several flags are OFF by default because the cost model rejects them in common workloads.

## 7. Optimizer Trace : Deep-Dive Diagnostics

```sql
-- 10.6+
SET optimizer_trace = 'enabled=on';

SELECT * FROM tabInvoice WHERE customer = 'CUST-001' AND posting_date > '2026-01-01';

SELECT * FROM information_schema.OPTIMIZER_TRACE \G

SET optimizer_trace = 'enabled=off';
```

The trace shows every transformation considered, every index ranked, every join order tried, with cost estimates. Use only when `EXPLAIN` and `ANALYZE SELECT` are insufficient.

## 8. Log File Locations

```sql
-- 10.6+, 11.x, 12.x
SHOW VARIABLES LIKE 'log_error';
SHOW VARIABLES LIKE 'slow_query_log_file';
SHOW VARIABLES LIKE 'general_log_file';
SHOW VARIABLES LIKE 'datadir';
```

Disk artifacts under `datadir` :

```
ib_logfile0          -- 10.5+ single redo log file
ibdata1              -- System tablespace
mysql.err            -- Error log (or @log_error path)
mysql-slow.log       -- Slow query log (if enabled)
mysql-bin.000001     -- Binary log file 1
mysql-bin.index      -- Binary log index
<schema>/<table>.ibd -- Per-table tablespace (innodb_file_per_table=ON, default)
undo001, undo002     -- Separate undo tablespaces (10.6+)
aria_log.00000001    -- Aria crash recovery log
```

## 9. Enable Thread-Pool For High Concurrency

```ini
# my.cnf (MariaDB 10.6+, Linux)
[mariadb]
thread_handling=pool-of-threads
thread_pool_size=16                    # = number of CPU cores
thread_pool_max_threads=2000
thread_pool_stall_limit=500            # ms
thread_pool_idle_timeout=60            # s
thread_pool_oversubscribe=3
```

Verify after restart :

```sql
-- 10.6+
SHOW VARIABLES LIKE 'thread_handling';
SHOW STATUS LIKE 'Threadpool_threads';
SHOW STATUS LIKE 'Threadpool_idle_threads';
```

## 10. Sizing the InnoDB Buffer Pool

```ini
# my.cnf (MariaDB 10.6+ on a dedicated DB host with 32 GB RAM)
[mariadb]
innodb_buffer_pool_size=24G
innodb_log_file_size=2G
innodb_log_buffer_size=64M
innodb_flush_log_at_trx_commit=1
sync_binlog=1
innodb_io_capacity=2000          # SSD baseline
innodb_io_capacity_max=4000      # SSD burst
innodb_thread_concurrency=0      # unlimited, modern default
innodb_file_per_table=ON         # default, kept for clarity
query_cache_type=OFF
query_cache_size=0
```

Verify hit ratio after a representative load :

```sql
-- 10.6+
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_reads';
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_read_requests';
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_wait_free';
```

If `Innodb_buffer_pool_wait_free` grows over time, the pool is too small. If `Innodb_buffer_pool_reads` is < 1% of `Innodb_buffer_pool_read_requests` deltas, sizing is good.

## 11. Dynamically Resize Buffer Pool (10.11.12+, 11.4.6+, 11.8.2+)

```sql
-- 10.11.12+, 11.4.6+, 11.8.2+
-- innodb_buffer_pool_size_max must be set at startup, e.g. 32G in my.cnf
SET GLOBAL innodb_buffer_pool_size = 28 * 1024 * 1024 * 1024;
-- Statement now blocks until resize completes.
```

NEVER attempt a runtime resize beyond `innodb_buffer_pool_size_max` ; the value must be set at startup and is the hard ceiling.

## 12. Compose the Connection-Lifecycle Diagnostic Bundle

```sql
-- 10.6+
SELECT
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME='Threads_connected') AS connected,
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME='Threads_running')   AS running,
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME='Threads_created')   AS created,
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES WHERE VARIABLE_NAME='max_connections') AS max_conn,
  (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_VARIABLES WHERE VARIABLE_NAME='thread_cache_size') AS thread_cache;
```

If `created` rises with every connect-disconnect cycle, raise `thread_cache_size` (or move to thread-pool mode for 1000+ concurrent).
