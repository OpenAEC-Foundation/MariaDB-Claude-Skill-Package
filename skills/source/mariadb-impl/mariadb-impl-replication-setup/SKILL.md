---
name: mariadb-impl-replication-setup
description: >
  Use when setting up primary-replica replication, enabling semi-sync, configuring parallel applier, setting up GTID,
  configuring multi-source channels with named connections, or bridging replication from MySQL.
  Prevents the common mistake of expecting MariaDB GTID to interop with MySQL GTID (incompatible per L-004),
  using STATEMENT binlog with non-deterministic functions, copying MySQL `FOR CHANNEL` syntax,
  or running parallel_threads without a disk-IO benchmark.
  Covers server-id assignment, log_bin enable, CHANGE MASTER setup (positional and GTID),
  START SLAVE, semi-sync enable (built-in 10.3+), slave_parallel_threads and slave_parallel_mode
  optimistic (default 10.5.1+), MariaDB GTID (domain-server-sequence format), multi-source replication
  with NAMED CONNECTION syntax, and SHOW SLAVE STATUS monitoring.
  Keywords: replication setup, CHANGE MASTER, START SLAVE, server-id, log_bin, semi-sync, rpl_semi_sync_master_enabled,
  parallel applier, slave_parallel_threads, slave_parallel_mode optimistic, GTID, gtid_strict_mode, gtid_slave_pos,
  MASTER_USE_GTID, named connection, multi-source replication, SHOW SLAVE STATUS, SHOW ALL SLAVES STATUS,
  REPLICATION SLAVE grant, how do I set up replication, replication broken, replica lag, can mariadb replicate to mysql,
  why does FOR CHANNEL fail, what is domain-server-sequence
license: MIT
compatibility: "Designed for Claude Code. Requires MariaDB 10.6-LTS, 10.11-LTS, 11.x, 12.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# MariaDB Replication Setup

Deterministic patterns for primary-replica replication: async baseline, semi-sync, parallel applier, GTID, and multi-source named connections. Skill covers MariaDB 10.6-LTS, 10.11-LTS, 11.x, 12.x.

## Quick Reference

- ALWAYS set a UNIQUE `server_id` (1 to 2^32-1) on every node before enabling `log_bin`. Two nodes with the same `server_id` silently corrupt replication.
- ALWAYS use `binlog_format=MIXED` (default since 10.2.4) or `ROW`. NEVER use `STATEMENT` alone with non-deterministic functions (`UUID()`, `NOW(6)`, `LIMIT` without `ORDER BY`, `SLEEP`, user-defined functions).
- Semi-sync is BUILT-IN from MariaDB 10.3+. NEVER run `INSTALL PLUGIN rpl_semi_sync_master`. Just set `rpl_semi_sync_master_enabled=ON` on the primary and `rpl_semi_sync_slave_enabled=ON` on the replica.
- On `rpl_semi_sync_master_timeout` (default 10000 ms) the primary falls back to async and logs a warning. When a replica catches up, semi-sync resumes automatically. Apparent "hangs" under load are usually this timeout being too low.
- `slave_parallel_mode=optimistic` is the DEFAULT from 10.5.1+. Earlier versions default to `conservative`. Both primary and replica must be 10.0.5+ for any parallel mode.
- Start `slave_parallel_threads` at 4-8 and TUNE based on actual disk-IO benchmark on the replica. NEVER set above 16 without proof it helps.
- MariaDB GTID format is `domain-server-sequence` (e.g. `0-1-1234`). MySQL GTID is `uuid:seqno`. They are INCOMPATIBLE. MariaDB can replicate FROM a MySQL primary, but MySQL CANNOT replicate from a MariaDB primary. For two-way or MariaDB-to-MySQL: dump-and-load, not GTID.
- Multi-source uses NAMED CONNECTIONS: `CHANGE MASTER 'channel_name' TO ...`. NEVER use MySQL `FOR CHANNEL` syntax: it returns a syntax error.
- `SHOW SLAVE STATUS` (single source) or `SHOW ALL SLAVES STATUS` (multi-source, 10.0+). Replica aliases `SHOW REPLICA STATUS` exist from 10.5+. Both `Slave_IO_Running` and `Slave_SQL_Running` MUST be `Yes`.
- Replication user privileges: `REPLICATION SLAVE` (mandatory) + `REPLICATION CLIENT` (recommended, for `SHOW MASTER STATUS`).

## Decision Tree : Which Replication Shape

