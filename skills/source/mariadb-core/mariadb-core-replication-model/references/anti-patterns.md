# anti-patterns.md : Replication Anti-Patterns

Field-tested anti-patterns. Each entry : code, why it fails, correct alternative.

---

## AP-01 : STATEMENT binlog with non-deterministic functions

**Wrong** :

```sql
-- on primary, with binlog_format = STATEMENT
INSERT INTO orders (order_id, created_at) VALUES (UUID(), NOW());
INSERT INTO logs   (id, msg, rnd) VALUES (NULL, 'event', RAND());
```

**Why it fails** : `UUID()`, `NOW()` in some contexts, and `RAND()` evaluate to DIFFERENT values on the primary vs each replica. STATEMENT replication ships the SQL text, the replica re-executes, and gets different rows. Silent data drift accumulates ; replication can stay `Slave_SQL_Running = Yes` while diverging.

**Right** :

```ini
# my.cnf : use MIXED (default since 10.2.4) or ROW
[mariadb]
binlog_format = MIXED
```

MIXED auto-switches to row-encoding for unsafe statements. ROW is the safest option (mandatory for Galera). NEVER ship STATEMENT to production in 10.x.

---

## AP-02 : Semi-sync without understanding the timeout fallback

**Wrong** :

```ini
# my.cnf
rpl_semi_sync_master_enabled = ON
rpl_semi_sync_master_timeout = 30000   # 30 seconds, "to be safe"
```

Operators assume "semi-sync = always synchronous, commits will hang if replica is down".

**Why it fails** : On timeout the primary REVERTS TO ASYNC. After 30 seconds of no ack, every subsequent commit returns immediately without waiting. The next time a replica acks, semi-sync resumes silently. Operators monitoring only `rpl_semi_sync_master_enabled=ON` think they have synchronous guarantees, but `Rpl_semi_sync_master_status` was OFF for 12 minutes during a network glitch and they lost data.

**Right** :

```sql
-- alert on status, not just on configured value
SHOW STATUS LIKE 'Rpl_semi_sync_master_status';   -- alert if not ON
SHOW STATUS LIKE 'Rpl_semi_sync_master_no_tx';    -- alert on increase
```

Tune `rpl_semi_sync_master_timeout` deliberately : too low and any network blip flips to async ; too high and clients see commit latency spikes. ALWAYS pair semi-sync with explicit monitoring of `Rpl_semi_sync_master_status`.

---

## AP-03 : slave_parallel_threads cranked far above disk IOPS

**Wrong** :

```ini
# replica my.cnf, on a 4-core VM with single SATA SSD
slave_parallel_threads = 64
slave_parallel_mode    = optimistic
```

**Why it fails** : More workers means more concurrent fsync requests. Once the disk subsystem saturates (one SATA SSD : ~20-30k IOPS), additional workers wait in the I/O queue. Apply throughput PEAKS at 8-16 threads on this hardware and DROPS beyond that as context-switching dominates. Optimistic mode adds retry storms when workers conflict on hot rows, multiplying the wasted I/O.

**Right** :

```ini
slave_parallel_threads = 8       # benchmark up from here
slave_parallel_mode    = optimistic
```

Benchmark : start at `slave_parallel_threads = 4`, double, measure apply rate on real workload, stop when adding more threads stops increasing apply rate. ALWAYS pair the change with `iostat -x 1` to confirm disk is the bottleneck. NEVER set `> 16` without a measured reason.

---

## AP-04 : Multi-source replication with overlapping auto_increment ranges

**Wrong** :

```sql
-- both primaries write to the same logical table 'events'
-- primary A : default auto_increment_increment=1, auto_increment_offset=1
-- primary B : default auto_increment_increment=1, auto_increment_offset=1

-- replica receives from both via multi-source
CHANGE MASTER 'a' TO MASTER_HOST='primary-a' ...;
CHANGE MASTER 'b' TO MASTER_HOST='primary-b' ...;
```

**Why it fails** : Both primaries hand out the same `events.id` sequence (1, 2, 3, ...). When the replica applies events from both, PK collisions are guaranteed. `Slave_SQL_Running` flips to `No` with `ER_DUP_ENTRY`, replication stops, and recovery requires manual gap-fill.

**Right** :

```ini
# primary A
[mariadb]
auto_increment_increment = 2
auto_increment_offset    = 1   # generates 1, 3, 5, 7, ...

# primary B
[mariadb]
auto_increment_increment = 2
auto_increment_offset    = 2   # generates 2, 4, 6, 8, ...
```

ALWAYS plan auto-increment ranges before configuring multi-source. With N writers, set `auto_increment_increment = N` on every primary and a distinct `auto_increment_offset` (1..N) per primary. Better still : use `BIGINT` PKs from a centralized sequence service or UUIDv7-style ULIDs and skip the collision class entirely.

---

## AP-05 : Continuous MariaDB-to-MySQL replication with GTID

**Wrong** :

