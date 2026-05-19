# anti-patterns.md : Replication Failure Modes

Real-world failure modes for MariaDB replication, each with code that fails, root cause, and the correct alternative.

All anti-patterns verified against MariaDB KB and JIRA on 2026-05-19.

---

## AP-01 : Using MySQL `FOR CHANNEL` Syntax on MariaDB

```sql
-- WRONG (MySQL syntax) : parser error on MariaDB
CHANGE MASTER TO MASTER_HOST = 'analytics' FOR CHANNEL 'analytics';
START SLAVE FOR CHANNEL 'analytics';
```

**Why it fails** : MariaDB multi-source replication uses NAMED CONNECTIONS, not `FOR CHANNEL` clauses. MySQL borrowed multi-source replication years after MariaDB and chose a different syntax. The MariaDB parser rejects `FOR CHANNEL` with a generic `You have an error in your SQL syntax`, which leads users to assume a typo rather than a fundamental syntax difference.

**Correct alternative** :

```sql
-- MariaDB NAMED CONNECTION syntax : quotes around channel name
CHANGE MASTER 'analytics' TO
  MASTER_HOST     = 'analytics-primary.example.com',
  MASTER_USER     = 'repl',
  MASTER_PASSWORD = 'CHANGEME',
  MASTER_USE_GTID = slave_pos;
START SLAVE 'analytics';
SHOW ALL SLAVES STATUS \G
```

Source : https://mariadb.com/kb/en/multi-source-replication/

---

## AP-02 : Expecting MariaDB-to-MySQL Continuous GTID-Tracked Replication

```sql
-- WRONG : MySQL replica configured to follow MariaDB primary using GTID
-- On MySQL replica :
CHANGE MASTER TO
  MASTER_HOST           = 'mariadb-primary',
  MASTER_AUTO_POSITION  = 1;                  -- MySQL GTID auto-position
START SLAVE;                                  -- breaks immediately
```

**Why it fails** : MariaDB GTID format is `domain-server-sequence` (e.g. `0-1-1234`). MySQL GTID format is `uuid:seqno` (e.g. `3E11FA47-71CA-11E1-9E33-C80AA9429562:23`). The formats are not interconvertible at the protocol level. Per MariaDB KB : "MariaDB can be a replica for a MySQL primary, but MySQL cannot be a replica for a MariaDB primary." Even positional replication MariaDB-to-MySQL fails because MySQL refuses MariaDB binlog event types it does not recognize.

**Correct alternative** : MariaDB-to-MySQL migration is a ONE-WAY dump-and-load cutover :

```bash
# on MariaDB source
mariadb-dump --all-databases --single-transaction --master-data=2 > dump.sql

# transfer dump.sql to MySQL target, then on MySQL :
mysql < dump.sql
# point applications at the new MySQL host, retire the MariaDB source
```

For MySQL-to-MariaDB (the supported direction) use positional replication. From MariaDB 11.4.5+ MySQL 8.0 source compatibility is improved.

Source : https://mariadb.com/kb/en/gtid/ (L-004 in LESSONS.md)

---

## AP-03 : `STATEMENT` Binlog with Non-Deterministic Functions

```sql
-- WRONG : binlog_format = STATEMENT + non-deterministic SQL
SET GLOBAL binlog_format = STATEMENT;

INSERT INTO audit_log (id, occurred_at, token)
VALUES (NULL, NOW(6), UUID());
-- replication drift : replica generates DIFFERENT UUID() and microsecond NOW(6) values
```

**Why it fails** : `STATEMENT` binlog replays the SQL text on the replica. Non-deterministic functions (`UUID()`, `NOW(6)` at high resolution, `RAND()`, `SLEEP()`, `LIMIT` without `ORDER BY`, user-defined functions, certain procedure calls) produce different results on the replica, silently diverging the data. The KB warns "the set of rows included cannot be predicted" for unsafe statements.

**Correct alternative** : use `MIXED` (default since 10.2.4) which automatically switches to row-based encoding for unsafe statements, or use `ROW` unconditionally :

```ini
[mariadb]
binlog_format = MIXED                 # default 10.2.4+
# or
binlog_format = ROW                   # always safe, larger binlog volume
binlog_row_image = MINIMAL            # reduce row-image size if disk pressure
```

Source : https://mariadb.com/kb/en/binary-log-formats/

---

## AP-04 : `slave_parallel_threads` Above 16 Without Disk-IO Benchmark

```ini
# WRONG : sky-high parallelism on a single-disk replica
[mariadb]
slave_parallel_threads = 64
slave_parallel_mode    = optimistic
```

