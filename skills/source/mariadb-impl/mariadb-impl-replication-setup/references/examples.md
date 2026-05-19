# examples.md : Replication Setup Examples

10+ runnable end-to-end examples for MariaDB replication. All examples verified against MariaDB 10.6-LTS, 10.11-LTS, 11.x, 12.x on 2026-05-19.

---

## Example 1 : Minimal Async Primary-Replica (positional, 10.6+)

```ini
# PRIMARY my.cnf
[mariadb]
server_id      = 1
log_bin        = mariadb-bin
log_basename   = primary1
binlog_format  = MIXED
sync_binlog    = 1
```

```ini
# REPLICA my.cnf
[mariadb]
server_id      = 2
log_bin        = mariadb-bin
log_basename   = replica1
binlog_format  = MIXED
read_only      = ON
```

```sql
-- PRIMARY : create user and capture position
CREATE USER 'repl'@'10.0.0.%' IDENTIFIED BY 'CHANGEME';
GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'repl'@'10.0.0.%';

FLUSH TABLES WITH READ LOCK;
SHOW MASTER STATUS;                -- record File and Position
-- copy datadir or run mariabackup to replica
UNLOCK TABLES;
```

```sql
-- REPLICA : point at primary, start
CHANGE MASTER TO
  MASTER_HOST     = 'primary.example.com',
  MASTER_USER     = 'repl',
  MASTER_PASSWORD = 'CHANGEME',
  MASTER_LOG_FILE = 'primary1-bin.000001',
  MASTER_LOG_POS  = 568;
START SLAVE;
SHOW SLAVE STATUS \G
```

---

## Example 2 : GTID-Based Setup (recommended, 10.6+)

```sql
-- PRIMARY : set domain id and strict mode
SET GLOBAL gtid_domain_id   = 1;
SET GLOBAL gtid_strict_mode = ON;
```

```sql
-- REPLICA : set domain id, strict mode, MASTER_USE_GTID = slave_pos
SET GLOBAL gtid_domain_id   = 1;
SET GLOBAL gtid_strict_mode = ON;

CHANGE MASTER TO
  MASTER_HOST     = 'primary.example.com',
  MASTER_USER     = 'repl',
  MASTER_PASSWORD = 'CHANGEME',
  MASTER_USE_GTID = slave_pos;

START SLAVE;
SHOW SLAVE STATUS \G              -- Using_Gtid should show Slave_Pos
```

For chained replication A -> B -> C, use `MASTER_USE_GTID = slave_pos` on C so its position tracks only GTIDs replicated through B (not local writes on B).

---

## Example 3 : Semi-Sync Enable (10.3+, built-in)

```sql
-- PRIMARY : runtime enable, NO plug-in install needed in 10.3+
SET GLOBAL rpl_semi_sync_master_enabled         = ON;
SET GLOBAL rpl_semi_sync_master_timeout         = 10000;     -- ms
SET GLOBAL rpl_semi_sync_master_wait_point      = AFTER_SYNC;
SET GLOBAL rpl_semi_sync_master_wait_for_slave_count = 1;
```

```sql
-- REPLICA : runtime enable + restart I/O thread to engage
SET GLOBAL rpl_semi_sync_slave_enabled = ON;
STOP SLAVE IO_THREAD;
START SLAVE IO_THREAD;
```

```sql
-- verify on PRIMARY
SHOW STATUS LIKE 'Rpl_semi_sync_master_status';      -- expect ON
SHOW STATUS LIKE 'Rpl_semi_sync_master_clients';     -- expect 1+
SHOW STATUS LIKE 'Rpl_semi_sync_master_no_tx';       -- count fallen back to async
```

To persist, mirror the variables in my.cnf under `[mariadb]`.

---

## Example 4 : Parallel Applier Tuning (10.5.1+ optimistic default)

```ini
# REPLICA my.cnf
[mariadb]
slave_parallel_threads        = 8
slave_parallel_mode           = optimistic
slave_parallel_max_queued     = 524288
slave_domain_parallel_threads = 4
```

```sql
-- runtime change : stop all replicas first
STOP ALL SLAVES;
SET GLOBAL slave_parallel_threads        = 8;
SET GLOBAL slave_parallel_mode           = optimistic;
SET GLOBAL slave_parallel_max_queued     = 524288;
START ALL SLAVES;

-- verify workers are running
SHOW PROCESSLIST;                          -- look for 'Slave_worker' rows
```

ALWAYS benchmark on actual workload before raising threads above 8. With slow disks more threads only add contention.

---

## Example 5 : Multi-Source NAMED CONNECTION (10.0+)

```sql
-- REPLICA : aggregate from two primaries into one replica
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

SHOW ALL SLAVES STATUS \G                  -- per-channel state
SHOW SLAVE 'analytics' STATUS \G

-- send subsequent legacy single-source commands to a specific channel
SET @@session.default_master_connection = 'analytics';
SHOW SLAVE STATUS \G                       -- now shows analytics
```

NEVER use MySQL `FOR CHANNEL` syntax on MariaDB. The parser rejects it.

---

## Example 6 : Remove a Multi-Source Channel Permanently

```sql
STOP SLAVE 'analytics';
RESET SLAVE 'analytics' ALL;               -- ALL = delete channel completely
SHOW ALL SLAVES STATUS \G                  -- analytics no longer listed
```

Without `ALL`, the channel definition stays but its position is cleared.

---

## Example 7 : TLS-Encrypted Replication