```sql
-- on a MySQL 8.0 replica, pointing at a MariaDB primary
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST = 'mariadb-primary.example.com',
  SOURCE_USER = 'repl',
  SOURCE_AUTO_POSITION = 1;     -- MySQL GTID auto-position
```

**Why it fails** : Per the MariaDB KB, "MariaDB can be a replica for a MySQL primary, but MySQL cannot be a replica for a MariaDB primary." MariaDB GTID is `domain-server-sequence` ; MySQL expects `uuid:seqno`. The formats are NOT interconvertible. The MySQL replica rejects the binlog format mismatch and never starts.

**Right** : There is NO continuous GTID-tracked path MariaDB-primary -> MySQL-replica. Use one-shot dump-and-load :

```bash
# on MariaDB primary
mariadb-dump --master-data=2 --single-transaction --routines --events --triggers \
  --all-databases > maria.sql

# on MySQL target
mysql < maria.sql
# then set up native MySQL replication FROM ANOTHER MYSQL SERVER, not from MariaDB
```

For MySQL -> MariaDB continuous replication, this DOES work. Use `MASTER_USE_GTID = no` (binlog position) on the MariaDB replica side since the GTID formats differ.

---

## AP-06 : slave_parallel_mode=aggressive in production

**Wrong** :

```sql
SET GLOBAL slave_parallel_mode = 'aggressive';   -- "for speed"
START SLAVE;
```

**Why it fails** : `aggressive` mode disables most conflict-avoidance heuristics. Transactions that have implicit dependencies (FK chains, triggers, multi-table updates) may apply out of order. The replica's state diverges from the primary silently. The replica reports `Slave_SQL_Running = Yes` while serving inconsistent data ; only a full row-by-row checksum against the primary will catch it.

**Right** :

```sql
SET GLOBAL slave_parallel_mode = 'optimistic';   -- default since 10.5.1
-- OR
SET GLOBAL slave_parallel_mode = 'conservative'; -- safest, ordering-preserving
```

NEVER use `aggressive` in production without :
1. Exhaustive testing on a clone of production data + workload.
2. Daily checksum jobs (`pt-table-checksum`) comparing primary vs replica.
3. A rollback plan and known-good backup checkpoint.

For 99% of workloads `optimistic` is the right choice. If you see retry storms, drop to `conservative`, not up to `aggressive`.

---

## AP-07 : Alerting only on `Seconds_Behind_Master`

**Wrong** :

```bash
# alert if Seconds_Behind_Master > 60
mariadb -e "SHOW SLAVE STATUS\G" | grep Seconds_Behind_Master
```

**Why it fails** : `Seconds_Behind_Master` measures the timestamp gap between the event the SQL thread is APPLYING and the replica's wall clock. Between transactions it drops to `0`. It also returns `NULL` when the IO thread is down (the replica is not falling behind, it is DISCONNECTED, far worse). Periods of "0 seconds behind" can coexist with hours of buffered events the IO thread has fetched but the SQL thread has not yet applied. Operators see green dashboards while replication is actually broken.

**Right** : Use a heartbeat table managed by `pt-heartbeat` (Percona Toolkit) or an equivalent token-based heartbeat :

```sql
-- on replica, computed against a heartbeat row updated every second on primary
SELECT TIMESTAMPDIFF(MICROSECOND, ts, NOW(6)) / 1000000.0 AS lag_seconds
FROM   ops.heartbeat
WHERE  id = 1;
```

ALWAYS additionally alert on `Slave_IO_Running != Yes`, `Slave_SQL_Running != Yes`, and `Last_IO_Errno != 0` / `Last_SQL_Errno != 0`. `Seconds_Behind_Master` is at best a complementary signal.

---

## AP-08 : binlog_row_image=MINIMAL with point-in-time recovery requirement

**Wrong** :

```ini
# primary my.cnf, "to save space"
[mariadb]
binlog_format    = ROW
binlog_row_image = MINIMAL
```

**Why it fails** : `MINIMAL` writes only the PK in the before-image and only changed columns in the after-image. Sufficient for downstream replication, INSUFFICIENT for PITR with full row reconstruction. Auditors, CDC pipelines, and `mariadb-binlog` replay tools that need to reproduce the exact pre-change row state cannot do so. After a logical-corruption incident, you have backups but cannot replay binlogs to reconstruct individual rows ; the PITR procedure fails halfway through and you fall back to a much older full backup.

**Right** :

```ini
[mariadb]
binlog_format    = ROW
binlog_row_image = FULL   # default, keep it
```

Or for size optimization without losing PITR :

```ini
binlog_row_image = NOBLOB     # full image minus unchanged BLOB/TEXT
# OR on 11.4+
binlog_row_image = FULL_NODUP # full image with duplicate-column dedup
```

ALWAYS keep `FULL` (or `NOBLOB` / `FULL_NODUP`) on any primary that needs PITR. NEVER set `MINIMAL` unless you have explicitly accepted the loss of PITR row-reconstruction.

---

## AP-09 : Using FOR CHANNEL syntax for multi-source

**Wrong** :

