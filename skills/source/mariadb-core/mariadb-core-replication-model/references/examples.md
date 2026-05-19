# examples.md : Working Replication Examples

End-to-end working examples for the patterns in `SKILL.md`. All examples version-annotated against MariaDB 10.6-LTS, 10.11-LTS, 11.x, 12.x.

---

## Example 1 : Minimal Async Primary-Replica (10.6+)

Two servers, async replication, GTID-based attach, parallel-applier on.

```ini
# primary : /etc/mysql/mariadb.conf.d/50-server.cnf
[mariadb]
server_id              = 1
log_bin                = mariadb-bin
binlog_format          = MIXED
sync_binlog            = 1
binlog_expire_logs_seconds = 1209600   # 14 days
max_binlog_size        = 1G
gtid_domain_id         = 0
gtid_strict_mode       = ON
```

```ini
# replica : /etc/mysql/mariadb.conf.d/50-server.cnf
[mariadb]
server_id              = 2
log_bin                = mariadb-bin
log_slave_updates      = ON
binlog_format          = MIXED
read_only              = ON
slave_parallel_threads = 4
slave_parallel_mode    = optimistic
gtid_strict_mode       = ON
```

```sql
-- on primary : create replication user
CREATE USER 'repl'@'10.0.0.%' IDENTIFIED BY 'strongpw';
GRANT REPLICATION REPLICA ON *.* TO 'repl'@'10.0.0.%';
-- alias for 10.5+ : REPLICATION SLAVE works on all versions

-- bootstrap the replica from a mariabackup snapshot, then start
-- on replica :
CHANGE MASTER TO
  MASTER_HOST     = 'primary.example.com',
  MASTER_USER     = 'repl',
  MASTER_PASSWORD = 'strongpw',
  MASTER_USE_GTID = slave_pos;

START SLAVE;
SHOW SLAVE STATUS\G
```

Verify : `Slave_IO_Running = Yes`, `Slave_SQL_Running = Yes`, `Using_Gtid = Slave_Pos`.

---

## Example 2 : Enable Semi-Sync on Primary + Replica (10.3+)

```ini
# primary : add to my.cnf
[mariadb]
rpl_semi_sync_master_enabled     = ON
rpl_semi_sync_master_timeout     = 10000   # 10 seconds, KB default
rpl_semi_sync_master_wait_no_slave = ON
rpl_semi_sync_master_wait_point  = AFTER_SYNC
```

```ini
# replica : add to my.cnf
[mariadb]
rpl_semi_sync_slave_enabled = ON
```

Apply without restart :

```sql
-- on primary
SET GLOBAL rpl_semi_sync_master_enabled        = ON;
SET GLOBAL rpl_semi_sync_master_timeout        = 10000;
SET GLOBAL rpl_semi_sync_master_wait_point     = 'AFTER_SYNC';

-- on replica : MUST restart the IO thread for it to pick up the change
SET GLOBAL rpl_semi_sync_slave_enabled = ON;
STOP SLAVE IO_THREAD;
START SLAVE IO_THREAD;
```

Verify :

```sql
-- on primary
SHOW STATUS LIKE 'Rpl_semi_sync_master_status';    -- expect ON
SHOW STATUS LIKE 'Rpl_semi_sync_master_clients';   -- expect >= 1

-- on replica
SHOW STATUS LIKE 'Rpl_semi_sync_slave_status';     -- expect ON
```

Monitor flips :

```sql
-- on primary, run periodically
SHOW STATUS LIKE 'Rpl_semi_sync_master_no_tx';
SHOW STATUS LIKE 'Rpl_semi_sync_master_yes_tx';
SHOW STATUS LIKE 'Rpl_semi_sync_master_tx_avg_wait_time';
```

If `no_tx` increases or `master_status` flips to `OFF`, the primary has reverted to async ; investigate replica health or raise `rpl_semi_sync_master_timeout` if network jitter is the cause.

---

## Example 3 : Parallel-Applier Tuning Pass (10.5.1+)

```ini
# replica : my.cnf
[mariadb]
slave_parallel_threads        = 8
slave_parallel_mode           = optimistic
slave_parallel_max_queued     = 262144     # 256 KB per worker
slave_domain_parallel_threads = 0          # no per-domain cap
```

Dynamic adjustment without restart :

```sql
STOP SLAVE SQL_THREAD;
SET GLOBAL slave_parallel_threads = 12;
SET GLOBAL slave_parallel_mode    = 'optimistic';
START SLAVE SQL_THREAD;
```

NOTE : `slave_parallel_threads` requires the SQL thread to be stopped before it can change. The IO thread can keep running.

Benchmark loop (run from a load generator) :