```sql
-- REPLICA (10.6+) : encrypt + verify primary's certificate
STOP SLAVE;
CHANGE MASTER TO
  MASTER_SSL                    = 1,
  MASTER_SSL_CA                 = '/etc/mysql/certs/ca.pem',
  MASTER_SSL_CERT               = '/etc/mysql/certs/replica-cert.pem',
  MASTER_SSL_KEY                = '/etc/mysql/certs/replica-key.pem',
  MASTER_SSL_VERIFY_SERVER_CERT = 1,
  MASTER_SSL_CIPHER             = 'ECDHE-RSA-AES256-GCM-SHA384';
START SLAVE;
SHOW SLAVE STATUS \G                       -- Master_SSL_Allowed = Yes
```

`MASTER_SSL_VERIFY_SERVER_CERT = 1` is MANDATORY in any production setup. From 12.3+ each TLS option also accepts `DEFAULT` to inherit server-level TLS config.

---

## Example 8 : Skip a Bad Event (recovery, GTID mode)

```sql
-- replica stopped on duplicate-key error from a row already present
STOP SLAVE;
SHOW SLAVE STATUS \G                       -- read Last_SQL_Error and Gtid_Slave_Pos

-- advance past the offending GTID
SET GLOBAL gtid_slave_pos = '0-1-1234,1-2-5678';     -- target is one past failing GTID
CHANGE MASTER TO MASTER_USE_GTID = slave_pos;
START SLAVE;
SHOW SLAVE STATUS \G
```

For positional mode use `SET GLOBAL sql_slave_skip_counter = 1;` instead. Skipping events should be a LAST resort; investigate root cause first.

---

## Example 9 : MariaDB Replicating FROM MySQL (one-way migration bridge)

```sql
-- on MariaDB REPLICA (target version 11.4.5+ for best MySQL 8.0 compatibility)
CHANGE MASTER TO
  MASTER_HOST     = 'mysql-primary.example.com',
  MASTER_USER     = 'repl',
  MASTER_PASSWORD = 'CHANGEME',
  MASTER_LOG_FILE = 'mysql-bin.000017',    -- MySQL binlog file
  MASTER_LOG_POS  = 12345,
  MASTER_USE_GTID = no;                    -- MySQL GTID format incompatible
START SLAVE;
```

MariaDB CAN read MySQL binlogs in positional mode (per KB : "MySQL 8.0 -> MariaDB requires MariaDB 11.4.5 or newer"). MySQL GTID `uuid:seqno` does NOT translate to MariaDB GTID `domain-server-sequence`. After cutover, switch to MariaDB-native GTID by stopping replication and issuing a fresh `CHANGE MASTER TO MASTER_USE_GTID = slave_pos` on a pure MariaDB chain.

MariaDB CANNOT replicate TO MySQL : MySQL refuses MariaDB binlog events.

---

## Example 10 : Monitoring Replication Health

```sql
-- per-channel state
SHOW ALL SLAVES STATUS \G

-- single quick check
SELECT
  CONNECTION_NAME           AS channel,
  SLAVE_IO_RUNNING          AS io_run,
  SLAVE_SQL_RUNNING         AS sql_run,
  SECONDS_BEHIND_MASTER     AS lag_sec,
  USING_GTID,
  GTID_IO_POS,
  GTID_SLAVE_POS
FROM information_schema.SLAVE_STATUS;

-- semi-sync health
SHOW STATUS LIKE 'Rpl_semi_sync_master_status';
SHOW STATUS LIKE 'Rpl_semi_sync_master_no_tx';
SHOW STATUS LIKE 'Rpl_semi_sync_master_tx_avg_wait_time';

-- parallel applier health
SELECT @@global.slave_parallel_threads, @@global.slave_parallel_mode;
SHOW STATUS LIKE 'Slave_running';
```

Alert thresholds typically used in production :
- `Seconds_Behind_Master > 60` for sustained 5 minutes
- `Slave_IO_Running != 'Yes'` or `Slave_SQL_Running != 'Yes'`
- `Rpl_semi_sync_master_status = OFF` while semi-sync is supposed to be active
- `Rpl_semi_sync_master_no_tx` increment rate > 0 (means fall-backs are happening)

---

## Example 11 : Switch a Replica to a New Primary (failover, GTID mode)

```sql
-- on the REPLICA being promoted to follow new-primary
STOP SLAVE;
CHANGE MASTER TO
  MASTER_HOST     = 'new-primary.example.com',
  MASTER_USER     = 'repl',
  MASTER_PASSWORD = 'CHANGEME',
  MASTER_USE_GTID = slave_pos;
START SLAVE;
SHOW SLAVE STATUS \G
```

With GTID-based replication the replica resumes from its `gtid_slave_pos` on the new primary automatically, no log-file or position juggling. With positional replication you must compute the new MASTER_LOG_FILE and MASTER_LOG_POS from `SHOW MASTER STATUS` on the new primary, which is error-prone : prefer GTID for any environment that may failover.

---

## Example 12 : Force Reset After Repointing Data Source (clean slate)

```sql
-- REPLICA : wipe all replication state before pointing at a freshly-restored snapshot
STOP ALL SLAVES;
RESET SLAVE ALL;                           -- clears default channel completely
RESET MASTER;                              -- clears local binlog + gtid_binlog_pos

-- if multi-source, wipe each named channel too
RESET SLAVE 'analytics' ALL;
RESET SLAVE 'orders' ALL;

-- then re-CHANGE MASTER and START SLAVE per Example 2 or 5
```

`RESET MASTER` is destructive to local binlogs : only use on a node that does not itself act as a primary for other replicas.
