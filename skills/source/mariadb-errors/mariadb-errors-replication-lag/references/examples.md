# Examples : Replication Lag Diagnosis and Repair

Working, version-annotated examples for measuring lag, tuning the applier, and fixing the most common root causes.

---

## Example 1 : Install and Use pt-heartbeat (10.6+, 10.11+, 11.x, 12.x)

```bash
# 1. Install Percona Toolkit on master and replica
apt-get install percona-toolkit

# 2. Create heartbeat database and grant on master
mariadb -h master -u root -p <<'SQL'
CREATE DATABASE IF NOT EXISTS heartbeat ;
CREATE USER IF NOT EXISTS 'pt_heartbeat'@'%' IDENTIFIED BY 'hb_secret' ;
GRANT CREATE, INSERT, UPDATE, DELETE, SELECT ON heartbeat.* TO 'pt_heartbeat'@'%' ;
SQL

# 3. Start the heartbeat-emitter daemon on master (writes every 1 second)
pt-heartbeat \
  --daemonize --update --interval=1 --create-table \
  -h master.example.com -u pt_heartbeat -p hb_secret \
  --database heartbeat --table heartbeat \
  --pid /var/run/pt-heartbeat.pid

# 4. On replica : measure real lag (output is a number of seconds)
pt-heartbeat --check \
  -h replica.example.com -u pt_heartbeat -p hb_secret \
  --database heartbeat --table heartbeat
# 0.04   <- sub-second lag, much more reliable than Seconds_Behind_Master
```

---

## Example 2 : Discover that Seconds_Behind_Master is Lying (10.6+)

```sql
-- On the replica, while a large DELETE is replaying on the master
mysql> SHOW SLAVE STATUS\G
*************************** 1. row ***************************
              Slave_IO_Running: Yes
             Slave_SQL_Running: Yes
         Seconds_Behind_Master: 0     <-- LIES : the applier is mid-replay of a huge event
                 Last_SQL_Error:
                 ...

-- At the same moment, pt-heartbeat reports the truth :
$ pt-heartbeat --check -h replica ...
47.2     <-- actual lag is 47 seconds
```

Lesson : `Seconds_Behind_Master` updates only when the applier finishes one event and starts the next. During a long single-event replay, it remains at `0`.

---

## Example 3 : Enable Parallel Applier on a Legacy Replica (10.6+)

```sql
-- Inspect current state
SHOW VARIABLES LIKE 'slave_parallel_threads' ;       -- value : 0 (single-thread)
SHOW VARIABLES LIKE 'slave_parallel_mode' ;          -- value : optimistic (default 10.5.1+)

-- Enable parallel applier with 8 workers
STOP SLAVE SQL_THREAD ;
SET GLOBAL slave_parallel_threads = 8 ;
START SLAVE SQL_THREAD ;

-- Verify
SHOW VARIABLES LIKE 'slave_parallel_threads' ;       -- value : 8
SELECT COUNT(*) AS workers
FROM information_schema.PROCESSLIST
WHERE USER = 'system user' AND STATE LIKE 'Slave_worker%' ;
-- workers : 8

-- Persist across restart in my.cnf
-- [mariadb]
-- slave_parallel_threads = 8
-- slave_parallel_mode    = optimistic
```

---

## Example 4 : Batch a Large DELETE on the Master (10.6+)

```sql
-- BAD : one event of ~50 million rows ; replica applier stuck for minutes
DELETE FROM audit_log WHERE created_at < '2024-01-01' ;

-- GOOD : batched DELETE ; one small event per chunk
DELIMITER //
CREATE PROCEDURE archive_old_audit_log()
BEGIN
  DECLARE done INT DEFAULT 0 ;
  REPEAT
    DELETE FROM audit_log WHERE created_at < '2024-01-01' LIMIT 10000 ;
    SET done = (ROW_COUNT() = 0) ;
    DO SLEEP(0.1) ;                                    -- let replica catch up between batches
  UNTIL done = 1 END REPEAT ;
END //
DELIMITER ;

CALL archive_old_audit_log() ;
DROP PROCEDURE archive_old_audit_log ;
```