```sql
-- baseline single-threaded
SET GLOBAL slave_parallel_threads = 0;
-- generate load on primary, measure lag

-- mid-tier
SET GLOBAL slave_parallel_threads = 4;
-- same load, measure lag

-- aggressive
SET GLOBAL slave_parallel_threads = 16;
-- same load, measure lag

-- back off if disk IOPS saturates
```

ALWAYS confirm with `iostat -x 1` that disk is not 100% busy ; if it is, more workers add nothing.

---

## Example 4 : Multi-Source Replication (10.0+)

One replica pulls from two primaries.

```sql
-- replica side : configure two named channels
CHANGE MASTER 'analytics' TO
  MASTER_HOST     = 'primary-a.example.com',
  MASTER_USER     = 'repl',
  MASTER_PASSWORD = 'pw-a',
  MASTER_USE_GTID = slave_pos;

CHANGE MASTER 'reporting' TO
  MASTER_HOST     = 'primary-b.example.com',
  MASTER_USER     = 'repl',
  MASTER_PASSWORD = 'pw-b',
  MASTER_USE_GTID = slave_pos;

START SLAVE 'analytics';
START SLAVE 'reporting';

-- inspect both at once
SHOW ALL SLAVES STATUS\G
```

Switch session default to operate on one without naming it :

```sql
SET @@default_master_connection = 'reporting';
SHOW SLAVE STATUS\G                -- now shows the 'reporting' channel
STOP SLAVE;                        -- stops 'reporting'
START SLAVE;                       -- starts 'reporting'
SET @@default_master_connection = '';
```

Remove a channel permanently :

```sql
STOP SLAVE 'reporting';
RESET SLAVE 'reporting' ALL;       -- ALL erases the connection metadata
```

ALWAYS configure distinct `auto_increment_offset` and identical `auto_increment_increment` per primary if both write to overlapping table schemas, otherwise PK collisions are guaranteed on the aggregating replica.

---

## Example 5 : Switch Binlog Format Per Session

Default is `MIXED`. For one heavy import you might want `ROW` explicitly :

```sql
-- session level (does NOT need SUPER on 10.5+ for session change)
SET SESSION binlog_format = 'ROW';

-- bulk load
LOAD DATA INFILE '/tmp/import.csv' INTO TABLE staging
  FIELDS TERMINATED BY ',' OPTIONALLY ENCLOSED BY '"';

-- restore session default automatically when connection closes
```

Global override (use sparingly) :

```sql
SET GLOBAL binlog_format = 'ROW';
-- existing connections keep their session value
-- new connections inherit the new global default
```

NEVER set `STATEMENT` globally on a production primary in 10.x. Use `MIXED` or `ROW`.

---

## Example 6 : TLS-Encrypted Replication (10.6+)

```sql
-- on replica
STOP SLAVE;

CHANGE MASTER TO
  MASTER_HOST                  = 'primary.example.com',
  MASTER_USER                  = 'repl',
  MASTER_PASSWORD              = 'strongpw',
  MASTER_USE_GTID              = slave_pos,
  MASTER_SSL                   = 1,
  MASTER_SSL_CA                = '/etc/mysql/certs/ca.pem',
  MASTER_SSL_CERT              = '/etc/mysql/certs/replica-cert.pem',
  MASTER_SSL_KEY               = '/etc/mysql/certs/replica-key.pem',
  MASTER_SSL_VERIFY_SERVER_CERT = 1;

START SLAVE;
SHOW SLAVE STATUS\G   -- Master_SSL_Allowed = Yes, Master_SSL_Verify_Server_Cert = Yes
```

ALWAYS set `MASTER_SSL_VERIFY_SERVER_CERT = 1`. Without it, TLS encrypts the wire but does not validate the server identity, so a MITM attacker with a self-signed cert can intercept.

---

## Example 7 : Promote a Replica to Primary (GTID-Based Failover)

Async or semi-sync topology. Old primary unreachable.

```sql
-- on the replica being promoted
STOP SLAVE;
RESET SLAVE ALL;             -- forget the old primary
SET GLOBAL read_only = OFF;  -- accept writes

-- record this server's gtid_current_pos before any new writes
SELECT @@global.gtid_current_pos;
```

On other replicas, repoint to the new primary :

```sql
STOP SLAVE;
CHANGE MASTER TO
  MASTER_HOST     = 'new-primary.example.com',
  MASTER_USER     = 'repl',
  MASTER_PASSWORD = 'strongpw',
  MASTER_USE_GTID = current_pos;
START SLAVE;
SHOW SLAVE STATUS\G
```

`MASTER_USE_GTID = current_pos` lets the replica resume from its highest known position across binlog+slave-pos per domain. NEVER use `slave_pos` if the replica has its own binlog writes (intermediate replicas) ; use `current_pos`.

---

## Example 8 : Verify GTID Position on Both Ends

