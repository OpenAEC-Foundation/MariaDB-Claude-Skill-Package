# Anti-Patterns : Replication Lag

Real-world mistakes drawn from MariaDB KB warnings, JIRA reports, and production incident patterns. Each anti-pattern includes the failing code, why it fails, and the correct alternative.

---

## 1. Alerting on `Seconds_Behind_Master` Alone

**Anti-pattern** :

```yaml
# Prometheus / monitoring rule
- alert: ReplicationLag
  expr: mysql_slave_status_seconds_behind_master > 60
  for: 1m
```

**Why it fails** : `Seconds_Behind_Master` only updates between binlog-event boundaries. During replay of a single large event (e.g. a 50 M-row `DELETE`), the field stays at `0` while real lag grows to minutes. The alert NEVER fires until the event finishes. Worse : during a transient network outage, the value can also stay at `0` briefly because the SQL thread has nothing new to apply.

**Verified against** : [KB SHOW SLAVE STATUS](https://mariadb.com/kb/en/show-slave-status/) which documents `Seconds_Behind_Master` as "the number of seconds the slave SQL thread is behind processing the master binary log".

**Correct alternative** : Use `pt-heartbeat` (Percona Toolkit) or a custom heartbeat table that writes a timestamp on the master every 1 second. The replica reads the replicated timestamp and computes `NOW(6) - heartbeat.ts`. This measures TRUE lag, sub-second resolution.

```bash
pt-heartbeat --daemonize --update --interval=1 \
  -h master -u monitor -p secret \
  --database heartbeat --table heartbeat
```

---

## 2. Single-Thread Applier in 2025

**Anti-pattern** :

```ini
# my.cnf left at default after a 10.6 upgrade
[mariadb]
# slave_parallel_threads not set, defaults to 0 (single-thread applier)
```

**Why it fails** : `slave_parallel_threads = 0` is the default, but it means the SQL thread applies events one at a time. On any modern OLTP workload (more than a few hundred writes per second on the master) the single thread cannot keep up and lag grows monotonically. The replica looks healthy in `SHOW SLAVE STATUS` (both threads `Yes`) but `pt-heartbeat` shows minutes of lag.

**Verified against** : [KB parallel-replication](https://mariadb.com/kb/en/parallel-replication/) : "slave_parallel_threads default 0 disables parallel replication".

**Correct alternative** : Set a non-zero value AND ensure `slave_parallel_mode` is `optimistic` (default 10.5.1+).

```sql
STOP SLAVE SQL_THREAD ;
SET GLOBAL slave_parallel_threads = 8 ;
SET GLOBAL slave_parallel_mode    = 'optimistic' ;
START SLAVE SQL_THREAD ;
```

Persist in `my.cnf`.

---

## 3. Tuning `slave_parallel_threads` Without an IO Benchmark

**Anti-pattern** :

```sql
-- "We have 32 cores so we set this to 32"
SET GLOBAL slave_parallel_threads = 32 ;
```

**Why it fails** : Parallel applier throughput is bounded by disk IO, not CPU. On a HDD replica, even `slave_parallel_threads = 4` will not run faster than the disk can fsync. On a SSD replica, the sweet spot is typically 8-16 ; above that, InnoDB internal lock contention (log-buffer mutex, redo-log mutex) REDUCES throughput. Setting `32` on hardware that cannot sustain it just burns CPU on context switches.

**Verified against** : [KB parallel-replication](https://mariadb.com/kb/en/parallel-replication/) which describes the conflict-detection cost and recommends benchmarking.

**Correct alternative** : Benchmark with `fio` first to know your replica IOPS ceiling, then bracket `slave_parallel_threads` at `4`, `8`, `16` and measure applier throughput under realistic load. Pick the lowest value that meets your lag SLA.

```bash
# Quick IO benchmark : 4K random writes on replica disk
fio --name=replica-iops --rw=randwrite --bs=4k --size=1G \
    --runtime=60 --time_based --ioengine=libaio --iodepth=32 \
    --filename=/var/lib/mysql/iotest.dat
```

---

## 4. Replica Disk Slower Than Master Disk

**Anti-pattern** : Master on NVMe, replica on spinning HDD.

**Why it fails** : The replica MUST apply every write that the master does. If the replica disk has lower write IOPS than the master, lag is mathematically unavoidable and grows linearly with time. No parallel-applier tuning can compensate.

**Verified against** : MariaDB hardware-sizing guidance in the [KB performance tuning](https://mariadb.com/kb/en/performance/) section.

**Correct alternative** : Match or exceed the master's disk on the replica. For asymmetric setups (analytics replica with relaxed lag SLA), accept the lag explicitly and document the budget.

---

## 5. Running a Large UPDATE/DELETE on the Master Without Batching

**Anti-pattern** :

```sql
DELETE FROM audit_log WHERE created_at < '2020-01-01' ;
-- removes 50 million rows in one transaction
```

**Why it fails** : Produces a single huge binlog event. The replica applier processes one event at a time (even with `slave_parallel_threads > 0`, a single event runs on one worker). Replica is stuck for minutes ; `Seconds_Behind_Master` may still report 0 because the event is not yet finished. Galera nodes will also reject the write set if `wsrep_max_ws_rows` or `wsrep_max_ws_size` is exceeded.

**Verified against** : [KB binary-log-formats](https://mariadb.com/kb/en/binary-log-formats/) discussion of row-based logging volume.

**Correct alternative** : Batch into chunks of 1000-10000 rows. Use `pt-archiver` for production-grade chunked archival.

```bash
pt-archiver \
  --source h=master,D=mydb,t=audit_log \
  --where "created_at < '2020-01-01'" \
  --limit=10000 --commit-each --sleep=0.1 \
  --purge
```

---

## 6. STATEMENT-based Binlog with Non-Deterministic Functions

**Anti-pattern** :

```sql
-- Replica has binlog_format=STATEMENT (manually set, overriding MIXED default)
INSERT INTO event_log (id, ts, token)
  VALUES (NULL, NOW(6), UUID()) ;
```

**Why it fails** : With `binlog_format=STATEMENT`, the replica re-runs the statement. `NOW(6)` resolves to a different timestamp on the replica ; `UUID()` is a completely different value. The KB explicitly warns "the set of rows included cannot be predicted" for `LIMIT` without `ORDER BY`, and the same logic applies to non-deterministic functions. Result : silent data divergence between master and replica. Lag does not grow but the data is wrong, which is worse.

**Verified against** : [KB binary-log-formats](https://mariadb.com/kb/en/binary-log-formats/) : "MariaDB switches to row-based encoding for any statement the server determines is unsafe (non-deterministic functions, LIMIT without ORDER BY, certain stored procedures)".

**Correct alternative** : Use `MIXED` (default 10.2.3+) or `ROW`. `MIXED` switches per-statement, `ROW` is uniformly safe.

```ini
# my.cnf
[mariadb]
binlog_format = MIXED          # or ROW for stricter determinism
```

---

## 7. `binlog_row_image=MINIMAL` on Production with PITR Needs

**Anti-pattern** :

```ini
[mariadb]
binlog_row_image = MINIMAL     # "saves disk space"
```

**Why it fails** : `MINIMAL` records only the primary key in the before-image and only changed columns in the after-image. This BREAKS :

- Point-in-time recovery to a specific row state : you cannot reconstruct the full row.
- Fix-on-replica scripts that need to detect what changed (e.g. forensic audit, change-data-capture).
- Galera write-set merge on conflict (certification needs the full before-image).

The disk-space savings are usually 30-40%, but the operational cost on a real incident is unbounded.

**Verified against** : [KB replication-and-binary-log-system-variables](https://mariadb.com/kb/en/replication-and-binary-log-system-variables/) variable description : "MINIMAL : PK in before-image, changed columns in after-image".

**Correct alternative** : Leave `binlog_row_image = FULL` (default). If binlog size is a real bottleneck, use `NOBLOB` (excludes only unchanged BLOB/TEXT) or `FULL_NODUP` (11.4+, removes duplicated columns between before and after image).

---

## 8. Ignoring `Rpl_semi_sync_master_status` Changes

**Anti-pattern** : Dashboard tracks `Seconds_Behind_Master` and applier worker count, but NOT `Rpl_semi_sync_master_status`.

**Why it fails** : When semi-sync replication is enabled and the network has a transient hiccup, the master waits up to `rpl_semi_sync_master_timeout` (default 10000 ms) for the replica ack and then FALLS BACK to async. The graph shows a brief lag spike but the real symptom is loss of the semi-sync durability guarantee. The next master failure between the fallback and the next reconnect could lose committed writes that the operator believed were replicated.

**Verified against** : [KB semisynchronous-replication](https://mariadb.com/kb/en/semisynchronous-replication/) : `Rpl_semi_sync_master_status` "indicates whether semi-sync replication is currently active (ON) or has fallen back to async (OFF)".

**Correct alternative** : Alert on `Rpl_semi_sync_master_status = OFF` AND on `Rpl_semi_sync_master_no_times` increases. Investigate network RTT and raise `rpl_semi_sync_master_timeout` for cross-AZ replicas.

```sql
SHOW GLOBAL STATUS LIKE 'Rpl_semi_sync_master_status' ;     -- should stay ON
SHOW GLOBAL STATUS LIKE 'Rpl_semi_sync_master_no_times' ;   -- should not increase
```

```ini
# my.cnf for cross-AZ
[mariadb]
rpl_semi_sync_master_timeout    = 30000        # 30 s
rpl_semi_sync_master_wait_point = AFTER_SYNC   # stronger durability
```

---

## 9. Disabling `gtid_strict_mode` to Silence Errors

**Anti-pattern** :

```sql
-- Replica shows ER_GTID_STRICT_OUT_OF_ORDER (error 1962)
SET GLOBAL gtid_strict_mode = OFF ;
START SLAVE ;
-- "It works now."
```

**Why it fails** : `gtid_strict_mode` exists specifically to refuse events that would create binlog divergence. The error you saw means the replica's `gtid_slave_pos` is out of sequence relative to the master's `gtid_binlog_pos`. Turning the check off does NOT fix the position ; it just lets the replica silently accept the inconsistent event chain. Future failover from this replica will produce an inconsistent master.

**Verified against** : [KB gtid](https://mariadb.com/kb/en/gtid/) variable description of `gtid_strict_mode`.

**Correct alternative** : Reset `gtid_slave_pos` to the correct value derived from the master :

```sql
-- On master
SELECT @@gtid_binlog_pos ;       -- e.g. '0-1-678901'

-- On replica
STOP SLAVE ;
SET GLOBAL gtid_slave_pos = '0-1-678901' ;
START SLAVE ;
SHOW SLAVE STATUS\G              -- verify no Last_SQL_Error
```

Keep `gtid_strict_mode = ON` so future divergence fails fast.

---

## 10. `SHOW SLAVE STATUS` on a Multi-Source Setup

**Anti-pattern** :

```sql
-- Multi-source replica with 3 channels
SHOW SLAVE STATUS\G
```

**Why it fails** : Plain `SHOW SLAVE STATUS` returns ONLY the default-channel row. The other two channels could be hours behind and the operator would see nothing. Worse, scripts that grep `Seconds_Behind_Master` from this command will report a single value that does not represent any of the actual channels.

**Verified against** : [KB multi-source-replication](https://mariadb.com/kb/en/multi-source-replication/).

**Correct alternative** : Always use `SHOW ALL SLAVES STATUS` on multi-source setups (10.0+). It returns one row per channel.

```sql
SHOW ALL SLAVES STATUS\G          -- one row per replication channel
```

For per-channel control :

```sql
STOP SLAVE 'analytics' SQL_THREAD ;
SET GLOBAL slave_parallel_threads = 8 ;
START SLAVE 'analytics' SQL_THREAD ;
```

Also set `slave_domain_parallel_threads` to roughly `threads / number_of_sources` so one source cannot starve another.