```sql
-- carrying over MySQL 5.7+ habit
CHANGE MASTER TO MASTER_HOST = 'primary-a' ... FOR CHANNEL 'analytics';
START SLAVE FOR CHANNEL 'analytics';
```

**Why it fails** : `FOR CHANNEL` is MySQL 5.7+ syntax. MariaDB uses NAMED CONNECTIONS via a leading quoted identifier. The MariaDB parser rejects `FOR CHANNEL` ; the command fails with a syntax error and no replication is configured.

**Right** :

```sql
CHANGE MASTER 'analytics' TO MASTER_HOST = 'primary-a' ...;
START SLAVE 'analytics';
SHOW SLAVE 'analytics' STATUS\G
SHOW ALL SLAVES STATUS\G
```

Connection names are case-insensitive. The default unnamed connection uses an empty string `''`. NEVER mix MySQL channel syntax into MariaDB SQL.

---

## AP-10 : Forgetting log_slave_updates on intermediate replicas

**Wrong** :

```ini
# B is a replica of A and a primary for C (chain : A -> B -> C)
# B's my.cnf :
[mariadb]
server_id = 2
log_bin   = mariadb-bin
# log_slave_updates absent (defaults to OFF)
```

**Why it fails** : Without `log_slave_updates = ON`, server B applies events from A but does NOT write them to its own binlog. C is configured to replicate from B, but B's binlog only contains B's own local writes (typically none on a pure replica). C falls behind permanently, showing `Slave_IO_Running = Yes` and `Seconds_Behind_Master = 0` while not actually receiving any data from upstream.

**Right** :

```ini
# B's my.cnf
[mariadb]
server_id         = 2
log_bin           = mariadb-bin
log_slave_updates = ON          # critical for chains and for downstream async from Galera
```

ALWAYS enable `log_slave_updates` on any server that has downstream replicas. Required on intermediate replicas in chains and on Galera nodes that feed async replicas downstream.

---

## AP-11 : Disabling gtid_strict_mode "to avoid errors"

**Wrong** :

```sql
SET GLOBAL gtid_strict_mode = OFF;   -- "the replica keeps stopping with GTID errors"
START SLAVE;
```

**Why it fails** : `gtid_strict_mode = OFF` masks divergence. The replica continues applying events even when GTID sequences are out of order or missing on the primary. Logical corruption accumulates ; the replica's state silently drifts from the primary's. The original "errors" the operator suppressed were the SYSTEM TELLING THEM about real problems (manual binlog edits, split-brain, position rollback).

**Right** :

```sql
SET GLOBAL gtid_strict_mode = ON;
-- then ACTUALLY DIAGNOSE the original errors :
SHOW SLAVE STATUS\G
-- Last_SQL_Error tells you which GTID was rejected and why
SELECT @@global.gtid_slave_pos, @@global.gtid_binlog_pos;
-- compare to primary
```

ALWAYS keep `gtid_strict_mode = ON` on long-lived replicas. If the strict-mode rejection is correct (which it usually is), fix the underlying divergence (rebuild the replica from a fresh backup or use mariabackup to restore a consistent state). NEVER silence the diagnostic by disabling strict mode.

---

## AP-12 : Mixing `MASTER_USE_GTID = slave_pos` with intermediate replicas

**Wrong** :

```sql
-- on server B in chain A -> B -> C, configure B's replication from A
CHANGE MASTER TO
  MASTER_HOST     = 'A',
  MASTER_USE_GTID = slave_pos;   -- B is also a primary for C, has own binlog
```

**Why it fails** : `slave_pos` uses `gtid_slave_pos`, which tracks only events the SQL thread has APPLIED. On a server that also writes its own binlog (which B does because it has downstream replicas), `gtid_binlog_pos` may be ahead of `gtid_slave_pos`. After a failover, B might be repointed using `slave_pos` and skip events already in its own binlog, creating divergence with C.

**Right** :

```sql
-- on intermediate replica
CHANGE MASTER TO
  MASTER_HOST     = 'A',
  MASTER_USE_GTID = current_pos;   -- max of slave_pos and binlog_pos per domain
```

ALWAYS use `MASTER_USE_GTID = current_pos` on any server that has its own binlog writes (intermediate replicas, eventual primary candidates). Use `slave_pos` only on pure read-only replicas with `log_slave_updates = OFF`.

---

## Sources

All anti-patterns derived from operational experience cross-referenced against :

- MariaDB KB : Semi-synchronous replication. https://mariadb.com/kb/en/semisynchronous-replication/
- MariaDB KB : Parallel replication. https://mariadb.com/kb/en/parallel-replication/
- MariaDB KB : GTID. https://mariadb.com/kb/en/gtid/
- MariaDB KB : Binary log formats. https://mariadb.com/kb/en/binary-log-formats/
- MariaDB KB : Replication and binary log system variables. https://mariadb.com/kb/en/replication-and-binary-log-system-variables/
- MariaDB KB : Multi-source replication. https://mariadb.com/kb/en/multi-source-replication/
- MariaDB KB : SHOW SLAVE STATUS. https://mariadb.com/kb/en/show-slave-status/