```sql
-- on primary
SELECT @@global.gtid_domain_id, @@global.gtid_binlog_pos, @@global.gtid_current_pos;
SHOW BINARY LOGS;

-- on replica
SELECT @@global.gtid_slave_pos, @@global.gtid_current_pos;
SHOW SLAVE STATUS\G
```

In a healthy state :

- replica's `gtid_slave_pos` lags the primary's `gtid_binlog_pos` only by the in-flight transactions.
- `Slave_IO_Running = Yes`, `Slave_SQL_Running = Yes`.
- `Gtid_IO_Pos` on the replica tracks the primary's `gtid_binlog_pos`.

---

## Example 9 : Reduce Binlog Volume with NOBLOB

Tables with large `LONGTEXT` or `LONGBLOB` columns where most updates touch other columns :

```sql
-- session-level
SET SESSION binlog_row_image = 'NOBLOB';

-- global, persistent across restart only via my.cnf
SET GLOBAL  binlog_row_image = 'NOBLOB';
```

```ini
# my.cnf for global default
[mariadb]
binlog_row_image = NOBLOB
```

NOBLOB writes the full before-image MINUS unchanged BLOB and TEXT columns. ALWAYS verify downstream consumers (replicas, change-data-capture pipelines) can handle the partial image. NEVER use `MINIMAL` if any consumer needs full row reconstruction (PITR, audit).

---

## Example 10 : Heartbeat-Based Lag Monitoring

`Seconds_Behind_Master` is unreliable. Build a heartbeat table on the primary and read it on the replica.

```sql
-- on primary : heartbeat table
CREATE TABLE IF NOT EXISTS ops.heartbeat (
  id  TINYINT NOT NULL PRIMARY KEY,
  ts  DATETIME(6) NOT NULL
) ENGINE=InnoDB;

INSERT IGNORE INTO ops.heartbeat (id, ts) VALUES (1, NOW(6));

-- update every second from an external scheduler (cron, systemd timer)
-- UPDATE ops.heartbeat SET ts = NOW(6) WHERE id = 1;
```

```sql
-- on replica : measure lag
SELECT TIMESTAMPDIFF(MICROSECOND, ts, NOW(6)) / 1000000.0 AS lag_seconds
FROM ops.heartbeat
WHERE id = 1;
```

For production : use the `pt-heartbeat` daemon from Percona Toolkit, which manages the update cadence and resilience automatically. NEVER rely on `Seconds_Behind_Master` for alerting on tight SLAs.

---

## Example 11 : Inspect Parallel-Applier Worker State

```sql
SHOW PROCESSLIST;     -- look for replica workers (User = 'system user')

SELECT THREAD_ID, NAME, TYPE, PROCESSLIST_STATE
FROM   performance_schema.threads
WHERE  NAME LIKE 'thread/sql/rpl_parallel%';
```

Worker thread states to watch :

- `Waiting for prior transaction to commit` : normal serialization point.
- `Waiting for room in worker thread event queue` : raise `slave_parallel_max_queued`.
- `Waiting for prior transaction to start commit` : conservative-mode ordering.

NEVER kill a parallel-applier worker thread manually with `KILL`. Use `STOP SLAVE` to stop the whole apply pool cleanly.

---

## Example 12 : Confirm Cross-Vendor Migration Limit

Useful sanity check when planning a MySQL <-> MariaDB move.

```sql
-- on MariaDB primary, try pointing a MySQL replica at it
-- The MySQL replica will report (paraphrased) :
--   ER_MASTER_INFO : Master GTID format does not match.
-- KB explicitly states this is unsupported.
```

For one-shot migration MariaDB -> MySQL :

1. `mariadb-dump --master-data=2 --single-transaction --routines --events --triggers` on the MariaDB source.
2. Load into MySQL with `mysql < dump.sql`.
3. Set up MySQL-native replication on the MySQL side.
4. There is NO continuous GTID-tracked replication path back to MariaDB.

For MySQL -> MariaDB continuous replication : SUPPORTED. Point a MariaDB replica at a MySQL primary, MariaDB accepts MySQL binlog events. Use `MASTER_USE_GTID = no` (binlog position) since the GTID formats differ.

---

## Source Verification (2026-05-19)

All SQL syntax verified against :

- https://mariadb.com/kb/en/change-master-to/
- https://mariadb.com/kb/en/show-slave-status/
- https://mariadb.com/kb/en/multi-source-replication/
- https://mariadb.com/kb/en/parallel-replication/
- https://mariadb.com/kb/en/semisynchronous-replication/
- https://mariadb.com/kb/en/gtid/
- https://mariadb.com/kb/en/binary-log-formats/
- https://mariadb.com/kb/en/replication-and-binary-log-system-variables/