```
START
 |
 +-- Need lowest write latency, can tolerate up to a few seconds RPO on primary crash ?
 |       --> async replication (default), GTID-based (MASTER_USE_GTID = slave_pos)
 |
 +-- Need crash-safe replication with at-most-one-transaction loss ?
 |       --> semi-sync (rpl_semi_sync_master_enabled=ON on primary, rpl_semi_sync_slave_enabled=ON on replica)
 |       --> tune rpl_semi_sync_master_timeout for your network RTT (default 10000 ms is generous)
 |
 +-- Replica falling behind under write load on primary ?
 |       --> enable parallel applier on replica: slave_parallel_threads=4, slave_parallel_mode=optimistic
 |       --> benchmark before raising threads above 8
 |
 +-- Aggregating from N primaries into one replica ?
 |       --> multi-source, NAMED CONNECTION syntax, one CHANGE MASTER 'name' TO ... per source
 |       --> also set slave_domain_parallel_threads to prevent one domain starving others
 |
 +-- Migrating FROM MySQL ?
         --> MariaDB CAN replicate from MySQL (positional or limited GTID compatibility via 11.4.5+)
         --> MariaDB CANNOT replicate TO MySQL (one-way migration; use dump-and-load for cutover)
```

## Decision Tree : Positional vs GTID

```
START
 |
 +-- New deployment, both primary and replica are MariaDB 10.0.2+ ?
 |       --> use GTID. MASTER_USE_GTID = slave_pos on the replica.
 |       --> set gtid_strict_mode = ON globally to catch divergence early.
 |
 +-- Chained replication (A -> B -> C) and B is a relay ?
 |       --> on C, use MASTER_USE_GTID = slave_pos (NOT current_pos).
 |       --> slave_pos tracks ONLY replicated GTIDs, preventing local writes on B from breaking C.
 |
 +-- Need backward-compatible positional setup for tooling that expects MASTER_LOG_FILE / MASTER_LOG_POS ?
         --> use MASTER_USE_GTID = no, supply MASTER_LOG_FILE and MASTER_LOG_POS from SHOW MASTER STATUS.
```

## Minimal Primary-Replica Setup (positional, MariaDB 10.6+)

```ini
# /etc/mysql/mariadb.conf.d/50-server.cnf  -- PRIMARY (10.6+)
[mariadb]
server_id          = 1
log_bin            = mariadb-bin
log_basename       = primary1
binlog_format      = MIXED
sync_binlog        = 1
binlog_row_image   = FULL
```

```ini
# /etc/mysql/mariadb.conf.d/50-server.cnf  -- REPLICA (10.6+)
[mariadb]
server_id          = 2
log_bin            = mariadb-bin
log_basename       = replica1
binlog_format      = MIXED
read_only          = ON
```

Restart both, then on the primary:

```sql
-- 10.6+
CREATE USER 'repl'@'10.0.0.%' IDENTIFIED BY 'CHANGEME';
GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'repl'@'10.0.0.%';

FLUSH TABLES WITH READ LOCK;
SHOW MASTER STATUS;     -- record File and Position; copy data to replica; then:
UNLOCK TABLES;
```

On the replica:

```sql
-- 10.6+
CHANGE MASTER TO
  MASTER_HOST     = 'primary.example.com',
  MASTER_PORT     = 3306,
  MASTER_USER     = 'repl',
  MASTER_PASSWORD = 'CHANGEME',
  MASTER_LOG_FILE = 'primary1-bin.000001',
  MASTER_LOG_POS  = 568,
  MASTER_CONNECT_RETRY = 10;

START SLAVE;
SHOW SLAVE STATUS \G        -- Slave_IO_Running and Slave_SQL_Running must both be Yes
```

## GTID-Based Setup (recommended for new deployments)

```sql
-- on PRIMARY (10.6+) : set domain id and strict mode once
SET GLOBAL gtid_domain_id   = 0;
SET GLOBAL gtid_strict_mode = ON;
```

```sql
-- on REPLICA (10.6+) : same domain id + strict mode, then point at primary
SET GLOBAL gtid_domain_id   = 0;
SET GLOBAL gtid_strict_mode = ON;

CHANGE MASTER TO
  MASTER_HOST     = 'primary.example.com',
  MASTER_USER     = 'repl',
  MASTER_PASSWORD = 'CHANGEME',
  MASTER_USE_GTID = slave_pos;

START SLAVE;
SHOW SLAVE STATUS \G
```

Use `MASTER_USE_GTID = current_pos` ONLY when the replica also accepts local writes that are later promoted; otherwise pick `slave_pos`.

## Semi-Sync (10.3+, built-in)

```sql
-- PRIMARY (10.3+) : semi-sync built into server, no plug-in install
SET GLOBAL rpl_semi_sync_master_enabled = ON;
SET GLOBAL rpl_semi_sync_master_timeout = 10000;  -- ms; primary waits up to this for replica ACK
```

```sql
-- REPLICA (10.3+)
SET GLOBAL rpl_semi_sync_slave_enabled = ON;

-- if I/O thread already running, restart it to engage semi-sync
STOP SLAVE IO_THREAD;
START SLAVE IO_THREAD;
```

