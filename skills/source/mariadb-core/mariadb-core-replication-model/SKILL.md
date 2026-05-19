---
name: mariadb-core-replication-model
description: >
  Use when designing a MariaDB replication topology, choosing between async, semi-sync, or parallel-applier, configuring GTID, migrating replication from MySQL, or debugging replication lag at the model level.
  Prevents the common mistake of mixing MariaDB GTID with MySQL GTID, choosing STATEMENT binlog with non-deterministic functions, or running parallel-applier without understanding conflict semantics.
  Covers asynchronous replication (default), semi-sync built-in 10.3+, parallel-applier with slave_parallel_mode optimistic/conservative/aggressive/minimal/none, MariaDB GTID format domain-server-sequence, binlog formats ROW/STATEMENT/MIXED, multi-source named-connection replication.
  Keywords: mariadb replication, master slave, primary replica, async replication, semi-sync, semi-synchronous, parallel applier, slave_parallel_threads, slave_parallel_mode, GTID, domain-server-sequence, gtid_strict_mode, binlog format, ROW STATEMENT MIXED, binlog_row_image, NOBLOB, multi-source, replication lag, Seconds_Behind_Master, SHOW ALL SLAVES STATUS, can mariadb replicate to mysql, how do I set up replication, replica is falling behind, replication is broken, semi-sync hang.
license: MIT
compatibility: "Designed for Claude Code. Requires MariaDB 10.6-LTS, 10.11-LTS, 11.x, 12.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# mariadb-core-replication-model

Architectural reference for MariaDB replication. Covers the three replication shapes, the MariaDB-specific GTID format, binary log formats, the parallel-applier model, and multi-source replication. Topology setup details and recovery procedures live in `references/`.

## Quick Reference

- **Async is the default**. Primary writes the binary log, replica I/O thread fetches events into the relay log, replica SQL thread (or parallel workers) applies them. The primary does NOT wait for replicas. RPO depends on replication lag.
- **Semi-sync is built into the server from MariaDB 10.3+**. NEVER install a separate plug-in on 10.3 or later. Enable via `rpl_semi_sync_master_enabled=ON` on the primary plus `rpl_semi_sync_slave_enabled=ON` on the replica. Default `rpl_semi_sync_master_timeout` is 10000 ms ; on timeout the primary reverts to async until a replica re-ack.
- **Parallel-applier default mode is `optimistic` since MariaDB 10.5.1+**. Older 10.5.0 and earlier used `conservative`. Set worker pool with `slave_parallel_threads`. Both primary and replica MUST be MariaDB 10.0.5 or later.
- **MariaDB GTID format is `domain-server-sequence`** (for example `0-1-1234`). MySQL GTID format is `uuid:seqno`. The schemes are NOT interconvertible. Per the KB : "MariaDB can be a replica for a MySQL primary, but MySQL cannot be a replica for a MariaDB primary." Migration between the two with GTID is a one-shot dump-and-load, NEVER continuous GTID-tracked replication.
- **Binary log default format is `MIXED` since MariaDB 10.2.4** (was `STATEMENT` until 10.2.3). MIXED auto-switches to row-encoding when a statement is unsafe (non-deterministic functions, `LIMIT` without `ORDER BY`, certain stored routines). Use `ROW` explicitly for Galera and for any cluster that cannot tolerate replication drift.
- **`binlog_row_image` default is `FULL`**. Other values : `MINIMAL`, `NOBLOB` ; MariaDB 11.4+ adds `FULL_NODUP`. NEVER set `MINIMAL` in environments that need point-in-time recovery with full row reconstruction.
- **Multi-source replication uses NAMED CONNECTIONS, not `FOR CHANNEL`**. Syntax is `CHANGE MASTER 'connection_name' TO ...`. The `FOR CHANNEL` syntax is MySQL-only. Connection names are case-insensitive. Use `SHOW ALL SLAVES STATUS` to see all channels at once.
- **`Seconds_Behind_Master` is unreliable**. It measures the timestamp on the SQL thread relative to the event being applied, NOT actual wall-clock lag. Use `pt-heartbeat` (or the equivalent token-based heartbeat) for accurate lag measurement.

## Decision Tree : Which Replication Model

```
START : you need replication
├── data-loss tolerance is non-zero (analytics, reporting)
│   └── async (default)
│       └── tune : slave_parallel_threads, slave_parallel_mode=optimistic
├── data-loss tolerance is small but non-zero (most OLTP)
│   └── semi-sync (10.3+ built-in)
│       ├── set rpl_semi_sync_master_enabled=ON on primary
│       ├── set rpl_semi_sync_slave_enabled=ON on at least one replica
│       └── understand : on timeout (default 10s) the primary reverts to async
├── ZERO data-loss tolerance + multi-master writes
│   └── Galera Cluster (see mariadb-core-galera-cluster skill)
│       └── NEVER conflate Galera with semi-sync ; different consistency model
├── one replica must pull from multiple primaries
│   └── multi-source (10.0+, named connections)
└── you are migrating off MySQL and want continuous replication
    ├── MySQL primary -> MariaDB replica : OK with binlog-position or MySQL-GTID
    └── MariaDB primary -> MySQL replica : NOT SUPPORTED, dump-and-load only
```