**Why it fails** : parallel applier threads compete for replica disk I/O and InnoDB row-lock latches. On any storage that is not NVMe or a tuned RAID array, going above 8 threads typically REDUCES throughput due to contention. The KB recommends "at least two times the number of multi-source primary connections used" as a floor, NOT a ceiling. Setting 64 threads with no benchmark just hides the real bottleneck (I/O) behind a false fix (more workers).

**Correct alternative** : start at 4, measure, raise incrementally :

```sql
STOP ALL SLAVES;
SET GLOBAL slave_parallel_threads = 4;
SET GLOBAL slave_parallel_mode    = optimistic;
START ALL SLAVES;

-- measure for 24 hours; key metrics :
-- Seconds_Behind_Master trend, replica's iostat %util, innodb_row_lock_waits
-- only raise threads if Seconds_Behind_Master is consistently > 0 AND disk %util < 80%
```

Source : https://mariadb.com/kb/en/parallel-replication/

---

## AP-05 : Semi-Sync Enabled Without Understanding Async Fallback

```sql
-- WRONG : assumed semi-sync guarantees zero data loss; no monitoring
SET GLOBAL rpl_semi_sync_master_enabled = ON;
SET GLOBAL rpl_semi_sync_master_timeout = 1000;     -- 1 second, very aggressive
-- ... time passes, replica becomes slow ...
-- application sees occasional 1-second commit latency spikes (the timeout)
-- after which the primary silently runs in async mode
-- on primary crash : up to one full transaction lost despite "semi-sync"
```

**Why it fails** : `rpl_semi_sync_master_timeout` (default 10000 ms) controls how long the primary waits for at least one replica's ACK before falling back to async. On timeout the primary does NOT block forever: it commits without ACK and logs a warning. Apparent commit-latency spikes under load are this timeout firing; subsequent commits then run in pure async mode until a replica catches up. Without monitoring `Rpl_semi_sync_master_status` you cannot tell whether semi-sync is actually engaged.

**Correct alternative** : tune timeout for your real network RTT and monitor :

```sql
-- timeout = generous multiple of typical network RTT + replica apply time
SET GLOBAL rpl_semi_sync_master_timeout         = 10000;   -- default, fine for most LANs
SET GLOBAL rpl_semi_sync_master_wait_for_slave_count = 1;
SET GLOBAL rpl_semi_sync_master_wait_point      = AFTER_SYNC;

-- monitoring (run on schedule)
SHOW STATUS LIKE 'Rpl_semi_sync_master_status';      -- alert if OFF
SHOW STATUS LIKE 'Rpl_semi_sync_master_no_tx';       -- alert on increment rate
SHOW STATUS LIKE 'Rpl_semi_sync_master_tx_avg_wait_time';
```

For applications needing strict zero-loss semantics, semi-sync alone is INSUFFICIENT. Layer with Galera (synchronous multi-master) or with `rpl_semi_sync_master_wait_for_slave_count >= 2` and multi-AZ replicas.

Source : https://mariadb.com/kb/en/semisynchronous-replication/

---

## AP-06 : `slave_parallel_mode = aggressive` Without Testing

```ini
# WRONG : aggressive mode in production without staging benchmark
[mariadb]
slave_parallel_threads = 16
slave_parallel_mode    = aggressive
```

**Why it fails** : `aggressive` mode runs transactions in parallel without conflict-avoidance heuristics. On workloads with row-level write contention (hot row updates, INSERT-into-same-PK patterns) it triggers extensive optimistic-retry storms, sometimes making throughput WORSE than single-threaded `none`. The KB describes it as "similar to optimistic but disables heuristics for conflict avoidance" : it is a tuning knob for specific contention-free workloads, not a default.

**Correct alternative** : stick with `optimistic` (default 10.5.1+) which retries on conflict but uses heuristics to reduce wasted work. Only consider `aggressive` after benchmarking against `optimistic` on the SAME data and write pattern :

```ini
[mariadb]
slave_parallel_threads = 8
slave_parallel_mode    = optimistic   # default, well-tested
```

Source : https://mariadb.com/kb/en/parallel-replication/

---

## AP-07 : `gtid_strict_mode = OFF` Masking Binlog Divergence

```sql
-- WRONG : strict mode left at default OFF
SHOW GLOBAL VARIABLES LIKE 'gtid_strict_mode';    -- OFF
-- replica accidentally applies an event with a lower sequence than already present
-- no error raised; binlog now diverges silently between primary and replica
-- weeks later, failover promotes replica; data inconsistency emerges
```