To persist across restarts add to my.cnf under `[mariadb]`. On timeout the primary AUTOMATICALLY falls back to async and logs a warning; semi-sync resumes when a replica catches up. Apparent "hangs" or sudden latency spikes on the primary under load are typically a too-low timeout.

## Parallel Applier (replica side, both nodes 10.0.5+)

```ini
# REPLICA my.cnf (10.5.1+ : optimistic is default mode)
[mariadb]
slave_parallel_threads        = 4
slave_parallel_mode           = optimistic
slave_parallel_max_queued     = 262144
slave_domain_parallel_threads = 2
```

Modes :
- `optimistic` (default 10.5.1+) : transactional DML in parallel, rollback-and-retry on conflict
- `conservative` (default until 10.5.0) : uses group-commit info from primary, safest
- `aggressive` : optimistic without conflict-avoidance heuristics; benchmark before use
- `minimal` : only the commit phase runs in parallel
- `none` : single-threaded applier

ALWAYS benchmark on the replica's actual disk before raising `slave_parallel_threads` above 8. Without disk-IO headroom, more threads only add contention.

## Multi-Source (10.0+, NAMED CONNECTION syntax)

```sql
-- REPLICA (10.0+) : NAMED CONNECTION syntax; quotes around channel name
CHANGE MASTER 'analytics' TO
  MASTER_HOST     = 'analytics-primary.example.com',
  MASTER_USER     = 'repl',
  MASTER_PASSWORD = 'CHANGEME',
  MASTER_USE_GTID = slave_pos;

CHANGE MASTER 'orders' TO
  MASTER_HOST     = 'orders-primary.example.com',
  MASTER_USER     = 'repl',
  MASTER_PASSWORD = 'CHANGEME',
  MASTER_USE_GTID = slave_pos;

START SLAVE 'analytics';
START SLAVE 'orders';

SHOW ALL SLAVES STATUS \G

-- to remove a channel permanently:
STOP SLAVE 'analytics';
RESET SLAVE 'analytics' ALL;
```

NEVER use `FOR CHANNEL` (MySQL syntax) on MariaDB: it returns a parser error. The `default_master_connection` variable (default `''`) controls which connection legacy single-source commands address.

## Monitoring

```sql
-- single-source
SHOW SLAVE STATUS \G                 -- 10.x baseline
SHOW REPLICA STATUS \G               -- 10.5+ alias

-- multi-source
SHOW ALL SLAVES STATUS \G            -- 10.0+
SHOW SLAVE 'analytics' STATUS \G     -- 10.0+, per-channel

-- key columns to inspect
-- Slave_IO_Running, Slave_SQL_Running : both must be Yes
-- Seconds_Behind_Master              : applier lag in seconds
-- Last_IO_Error, Last_SQL_Error       : diagnostic on failure
-- Using_Gtid                          : Slave_Pos / Current_Pos / No
-- Gtid_IO_Pos, Gtid_Slave_Pos         : GTID coordinates
```

## TLS for Replication

```sql
-- REPLICA (10.6+) : encrypt + verify primary's certificate
STOP SLAVE;
CHANGE MASTER TO
  MASTER_SSL                    = 1,
  MASTER_SSL_CA                 = '/etc/mysql/certs/ca.pem',
  MASTER_SSL_CERT               = '/etc/mysql/certs/replica-cert.pem',
  MASTER_SSL_KEY                = '/etc/mysql/certs/replica-key.pem',
  MASTER_SSL_VERIFY_SERVER_CERT = 1;
START SLAVE;
```

ALWAYS set `MASTER_SSL_VERIFY_SERVER_CERT = 1`. From 12.3+ these options accept `DEFAULT` to inherit server-level TLS config.

## Reference Files

- [methods.md](references/methods.md) : complete variable list, full `CHANGE MASTER` grammar, GRANT syntax, monitoring queries
- [examples.md](references/examples.md) : 10+ runnable end-to-end setups (async, GTID, semi-sync, parallel, multi-source, TLS, MySQL-to-MariaDB bridge)
- [anti-patterns.md](references/anti-patterns.md) : 8+ real-world failure modes with root cause and fix

## Source Verification

All patterns verified against MariaDB KB on 2026-05-19 :
- https://mariadb.com/kb/en/setting-up-replication/
- https://mariadb.com/kb/en/semisynchronous-replication/
- https://mariadb.com/kb/en/parallel-replication/
- https://mariadb.com/kb/en/gtid/
- https://mariadb.com/kb/en/multi-source-replication/
- https://mariadb.com/kb/en/change-master-to/
- https://mariadb.com/kb/en/binary-log-formats/
