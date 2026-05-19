---
name: mariadb-errors-replication-lag
description: >
  Use when a replica is reporting Seconds_Behind_Master > 0 and growing, replication appears stuck, slave_parallel_threads is not helping, or you need to measure actual replication lag.
  Prevents the common mistake of trusting Seconds_Behind_Master as a real-time metric, leaving a single-threaded applier on a legacy setup, long master-transactions creating cascading lag, or binlog_row_image=MINIMAL causing replay or fix-on-replica failures.
  Covers measuring lag accurately with pt-heartbeat (NEVER with Seconds_Behind_Master alone), common root causes (single-threaded applier, long master transactions, replica IO bottleneck, large binlog events, GTID position mismatch with gtid_strict_mode), fixing with parallel-applier modes and threads, GTID position checks, and semi-sync timeout misunderstanding.
  Keywords: replication lag, Seconds_Behind_Master, replica behind, slave lag, pt-heartbeat, slave_parallel_threads, slave_parallel_mode optimistic, binlog_row_image MINIMAL, GTID position mismatch, IO bottleneck on replica, my replica is falling behind, replication broken, slave SQL thread, slave IO thread, replication keeps failing, gtid_strict_mode, rpl_semi_sync_master_timeout, semi-sync fallback to async, SHOW ALL SLAVES STATUS, slave_domain_parallel_threads, slave_parallel_max_queued, binlog event too large, long master transaction blocks applier, why is my replica slow
license: MIT
compatibility: "Designed for Claude Code. Requires MariaDB 10.6-LTS, 10.11-LTS, 11.x, 12.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# MariaDB Replication Lag

How to measure, diagnose, and fix replication lag on a MariaDB replica. Covers the unreliability of `Seconds_Behind_Master`, parallel-applier tuning, long-master-transaction root causes, GTID position checks, and the semi-sync-to-async fallback that looks like a lag spike but is not.

## Quick Reference

- `Seconds_Behind_Master` from `SHOW SLAVE STATUS` is UNRELIABLE for real-time lag : it updates only between binlog events and reports `0` while the applier is mid-replay of a large transaction. NEVER use it alone for alerting. ALWAYS pair with `pt-heartbeat` (Percona Toolkit) for true lag.
- `slave_parallel_threads` default is `0` (single-threaded applier). On any modern workload, set to `4-16` based on disk-IO benchmark. The value is dynamic but requires `STOP SLAVE ; SET GLOBAL ... ; START SLAVE`.
- `slave_parallel_mode` default is `optimistic` from 10.5.1+ (was `conservative` until 10.5.0). NEVER set to `none` in production unless you are reproducing a determinism bug ; that forces single-thread applier even when `slave_parallel_threads` is non-zero.
- Long master transactions (single `UPDATE` touching millions of rows, batch `DELETE` without `LIMIT`) produce one huge binlog event that the applier MUST process serially. ALWAYS batch large DML on the master into chunks of 1000-10000 rows.
- `binlog_row_image=MINIMAL` reduces binlog volume but RECORDS only the primary key in the before-image. Fix-on-replica row reconstruction and point-in-time recovery to a specific row state both BREAK with `MINIMAL`. NEVER set `MINIMAL` on PITR-critical or multi-master fix-able instances.
- `gtid_strict_mode=ON` REFUSES any out-of-sequence GTID. A failed-over or partially-re-cloned replica with a stale `gtid_slave_pos` will stop with `ER_GTID_STRICT_OUT_OF_ORDER` and silently grow lag until reset.
- Semi-sync fallback to async (when `rpl_semi_sync_master_timeout` is reached) appears as a lag spike on monitoring graphs but is NOT a real applier-side lag. ALWAYS check `Rpl_semi_sync_master_status` (`ON` vs `OFF`) before treating a spike as applier-thread slowness.
- Replica disk slower than master = permanent lag. NEVER pair a HDD replica with an SSD master for an OLTP workload ; the applier IO cannot catch up.

## Error Identification