## Decision Tree : Binary Log Format

```
START : choose binlog_format
├── Galera Cluster node
│   └── ROW (required, no exceptions)
├── async or semi-sync primary with non-deterministic functions (UUID(), NOW() in some contexts, RAND())
│   └── ROW (safest) OR MIXED (auto-switches per statement)
├── async or semi-sync primary with mostly deterministic SQL + tight binlog-volume budget
│   └── MIXED (default since 10.2.4)
├── primary needs point-in-time recovery with full row state
│   └── ROW + binlog_row_image=FULL (default)
└── you think STATEMENT will save space
    └── STOP. Use MIXED. STATEMENT corrupts on unsafe statements ; the savings are an illusion.
```

## Pattern : Async Replication Baseline (10.6+)

```ini
# primary my.cnf
[mariadb]
server_id=1
log_bin=mariadb-bin
binlog_format=MIXED
sync_binlog=1                  # crash-safe binlog
expire_logs_days=14
```

```ini
# replica my.cnf
[mariadb]
server_id=2
log_bin=mariadb-bin
binlog_format=MIXED
read_only=ON
slave_parallel_threads=4
slave_parallel_mode=optimistic # 10.5.1+ default, set explicitly for clarity
```

```sql
-- replica side, GTID-based attach (10.0.13+)
CHANGE MASTER TO
  MASTER_HOST='primary.example.com',
  MASTER_USER='repl',
  MASTER_PASSWORD='<secret>',
  MASTER_USE_GTID=slave_pos;

START SLAVE;
SHOW SLAVE STATUS\G
```

## Pattern : Enable Semi-Sync (10.3+)

```ini
# primary my.cnf (10.3+)
[mariadb]
rpl_semi_sync_master_enabled=ON
rpl_semi_sync_master_timeout=10000   # default, 10 seconds
rpl_semi_sync_master_wait_point=AFTER_SYNC
```

```ini
# replica my.cnf (10.3+)
[mariadb]
rpl_semi_sync_slave_enabled=ON
```

ALWAYS confirm both sides are active :

```sql
-- on primary
SHOW STATUS LIKE 'Rpl_semi_sync_master_status';   -- expect ON
SHOW STATUS LIKE 'Rpl_semi_sync_master_clients';  -- expect >= 1

-- on replica
SHOW STATUS LIKE 'Rpl_semi_sync_slave_status';    -- expect ON
```

NEVER assume semi-sync is "always synchronous". On `rpl_semi_sync_master_timeout` expiry the primary silently reverts to async. Watch `Rpl_semi_sync_master_status` over time, alert if it flips OFF.

## Pattern : Parallel-Applier Tuning (10.5.1+)

```ini
# replica my.cnf
[mariadb]
slave_parallel_threads=8                # worker pool size
slave_parallel_mode=optimistic          # default in 10.5.1+
slave_parallel_max_queued=262144        # per-worker queue in bytes
slave_domain_parallel_threads=2         # per-domain cap, 0 = no cap
```

| Mode | Behavior | When to use |
|------|----------|-------------|
| `optimistic` | Apply transactions in parallel ; on row-level conflict roll back and retry. Default 10.5.1+. | General OLTP workloads. |
| `conservative` | Respect group-commit ordering from the primary ; only non-conflicting transactions run in parallel. Default until 10.5.0. | Stable, predictable, no retry storms. |
| `aggressive` | Parallel without conflict-avoidance heuristics. Risk of data corruption if dependencies are misjudged. | NEVER in production without exhaustive testing. |
| `minimal` | Only commit-phase runs in parallel. | Marginal speed-up, very safe. |
| `none` | Single-threaded SQL thread. | Debugging only ; effectively disables parallel apply. |

ALWAYS benchmark `slave_parallel_threads > 16` against your actual disk IOPS. Beyond the point where workers wait on flushes, more threads ADD latency rather than removing it.

## Pattern : Multi-Source Replication (10.0+, Named Connections)

```sql
-- primary A
CHANGE MASTER 'analytics' TO
  MASTER_HOST='primary-a.example.com',
  MASTER_USER='repl',
  MASTER_PASSWORD='<secret>',
  MASTER_USE_GTID=slave_pos;

-- primary B
CHANGE MASTER 'reporting' TO
  MASTER_HOST='primary-b.example.com',
  MASTER_USER='repl',
  MASTER_PASSWORD='<secret>',
  MASTER_USE_GTID=slave_pos;

START SLAVE 'analytics';
START SLAVE 'reporting';

SHOW ALL SLAVES STATUS\G   -- all channels in one go
STOP SLAVE 'analytics';
RESET SLAVE 'analytics' ALL;   -- remove channel permanently
```

