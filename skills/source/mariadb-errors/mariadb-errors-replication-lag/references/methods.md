# Methods : Replication Lag Measurement and Tuning

Complete API and variable reference for diagnosing and fixing MariaDB replication lag.

## 1. Lag-Measurement Matrix

| Method | Source | Resolution | Reliability | When to use |
|--------|--------|------------|-------------|-------------|
| `SHOW SLAVE STATUS` `Seconds_Behind_Master` | replica | 1 second | LOW : `0` while applier replays a large event ; `NULL` when SQL thread stopped | Quick sanity check only. NEVER for alerting. |
| `SHOW ALL SLAVES STATUS` (10.0+) | replica | 1 second | LOW (same limitation) but multi-channel | Multi-source ; use for per-channel lag visibility. |
| `pt-heartbeat --check` (Percona Toolkit) | replica | sub-second | HIGH : reads a replicated timestamp row | DEFAULT for production lag monitoring. |
| `SELECT TIMESTAMPDIFF(SECOND, ts, NOW())` on heartbeat-table | replica | sub-second | HIGH : same mechanism, custom-built | When pt-heartbeat is not installable. |
| `SHOW MASTER STATUS` on master + `SHOW SLAVE STATUS` on replica, compare binlog file+pos | both | event count | MEDIUM : tells you events-behind, not seconds | When you need event-level lag. |

## 2. System Variables Affecting Lag

### Parallel-Applier

| Variable | Default | Range | Dynamic | Effect on lag |
|----------|---------|-------|---------|---------------|
| `slave_parallel_threads` | `0` | `0-16383` | Yes (STOP SLAVE first) | `0` = single-thread applier. Non-zero = N worker threads. Default `0` is the #1 latent lag source on upgraded servers. |
| `slave_parallel_mode` | `optimistic` (10.5.1+) ; `conservative` (10.5.0 and earlier) | `none`, `minimal`, `conservative`, `optimistic`, `aggressive` | Yes (STOP SLAVE first) | `none`, `minimal` force serial. `conservative` uses group-commit info. `optimistic` and `aggressive` allow more parallelism with conflict-retry. |
| `slave_parallel_max_queued` | `131072` bytes per worker | `0-2147483647` | Yes | Per-worker event queue. Total memory = value x threads. Set higher if events are large. |
| `slave_domain_parallel_threads` | `0` (no per-domain cap) | `0-16383` | Yes | Caps applier threads per GTID domain ; only meaningful for multi-source. |

### Binlog Logging

| Variable | Default | Effect on lag |
|----------|---------|---------------|
| `binlog_format` | `MIXED` (10.2.3+) | `STATEMENT` produces smaller logs but breaks on non-deterministic functions, causing applier re-run mismatches. `ROW` is safest but biggest binlog. |
| `binlog_row_image` | `FULL` | `MINIMAL` reduces binlog size but PREVENTS fix-on-replica reconstruction. `NOBLOB` excludes unchanged BLOB/TEXT. `FULL_NODUP` (11.4+) optimises duplicated-column case. |
| `sync_binlog` | `1` | `1` (default) flushes binlog to disk every commit. Higher values batch flushes, reducing master fsync cost but allowing crash-loss. |

### GTID

| Variable | Default | Effect on lag |
|----------|---------|---------------|
| `gtid_strict_mode` | `OFF` | When `ON`, refuses out-of-sequence GTID events. A stale replica fails fast instead of silently growing lag. |
| `gtid_slave_pos` | empty initially | Replica's last-replicated GTID. Must match master `gtid_binlog_pos` chain or replication stalls. |
| `gtid_current_pos` | derived | Composite of `gtid_binlog_pos` and `gtid_slave_pos`. Use for inspection only. |

### Semi-Sync

| Variable | Default | Effect on lag |
|----------|---------|---------------|
| `rpl_semi_sync_master_enabled` | `OFF` | Built-in from 10.3+. When `ON`, master waits for one replica ack per commit. |
| `rpl_semi_sync_master_timeout` | `10000` ms | When wait exceeds timeout, master FALLS BACK to async. Looks like a lag spike on graphs. |
| `rpl_semi_sync_master_wait_point` | `AFTER_COMMIT` | `AFTER_SYNC` gives stronger durability ; `AFTER_COMMIT` lower latency. |
| `Rpl_semi_sync_master_status` (status var) | `ON` when active | Drops to `OFF` on timeout fallback. Always check this before treating spike as applier lag. |

## 3. Parallel-Applier Sizing Grid

Benchmark `slave_parallel_threads` against the actual replica hardware. The grid below is a STARTING POINT, not a final tuning.

| Replica CPU | Replica disk | Workload | `slave_parallel_threads` start | Notes |
|-------------|--------------|----------|-------------------------------|-------|
| 1-4 cores | HDD | Low write | `2` | HDD IOPS will cap any higher value. |
| 4-8 cores | SSD | OLTP | `4-8` | Sweet spot for most production OLTP. |
| 8-16 cores | NVMe | Heavy write | `8-16` | Above 16, lock contention rises. |
| 16+ cores | NVMe | Analytics-replica | `16` max | Higher = diminishing returns. Always benchmark. |
| Multi-source | any | per-domain isolation | set `slave_domain_parallel_threads = ceil(threads / n_sources)` | Prevents one source starving another. |