Or, production-grade, use `pt-archiver` (Percona Toolkit) :

```bash
pt-archiver \
  --source h=master,D=mydb,t=audit_log \
  --where "created_at < '2024-01-01'" \
  --limit=10000 --commit-each --sleep=0.1 \
  --purge \
  --no-check-charset
```

---

## Example 5 : Switch slave_parallel_mode from conservative to optimistic (10.5.1+)

```sql
-- Older replicas upgraded from 10.4 may still have mode=conservative
SHOW VARIABLES LIKE 'slave_parallel_mode' ;          -- conservative

STOP SLAVE SQL_THREAD ;
SET GLOBAL slave_parallel_mode = 'optimistic' ;
START SLAVE SQL_THREAD ;

SHOW VARIABLES LIKE 'slave_parallel_mode' ;          -- optimistic
```

The `optimistic` mode parallelises any transaction by default and retries on conflict, which is significantly faster on most workloads than `conservative` which uses group-commit information to identify safe parallelism.

---

## Example 6 : Identify a Large Binlog Event (10.6+)

```sql
-- On the master, find the current binlog
SHOW MASTER STATUS ;
-- File : mariadb-bin.000123 , Position : 78463528

-- List events in this binlog
SHOW BINLOG EVENTS IN 'mariadb-bin.000123' LIMIT 100 ;

-- Look for rows where (End_log_pos - Pos) is multiple MB.
-- Large Write_rows_v1 / Update_rows_v1 / Delete_rows_v1 events block the applier.

-- For deeper inspection, use mariadb-binlog (binary log reader, renamed from mysqlbinlog in 10.5+)
mariadb-binlog --base64-output=DECODE-ROWS --verbose \
  /var/lib/mysql/mariadb-bin.000123 | less
```

---

## Example 7 : Detect and Reset Stale gtid_slave_pos (10.6+)

```sql
-- On replica, check positions
SELECT @@gtid_slave_pos    AS slave_pos,
       @@gtid_current_pos  AS current_pos ;

-- Compare against master
-- On master :
SELECT @@gtid_binlog_pos ;
-- e.g. 0-1-678901

-- If replica gtid_slave_pos is stale or missing :
STOP SLAVE ;
SET GLOBAL gtid_slave_pos = '0-1-678901' ;       -- value from master gtid_binlog_pos
START SLAVE ;

SHOW SLAVE STATUS\G                                 -- verify no Last_SQL_Error
```

NEVER disable `gtid_strict_mode` to make `ER_GTID_STRICT_OUT_OF_ORDER` go away ; reset the position correctly.

---

## Example 8 : Reproduce a Semi-Sync Timeout Lag-Spike (10.6+)

```sql
-- On master : confirm semi-sync is enabled and check timeout
SHOW VARIABLES LIKE 'rpl_semi_sync_master_enabled' ;  -- ON
SHOW VARIABLES LIKE 'rpl_semi_sync_master_timeout' ;  -- 10000 (ms)

-- Inspect status before
SHOW GLOBAL STATUS LIKE 'Rpl_semi_sync_master_status' ;       -- ON
SHOW GLOBAL STATUS LIKE 'Rpl_semi_sync_master_no_times' ;     -- 0

-- Simulate replica unavailability (e.g. block replica IO with `tc qdisc add dev eth0 root netem delay 15000ms`)
-- After 10 seconds of stalled ack, master falls back to async :
SHOW GLOBAL STATUS LIKE 'Rpl_semi_sync_master_status' ;       -- OFF  <- fallback
SHOW GLOBAL STATUS LIKE 'Rpl_semi_sync_master_no_times' ;     -- 1

-- This shows as a lag spike on monitoring. The cure is NETWORK, not applier-thread.
```

For cross-AZ or cross-region semi-sync, raise the timeout :

```ini
# my.cnf on master
[mariadb]
rpl_semi_sync_master_timeout = 30000      # 30 seconds, accommodates higher RTT
rpl_semi_sync_master_wait_point = AFTER_SYNC
```

---

## Example 9 : Multi-Source Per-Channel Lag Monitoring (10.0+)