NEVER use `FOR CHANNEL <name>` syntax in MariaDB. That is MySQL 5.7+ syntax and will fail. NEVER reuse `auto_increment_increment` and `auto_increment_offset` ranges across primaries fanning into the same replica ; conflicting ranges cause silent PK collisions on aggregation tables.

## Pattern : GTID Configuration

```ini
# primary my.cnf
[mariadb]
server_id=1
log_slave_updates=ON          # required on intermediate replicas in chains
gtid_domain_id=0              # 32-bit unsigned, identifies logical stream
gtid_strict_mode=ON           # reject divergent GTIDs proactively
```

```sql
-- attach by GTID
CHANGE MASTER TO
  MASTER_USE_GTID=slave_pos;   -- one of : slave_pos | current_pos | no

-- inspect
SELECT @@global.gtid_domain_id, @@global.gtid_current_pos, @@global.gtid_binlog_pos;
SHOW SLAVE STATUS\G            -- look at Gtid_IO_Pos and Using_Gtid
```

| Variable | Role |
|----------|------|
| `gtid_domain_id` | 32-bit unsigned, identifies a logical replication stream. Use distinct values when multiple writers exist (multi-source). |
| `gtid_slave_pos` | Last GTID the SQL thread applied. Authoritative position for `MASTER_USE_GTID=slave_pos`. |
| `gtid_binlog_pos` | Last GTID written to this server's binlog. |
| `gtid_current_pos` | Composite : highest of slave_pos and binlog_pos per domain. Used by `MASTER_USE_GTID=current_pos`. |
| `gtid_strict_mode` | Rejects lower sequence numbers, manual binlog entries with lower seq, missing GTIDs on the primary. ALWAYS enable on long-lived replicas. |

## Pattern : Monitoring Replication Health

```sql
-- single-channel async or semi-sync
SHOW SLAVE STATUS\G

-- multi-source
SHOW ALL SLAVES STATUS\G

-- 10.5+ also supports the REPLICA aliases
SHOW REPLICA STATUS\G
SHOW ALL REPLICAS STATUS\G
```

Critical columns :

| Column | Meaning | Healthy value |
|--------|---------|---------------|
| `Slave_IO_Running` | I/O thread connected to primary | `Yes` |
| `Slave_SQL_Running` | SQL thread applying events | `Yes` |
| `Seconds_Behind_Master` | Wall-clock delta on the event being applied. Unreliable for true lag. | `0` or steady-state low. Cross-check with `pt-heartbeat`. |
| `Last_IO_Error` | Last I/O error string | empty |
| `Last_SQL_Error` | Last SQL apply error | empty |
| `Gtid_IO_Pos` | Latest GTID received | tracks primary's `gtid_binlog_pos` |
| `Using_Gtid` | `Slave_Pos`, `Current_Pos`, or `No` | matches your `MASTER_USE_GTID` |

NEVER alert on `Seconds_Behind_Master > 0` alone. It jumps to 0 between transactions and can mislead. Pair with a real heartbeat table.

## Anti-Patterns (Top Picks)

ALWAYS read `references/anti-patterns.md` for the full set. Top-3 reminders :

1. STATEMENT binlog with `NOW()`, `UUID()`, `RAND()` : silent drift. Use MIXED or ROW.
2. Attempting MariaDB-primary -> MySQL-replica with GTID : not supported per the KB. One-way dump-and-load only.
3. `slave_parallel_mode=aggressive` in production without exhaustive testing : data corruption risk.

## Cross-References

- `mariadb-core-galera-cluster` : synchronous multi-master, different consistency model.
- `mariadb-core-architecture` : storage-engine + replication interaction (binlog hooks).
- `mariadb-impl-migration-mysql-to-mariadb` : GTID-incompatibility recovery patterns.
- `mariadb-errors-replication-broken` : Last_SQL_Error triage flow.
- `mariadb-impl-backup-and-pitr` : binlog-based point-in-time recovery.

## References

- `references/methods.md` : full variable reference (rpl_semi_sync_*, slave_parallel_*, gtid_*, binlog_*).
- `references/examples.md` : end-to-end working examples (async, semi-sync, parallel-applier, multi-source, TLS, monitoring).
- `references/anti-patterns.md` : 10+ field-tested anti-patterns with code, why-it-fails, and the correct alternative.

## Sources

- MariaDB KB : semi-synchronous replication. https://mariadb.com/kb/en/semisynchronous-replication/
- MariaDB KB : parallel replication. https://mariadb.com/kb/en/parallel-replication/
- MariaDB KB : Global Transaction ID (GTID). https://mariadb.com/kb/en/gtid/
- MariaDB KB : Binary log formats. https://mariadb.com/kb/en/binary-log-formats/
- MariaDB KB : Replication and binary log system variables. https://mariadb.com/kb/en/replication-and-binary-log-system-variables/
- MariaDB KB : Multi-source replication. https://mariadb.com/kb/en/multi-source-replication/
- MariaDB KB : SHOW SLAVE STATUS. https://mariadb.com/kb/en/show-slave-status/