Verification :

```sql
-- After setting slave_parallel_threads, confirm workers are running
SELECT COUNT(*) AS worker_count
FROM information_schema.PROCESSLIST
WHERE USER = 'system user'
  AND COMMAND = 'Daemon'
  AND STATE LIKE 'Slave_worker%' ;
```

## 4. pt-heartbeat Install and Usage

`pt-heartbeat` is part of [Percona Toolkit](https://www.percona.com/software/database-tools/percona-toolkit). It writes a timestamp to a row on the master at a configurable interval and reads it on the replica to compute real lag.

```bash
# Debian / Ubuntu
apt-get install percona-toolkit

# RHEL / CentOS / Rocky
yum install percona-toolkit

# Verify
pt-heartbeat --version
```

Setup the heartbeat table (one-time, on master) :

```sql
CREATE DATABASE IF NOT EXISTS heartbeat ;

CREATE TABLE heartbeat.heartbeat (
  ts                    VARCHAR(26) NOT NULL,
  server_id             INT UNSIGNED NOT NULL PRIMARY KEY,
  file                  VARCHAR(255) DEFAULT NULL,
  position              BIGINT UNSIGNED DEFAULT NULL,
  relay_master_log_file VARCHAR(255) DEFAULT NULL,
  exec_master_log_pos   BIGINT UNSIGNED DEFAULT NULL
) ENGINE=InnoDB ;
```

Run as daemon on master :

```bash
pt-heartbeat \
  --daemonize \
  --update \
  --interval=1 \
  -h master.example.com -u monitor -p secret \
  --database heartbeat --table heartbeat \
  --create-table
```

Query lag on replica :

```bash
# Single sample
pt-heartbeat --check \
  -h replica.example.com -u monitor -p secret \
  --database heartbeat --table heartbeat

# Output : e.g. 0.23   (seconds of lag, sub-second resolution)

# Continuous monitoring
pt-heartbeat --monitor \
  -h replica.example.com -u monitor -p secret \
  --database heartbeat --table heartbeat
```

## 5. Monitoring SQL

### Replica side

```sql
-- Full status, multi-source aware (10.0+)
SHOW ALL SLAVES STATUS\G

-- Single-source legacy view
SHOW SLAVE STATUS\G

-- Per-worker applier threads
SELECT ID, USER, HOST, DB, COMMAND, TIME, STATE
FROM information_schema.PROCESSLIST
WHERE USER = 'system user'
ORDER BY ID ;

-- GTID positions
SELECT @@gtid_slave_pos, @@gtid_current_pos, @@gtid_binlog_pos ;

-- Replication-domain status (multi-source)
SELECT * FROM information_schema.SLAVE_STATUS ;   -- 10.5+

-- Current applier configuration
SHOW VARIABLES WHERE Variable_name IN (
  'slave_parallel_threads',
  'slave_parallel_mode',
  'slave_parallel_max_queued',
  'slave_domain_parallel_threads',
  'gtid_strict_mode'
) ;
```

### Master side

```sql
-- Authoritative binlog position
SHOW MASTER STATUS ;
SELECT @@gtid_binlog_pos ;

-- Currently-connected replicas
SHOW SLAVE HOSTS ;

-- Semi-sync status (master)
SHOW GLOBAL STATUS LIKE 'Rpl_semi_sync_master_%' ;

-- Long-running transactions that may produce huge binlog events
SELECT trx_id, trx_started, TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS age_seconds,
       trx_state, trx_query
FROM information_schema.INNODB_TRX
ORDER BY trx_started ASC ;
```

## 6. Identifying Slow Applier Events

When the applier is stuck on a large event :

```sql
-- On the replica, find the running SQL-thread state
SELECT ID, STATE, TIME, INFO
FROM information_schema.PROCESSLIST
WHERE USER = 'system user'
  AND COMMAND = 'Connect'
  AND STATE IS NOT NULL ;

-- If you know the binlog file and position from SHOW SLAVE STATUS, list the events
SHOW BINLOG EVENTS IN 'mariadb-bin.000123' FROM 4 LIMIT 50 ;

-- Events with End_log_pos - Pos > 16 MB are "big". Likely cause : large transactional DML.
```

## 7. Switching Modes Safely

`slave_parallel_threads` and `slave_parallel_mode` are dynamic but require the SQL thread to be stopped. NEVER change them while `Slave_SQL_Running = Yes` : the change will be rejected.

```sql
STOP SLAVE SQL_THREAD ;   -- stop applier, leave IO thread running so binlog keeps streaming
SET GLOBAL slave_parallel_threads = 8 ;
SET GLOBAL slave_parallel_mode    = 'optimistic' ;
START SLAVE SQL_THREAD ;

-- Verify
SHOW VARIABLES LIKE 'slave_parallel_%' ;
SHOW SLAVE STATUS\G
```

For multi-source, stop the specific channel with `STOP SLAVE 'channel_name' SQL_THREAD`.