```sql
-- All channels at once
SHOW ALL SLAVES STATUS\G

-- Per-channel control
STOP SLAVE 'analytics' ;
SET GLOBAL slave_parallel_threads = 8 ;
START SLAVE 'analytics' ;

-- Per-domain parallel thread cap (prevent one source starving another)
SET GLOBAL slave_domain_parallel_threads = 4 ;

-- Inspect per-channel applier workers
SELECT *
FROM information_schema.PROCESSLIST
WHERE USER = 'system user'
ORDER BY DB, ID ;
```

For multi-source you SHOULD set `slave_domain_parallel_threads` to roughly `slave_parallel_threads / number_of_sources`, so each source gets a fair share of the worker pool.

---

## Example 10 : Set max_allowed_packet on Both Master and Replica (10.6+)

```sql
-- A "binlog event too large" applier failure usually means the replica's
-- max_allowed_packet is lower than what the master can send.

-- On both nodes
SHOW VARIABLES LIKE 'max_allowed_packet' ;

-- Set in my.cnf and restart, OR dynamically
SET GLOBAL max_allowed_packet = 1073741824 ;        -- 1 GB

-- Each new connection picks up the new value ; existing connections keep old value.
-- For applier-side, restart replication :
STOP SLAVE ;
START SLAVE ;
```

---

## Example 11 : Verify Applier Workers Are Actually Doing Work (10.6+)

```sql
-- After enabling slave_parallel_threads = N, the worker pool must show N rows
SELECT ID, USER, HOST, DB, COMMAND, TIME, STATE, INFO
FROM information_schema.PROCESSLIST
WHERE USER = 'system user'
ORDER BY ID ;

-- Expected states for healthy parallel applier :
--   - Slave_IO_Running                  (1 thread)
--   - Slave_SQL_Running                 (1 thread, the coordinator)
--   - Slave_worker (N threads)          (the parallel workers)

-- If you see ONLY Slave_IO and Slave_SQL with no worker rows, parallel applier is NOT actually active.
-- Common cause : slave_parallel_mode = none was left in my.cnf and overrides the dynamic SET.
```

---

## Example 12 : Compute Lag from a Custom Heartbeat Table (10.6+, when pt-heartbeat is unavailable)

```sql
-- On master, create the heartbeat table once
CREATE TABLE monitoring.heartbeat (
  id INT PRIMARY KEY,
  ts DATETIME(6) NOT NULL
) ENGINE=InnoDB ;

INSERT INTO monitoring.heartbeat (id, ts) VALUES (1, NOW(6))
ON DUPLICATE KEY UPDATE ts = NOW(6) ;

-- Schedule it to update every second (10.6+ uses EVENT)
CREATE EVENT monitoring.heartbeat_emit
  ON SCHEDULE EVERY 1 SECOND
  DO
    INSERT INTO monitoring.heartbeat (id, ts) VALUES (1, NOW(6))
    ON DUPLICATE KEY UPDATE ts = NOW(6) ;

-- Ensure the event scheduler is on
SET GLOBAL event_scheduler = ON ;

-- On replica, compute lag with sub-second resolution
SELECT TIMESTAMPDIFF(MICROSECOND, ts, NOW(6)) / 1000000.0 AS lag_seconds
FROM monitoring.heartbeat WHERE id = 1 ;
```

This pattern is good when `pt-heartbeat` cannot be installed (restricted production environment). It uses the MariaDB EVENT scheduler (verified against [KB events](https://mariadb.com/kb/en/events/)) instead of an external daemon.

---

## Example 13 : Capture the Slow-Event Backtrace with mariadb-binlog (10.6+)

```bash
# Find the binlog file and position the applier is stuck on
mariadb -h replica -e "SHOW SLAVE STATUS\G" | grep -E 'Relay_Master_Log_File|Exec_Master_Log_Pos'

# Decode and inspect events around that position
mariadb-binlog \
  --base64-output=DECODE-ROWS --verbose \
  --start-position=78400000 --stop-position=78500000 \
  /var/lib/mysql/mariadb-bin.000123 > /tmp/stuck_event.sql

# Inspect /tmp/stuck_event.sql to see the offending statement (or row image count).
```