| Symptom | Likely cause | Diagnose with |
|---------|--------------|---------------|
| `Seconds_Behind_Master` oscillates between 0 and growing | Large transaction on master, applier mid-replay | `pt-heartbeat` for true lag ; `SHOW PROCESSLIST` to see applier-thread state |
| `Seconds_Behind_Master = NULL` | Slave IO thread or SQL thread not running | `SHOW SLAVE STATUS\G` -> `Slave_IO_Running`, `Slave_SQL_Running`, `Last_IO_Error`, `Last_SQL_Error` |
| Lag grows monotonically, never drains | Single-threaded applier on multi-write workload, OR replica disk slower than master | Check `slave_parallel_threads` ; benchmark replica IO with `fio` |
| Lag spike then recovers within seconds | Semi-sync timeout, master fell back to async | `SHOW GLOBAL STATUS LIKE 'Rpl_semi_sync_master_status'` ; expect `OFF` during spike |
| `ER_GTID_STRICT_OUT_OF_ORDER` (error 1962) | `gtid_strict_mode=ON` and replica has stale `gtid_slave_pos` | `SELECT @@gtid_slave_pos, @@gtid_current_pos ;` compare with master `@@gtid_binlog_pos` |
| Replication stops with `binlog event too large` | `max_allowed_packet` lower on replica than master | Match `max_allowed_packet` on both ; restart replication |
| Replica catches up partially then stalls again | Long master transaction in flight ; one big event blocks applier serially | `SHOW BINLOG EVENTS IN 'file' FROM pos LIMIT 50` to see event size |

## Decision Tree

```
Replica reports lag
|
+-- Is the lag value from Seconds_Behind_Master only ?
|     YES : you do NOT yet know if lag is real. Set up pt-heartbeat (see methods.md).
|     NO  : continue.
|
+-- Is Slave_IO_Running = No or Slave_SQL_Running = No ?
|     YES : replication is BROKEN, not lagging. Read Last_IO_Error / Last_SQL_Error and
|           fix root cause before measuring lag.
|     NO  : continue.
|
+-- Is slave_parallel_threads = 0 ?
|     YES : single-thread applier. Set to 4-16 (see methods.md sizing matrix), STOP+SET+START.
|     NO  : continue.
|
+-- Is slave_parallel_mode = none or minimal ?
|     YES : applier is effectively serial. Set to optimistic (default 10.5.1+).
|     NO  : continue.
|
+-- Is the replica disk slower than master disk ?
|     YES : permanent lag. Either upgrade replica disk or accept lag SLA.
|     NO  : continue.
|
+-- Is there a long master transaction in flight ?
|     Check SHOW PROCESSLIST on master for transactions in State 'Sending data' > 30s.
|     YES : the binlog event is huge. Wait for it OR break the DML into batches on master.
|     NO  : continue.
|
+-- Is Rpl_semi_sync_master_status = OFF (fallback) ?
|     YES : not a real applier lag, it is a network or replica-ack issue.
|     NO  : check GTID position via SELECT @@gtid_slave_pos, @@gtid_current_pos.
```

## Key System Variables

| Variable | Default | Scope | Dynamic | Purpose |
|----------|---------|-------|---------|---------|
| `slave_parallel_threads` | `0` (single-thread applier) | Global | Yes (STOP SLAVE first) | Worker pool size for parallel applier ; range 0-16383 |
| `slave_parallel_mode` | `optimistic` (10.5.1+) ; `conservative` (10.5.0 and earlier) | Global | Yes (STOP SLAVE first) | Conflict-handling mode for parallel applier |
| `slave_parallel_max_queued` | `131072` bytes per worker | Global | Yes | Per-worker event queue ; total memory = value x threads |
| `slave_domain_parallel_threads` | `0` (no per-domain cap) | Global | Yes | Caps applier threads per GTID domain for multi-source setups |
| `binlog_row_image` | `FULL` | Global, Session | Yes | `FULL`, `MINIMAL`, `NOBLOB`, `FULL_NODUP` (11.4+) |
| `binlog_format` | `MIXED` (10.2.3+) ; `STATEMENT` earlier | Global, Session | Yes | Logging format ; `ROW` recommended for non-deterministic workloads |
| `gtid_strict_mode` | `OFF` | Global | Yes | When `ON`, refuses out-of-sequence GTID events |
| `rpl_semi_sync_master_timeout` | `10000` ms | Global | Yes | Master falls back to async after this wait |
| `rpl_semi_sync_master_wait_point` | `AFTER_COMMIT` | Global | Yes | `AFTER_SYNC` or `AFTER_COMMIT` ; affects durability vs latency |
| `max_allowed_packet` | `16777216` (16 MB) | Global, Session | Yes | Must be >= on replica what master sends, else applier fails |