**Why it fails** : `gtid_strict_mode = OFF` allows the server to apply GTIDs that would otherwise be rejected as inconsistent (lower-sequence-after-higher, duplicate sequence in same domain, etc). On a healthy chain these never occur; when they DO occur it signals operator error (manual binlog manipulation, partial restore, multi-source loop) and strict mode would have caught it at apply time. Leaving strict mode off turns this safety net off.

**Correct alternative** : enable strict mode on every node :

```sql
-- on every primary and replica
SET GLOBAL gtid_strict_mode = ON;

-- persist in my.cnf
-- [mariadb]
-- gtid_strict_mode = ON
```

If strict mode rejects an event with `Error_code: 1942` (Out-of-order GTID), STOP and investigate the source of the divergence before forcing past it.

Source : https://mariadb.com/kb/en/gtid/

---

## AP-08 : Forgetting `MASTER_USE_GTID = slave_pos` on Intermediate Replicas in a Chain

```sql
-- WRONG : chained replication A -> B -> C; B is also accepting local writes
-- on C, using current_pos (or worse, no GTID at all)
CHANGE MASTER TO
  MASTER_HOST     = 'B',
  MASTER_USE_GTID = current_pos;
-- after failover where B is taken out and C is repointed at A directly :
-- C's position includes B's LOCAL writes, which do not exist on A
-- CHANGE MASTER TO new primary fails or replicates stale data
```

**Why it fails** : `current_pos` includes both replicated GTIDs AND locally-written GTIDs in `gtid_current_pos`. On an intermediate node B that takes its own writes, those local writes get tracked too. When you later repoint C at A (skipping B), C's GTID position references events that A has never seen, leading to "got fatal error 1236" or silent drift.

**Correct alternative** : on any pure-replica node (no local writes), use `slave_pos` so position tracks ONLY replicated GTIDs :

```sql
-- on C (pure replica)
CHANGE MASTER TO MASTER_USE_GTID = slave_pos;

-- and enforce no local writes
SET GLOBAL read_only        = ON;
SET GLOBAL super_read_only  = ON;            -- 10.11+, blocks SUPER users too
```

Use `current_pos` only on a node that intentionally accepts writes and may be promoted to primary later.

Source : https://mariadb.com/kb/en/gtid/

---

## AP-09 : `MASTER_SSL = 1` Without `MASTER_SSL_VERIFY_SERVER_CERT = 1`

```sql
-- WRONG : encrypted but unverified
CHANGE MASTER TO
  MASTER_SSL                    = 1,
  MASTER_SSL_CA                 = '/etc/mysql/certs/ca.pem';
-- MASTER_SSL_VERIFY_SERVER_CERT defaults to OFF : connection encrypted but accepts ANY valid cert
```

**Why it fails** : without `MASTER_SSL_VERIFY_SERVER_CERT = 1`, the replica accepts any X.509 certificate the primary presents, even one signed by a CA the replica does not trust. A MITM attacker who can intercept the TCP connection can present a self-signed or unrelated valid certificate; the replica negotiates a TLS session and streams binlog into the attacker's pipe.

**Correct alternative** :

```sql
STOP SLAVE;
CHANGE MASTER TO
  MASTER_SSL                    = 1,
  MASTER_SSL_CA                 = '/etc/mysql/certs/ca.pem',
  MASTER_SSL_CERT               = '/etc/mysql/certs/replica-cert.pem',
  MASTER_SSL_KEY                = '/etc/mysql/certs/replica-key.pem',
  MASTER_SSL_VERIFY_SERVER_CERT = 1;
START SLAVE;
```

ALWAYS set verify. From MariaDB 12.3+ each TLS option also accepts `DEFAULT` to inherit server-level TLS config, reducing duplication.

Source : https://mariadb.com/kb/en/change-master-to/

---

## AP-10 : Duplicate `server_id` on Two Nodes

```ini
# WRONG : copy-pasted my.cnf from primary to replica without changing server_id
[mariadb]
server_id = 1                     # same value on BOTH primary and replica
log_bin   = mariadb-bin
```

**Why it fails** : `server_id` must be UNIQUE across the entire replication topology. Two nodes sharing the same id cause the primary to disconnect the duplicate replica with `error 1593 (HY000): Fatal error: The slave I/O thread stops because master and slave have equal MariaDB server ids`. In multi-source or chained setups, a duplicate id can also cause GTID domain conflicts and apparent "missing event" errors.

**Correct alternative** : assign a unique 1..2^32-1 value per node and persist in my.cnf :

```ini
# PRIMARY
server_id = 1

# REPLICA 1
server_id = 2

# REPLICA 2
server_id = 3
```

A common convention is to encode datacenter and node number (`server_id = <dc_int><node_int>`, e.g. `101`, `102`, `201`, `202`).

Source : https://mariadb.com/kb/en/setting-up-replication/