All defaults verified against the [MariaDB KB replication-and-binary-log-system-variables page](https://mariadb.com/kb/en/replication-and-binary-log-system-variables/), [parallel-replication page](https://mariadb.com/kb/en/parallel-replication/), and [semisynchronous-replication page](https://mariadb.com/kb/en/semisynchronous-replication/).

## Measuring Lag Accurately

`Seconds_Behind_Master` is computed as `now - timestamp_of_last_event_started_executing`. It does NOT see events still being applied, and it goes to `0` when the applier finishes one event before the next one arrives. The real cure is `pt-heartbeat`, which inserts a timestamp row on the master every N seconds and lets the replica compute lag as `now - timestamp_in_replicated_row`.

```bash
# Install Percona Toolkit (Debian/Ubuntu)
apt-get install percona-toolkit

# On master : start the heartbeat-emitter daemon
pt-heartbeat --daemonize --update \
  -h master.example.com -u monitor -p secret \
  --database heartbeat --table heartbeat

# On replica : measure lag from the replicated heartbeat row
pt-heartbeat --check -h replica.example.com -u monitor -p secret \
  --database heartbeat --table heartbeat
# Prints a single number : seconds of lag (sub-second resolution)
```

See `references/methods.md` for the full pt-heartbeat installation and monitoring-query matrix.

## Fixing : Parallel Applier Tuning

Single-threaded applier is the #1 cause of avoidable lag on 10.x setups that were upgraded from pre-10.5 without revisiting replication tuning.

```sql
-- MariaDB 10.5.1+ : enable parallel applier (default mode optimistic from 10.5.1+)
STOP SLAVE ;
SET GLOBAL slave_parallel_threads = 8 ;          -- start at 8 ; benchmark to confirm
SET GLOBAL slave_parallel_mode    = 'optimistic' ; -- already default 10.5.1+
START SLAVE ;
SHOW SLAVE STATUS\G                                -- confirm Slave_IO_Running and Slave_SQL_Running = Yes
```

Sizing rule (see `references/methods.md` for full matrix) :

- 1-4 cores, HDD replica : `slave_parallel_threads = 2`
- 8 cores, SSD replica   : `slave_parallel_threads = 8`
- 16+ cores, NVMe replica : `slave_parallel_threads = 16` (do not exceed 16 without benchmarking ; lock contention rises)

NEVER set `slave_parallel_threads` higher than `2 * cpu_cores` without measuring. Worker-thread context-switch overhead can REDUCE throughput beyond a point.

## Fixing : Long Master Transactions

A single `DELETE FROM t WHERE created < '2020-01-01'` that removes 50 million rows produces one massive binlog event. The replica applier MUST process this event in one piece (even with `slave_parallel_threads`, a single event runs on one worker). The fix is to batch the DML on the master :

```sql
-- BAD : one huge event blocks the applier for minutes
DELETE FROM events WHERE created < '2020-01-01' ;

-- GOOD : batch into chunks of 10000 rows ; one event per chunk
WHILE (1) DO
  DELETE FROM events WHERE created < '2020-01-01' LIMIT 10000 ;
  IF ROW_COUNT() = 0 THEN LEAVE ; END IF ;
  DO SLEEP(0.1) ;
END WHILE ;
```

See `references/examples.md` for the same pattern in `pt-archiver` (recommended for production-grade chunked deletes).

## Fixing : GTID Position Mismatch

When `gtid_strict_mode=ON` and the replica has been reseeded or failed-over with a stale `gtid_slave_pos`, replication stops with `ER_GTID_STRICT_OUT_OF_ORDER` (error 1962).

```sql
-- On replica : check current GTID positions
SELECT @@gtid_slave_pos AS slave_pos,
       @@gtid_current_pos AS current_pos,
       @@gtid_binlog_pos  AS binlog_pos ;

-- On master : the authoritative position
SELECT @@gtid_binlog_pos ;

-- If the replica gtid_slave_pos is missing or wrong, reset it explicitly
STOP SLAVE ;
SET GLOBAL gtid_slave_pos = '0-1-12345' ;   -- value from master's gtid_binlog_pos
START SLAVE ;
```

NEVER blindly disable `gtid_strict_mode` to make the error go away. The strict-mode check exists to prevent silent binlog divergence ; turning it off masks a real consistency risk.

## Fixing : Semi-Sync Fallback Misinterpretation

If a monitoring graph shows lag spikes that recover within seconds and you see this on the master :

```sql
SHOW GLOBAL STATUS LIKE 'Rpl_semi_sync_master_status' ;
-- Value : OFF      <- master has fallen back to async
SHOW GLOBAL STATUS LIKE 'Rpl_semi_sync_master_no_times' ;
-- Counts how many times this happened
```

This is NOT applier-thread slowness. The master waited longer than `rpl_semi_sync_master_timeout` (default 10000 ms) for the replica acknowledgement and dropped to async. Investigate the network round-trip and the replica's binlog-write latency, NOT the SQL thread.

```ini
# my.cnf : tune timeout based on actual network RTT, not default
[mariadb]
rpl_semi_sync_master_timeout = 30000   # 30 seconds, for cross-AZ replicas
rpl_semi_sync_master_wait_point = AFTER_SYNC   # for stronger durability on master crash
```

## Monitoring Queries

```sql
-- Lag and replication state (multi-source aware, 10.0+)
SHOW ALL SLAVES STATUS\G

-- Single-source legacy form
SHOW SLAVE STATUS\G

-- Applier-worker status (one row per parallel worker thread)
SELECT * FROM information_schema.PROCESSLIST
WHERE COMMAND LIKE 'Slave_%' OR USER = 'system user' ;

-- GTID positions
SELECT @@gtid_slave_pos, @@gtid_current_pos, @@gtid_binlog_pos ;

-- Semi-sync status (master side)
SHOW GLOBAL STATUS LIKE 'Rpl_semi_sync_%' ;
```

`SHOW ALL SLAVES STATUS` (10.0+, see [KB multi-source-replication](https://mariadb.com/kb/en/multi-source-replication/)) returns one row per replication channel. NEVER use plain `SHOW SLAVE STATUS` on a multi-source setup ; it returns only the default-channel row and hides per-channel lag.

## Cross-References

- `mariadb-errors-galera-conflicts` : for Galera-cluster certification-time errors that look like lag spikes but are not.
- `mariadb-impl-replication-setup` : for initial replication configuration including `MASTER_USE_GTID` and TLS.
- `mariadb-core-architecture` : for the binlog group-commit and applier-worker architecture.

## References

- `references/methods.md` : full lag-measurement matrix, system variable interactions, parallel-applier tuning grid, pt-heartbeat installation and queries, monitoring SQL.
- `references/examples.md` : 10+ working examples (install pt-heartbeat, enable parallel applier, identify slow-applier transactions, switch slave_parallel_mode, batch large DML, GTID position reset, semi-sync timeout reproduction, multi-source per-channel monitoring).
- `references/anti-patterns.md` : 8+ documented anti-patterns with root cause and the correct alternative.

## Approved Sources

- [MariaDB KB : Replication and Binary Log System Variables](https://mariadb.com/kb/en/replication-and-binary-log-system-variables/)
- [MariaDB KB : Parallel Replication](https://mariadb.com/kb/en/parallel-replication/)
- [MariaDB KB : Semisynchronous Replication](https://mariadb.com/kb/en/semisynchronous-replication/)
- [MariaDB KB : GTID](https://mariadb.com/kb/en/gtid/)
- [MariaDB KB : Multi-Source Replication](https://mariadb.com/kb/en/multi-source-replication/)
- [MariaDB KB : Binary Log Formats](https://mariadb.com/kb/en/binary-log-formats/)
