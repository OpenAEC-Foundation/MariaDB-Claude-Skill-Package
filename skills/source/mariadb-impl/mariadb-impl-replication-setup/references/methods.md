# methods.md : Replication Reference

Complete variable list, statement grammar, GRANT syntax, and monitoring queries for MariaDB primary-replica replication.

All entries verified against MariaDB KB on 2026-05-19. Version annotations apply to MariaDB 10.6-LTS, 10.11-LTS, 11.x, 12.x.

## 1. Server Identity and Binary Log Variables

| Variable | Scope | Default | Notes |
|----------|-------|---------|-------|
| `server_id` | Global | 1 | Unique 1 to 2^32-1 per node. ALWAYS set explicitly. |
| `log_bin` | Global, read-only at runtime | OFF | Set in my.cnf as `log_bin = mariadb-bin` to enable. |
| `log_basename` | Global | host-derived | Sets base name for binlog files. Prevents breakage on hostname change. |
| `binlog_format` | Global, Session | `MIXED` (since 10.2.4) | Values : `STATEMENT`, `ROW`, `MIXED`. |
| `binlog_row_image` | Global, Session | `FULL` | Values : `FULL`, `MINIMAL`, `NOBLOB`. |
| `sync_binlog` | Global | 0 | Set to 1 for crash-safe binlog. Pairs with `innodb_flush_log_at_trx_commit=1`. |
| `expire_logs_days` | Global | 0 | Auto-purge binlog older than N days. |
| `max_binlog_size` | Global | 1 GB | Binlog rotation size. |
| `read_only` | Global | OFF | Set ON on replicas to block non-SUPER writes. |
| `super_read_only` | Global | OFF (10.11+) | Stronger than `read_only`; blocks SUPER users too. |

## 2. Semi-Sync Variables (built-in 10.3+)

| Variable | Side | Default | Notes |
|----------|------|---------|-------|
| `rpl_semi_sync_master_enabled` | Primary | OFF | Set ON to enable semi-sync on primary. NO plug-in install required 10.3+. |
| `rpl_semi_sync_master_timeout` | Primary | 10000 (ms) | Wait time for replica ACK before falling back to async. |
| `rpl_semi_sync_master_wait_for_slave_count` | Primary | 1 | Number of replica ACKs required before commit returns. |
| `rpl_semi_sync_master_wait_point` | Primary | `AFTER_SYNC` | `AFTER_SYNC` (safer) or `AFTER_COMMIT` (legacy). |
| `rpl_semi_sync_slave_enabled` | Replica | OFF | Set ON to enable semi-sync on replica. |
| `rpl_semi_sync_master_clients` | Primary (status) | 0 | Status variable : count of semi-sync replicas connected. |
| `rpl_semi_sync_master_status` | Primary (status) | OFF | Status : ON when semi-sync active, OFF when fallen back to async. |

After enabling on a running replica, run :
```sql
STOP SLAVE IO_THREAD;
START SLAVE IO_THREAD;
```

## 3. Parallel Applier Variables (both nodes 10.0.5+)

| Variable | Default | Notes |
|----------|---------|-------|
| `slave_parallel_threads` | 0 | Pool of applier worker threads. 0 = single-threaded. Recommended start : 4-8. |
| `slave_parallel_mode` | `optimistic` (10.5.1+), `conservative` (until 10.5.0) | One of `optimistic`, `conservative`, `aggressive`, `minimal`, `none`. |
| `slave_parallel_max_queued` | 131072 | Bytes queued per worker. Raise to 262144-1048576 if I/O thread blocks. |
| `slave_domain_parallel_threads` | 0 | Per-domain cap (multi-source). 0 = no cap. |
| `binlog_commit_wait_usec` | 0 | Primary : microseconds to wait before commit, to batch for conservative mode. |
| `binlog_commit_count` | 0 | Primary : transaction count threshold per batch. |

To change `slave_parallel_threads` or `slave_parallel_mode` at runtime, ALL replica connections must be stopped first :
```sql
STOP ALL SLAVES;
SET GLOBAL slave_parallel_threads = 8;
SET GLOBAL slave_parallel_mode    = optimistic;
START ALL SLAVES;
```

## 4. GTID Variables

| Variable | Scope | Default | Notes |
|----------|-------|---------|-------|
| `gtid_domain_id` | Global, Session | 0 | 32-bit unsigned. Identifies a logical replication stream. |
| `gtid_strict_mode` | Global | OFF | Set ON to reject any operation that could diverge binlog. |
| `gtid_slave_pos` | Global | empty | Last applied replicated GTID per domain. Backed by `mysql.gtid_slave_pos`. |
| `gtid_binlog_pos` | Global, read-only | empty | Last GTID written to local binlog. |
| `gtid_current_pos` | Global, read-only | empty | Union of `gtid_slave_pos` and `gtid_binlog_pos`. |
| `gtid_seq_no` | Session | 0 | Force next transaction's sequence number (advanced/replay use). |

GTID format : `domain-server-sequence`. Example : `0-1-1234`.

`MASTER_USE_GTID` values in `CHANGE MASTER` :
- `slave_pos` : track ONLY replicated GTIDs. Use on pure replicas.
- `current_pos` : track both replicated AND locally-logged GTIDs. Use on intermediate relays that also accept writes.
- `no` : disable GTID, fall back to MASTER_LOG_FILE + MASTER_LOG_POS positional replication.

## 5. CHANGE MASTER TO : Complete Grammar

```sql
-- single-source (positional)
CHANGE MASTER TO
  MASTER_HOST                   = 'host',
  MASTER_PORT                   = 3306,
  MASTER_USER                   = 'repl',
  MASTER_PASSWORD               = 'secret',
  MASTER_LOG_FILE               = 'primary1-bin.000001',
  MASTER_LOG_POS                = 568,
  MASTER_CONNECT_RETRY          = 10,
  MASTER_HEARTBEAT_PERIOD       = 30,
  MASTER_USE_GTID               = no;

-- single-source (GTID)
CHANGE MASTER TO
  MASTER_HOST                   = 'host',
  MASTER_USER                   = 'repl',
  MASTER_PASSWORD               = 'secret',
  MASTER_USE_GTID               = slave_pos;

-- multi-source (NAMED CONNECTION; quotes around channel name)
CHANGE MASTER 'analytics' TO
  MASTER_HOST                   = 'analytics-primary',
  MASTER_USER                   = 'repl',
  MASTER_PASSWORD               = 'secret',
  MASTER_USE_GTID               = slave_pos;

-- TLS additions
CHANGE MASTER TO
  MASTER_SSL                    = 1,
  MASTER_SSL_CA                 = '/etc/mysql/certs/ca.pem',
  MASTER_SSL_CERT               = '/etc/mysql/certs/client-cert.pem',
  MASTER_SSL_KEY                = '/etc/mysql/certs/client-key.pem',
  MASTER_SSL_VERIFY_SERVER_CERT = 1,
  MASTER_SSL_CIPHER             = 'ECDHE-RSA-AES256-GCM-SHA384',
  MASTER_SSL_CRL                = '/etc/mysql/certs/crl.pem';
```

The replica must be stopped (or never started) when calling `CHANGE MASTER`. The statement supersedes any prior call for the same channel.

## 6. Replication Control Statements

```sql
-- start/stop
START SLAVE;                           -- single-source baseline
START SLAVE 'analytics';               -- single channel
START ALL SLAVES;                      -- all channels (10.0+)
START SLAVE UNTIL MASTER_LOG_FILE='primary1-bin.000003', MASTER_LOG_POS=4;
START SLAVE UNTIL master_gtid_pos = '0-1-100';

STOP SLAVE;
STOP SLAVE 'analytics';
STOP ALL SLAVES;
STOP SLAVE IO_THREAD;                  -- only I/O thread, keep SQL thread running
STOP SLAVE SQL_THREAD;

-- reset
RESET SLAVE;                           -- clear replication state for default channel
RESET SLAVE 'analytics';
RESET SLAVE 'analytics' ALL;           -- delete channel permanently (10.0+)
RESET MASTER;                          -- erase binlog and reset gtid_binlog_pos
```

`SHOW REPLICA STATUS` / `START REPLICA` / `STOP REPLICA` are aliases introduced 10.5+ for inclusive terminology.

## 7. GRANT Syntax for Replication User

```sql
-- create user (10.6+)
CREATE USER 'repl'@'10.0.0.%' IDENTIFIED BY 'CHANGEME';

-- minimum privilege (mandatory)
GRANT REPLICATION SLAVE  ON *.* TO 'repl'@'10.0.0.%';

-- recommended additional (lets the user run SHOW MASTER STATUS, SHOW BINLOG EVENTS)
GRANT REPLICATION CLIENT ON *.* TO 'repl'@'10.0.0.%';

-- for tools that need to switch primary at runtime (mariabackup, MaxScale)
GRANT REPLICATION SLAVE ADMIN ON *.* TO 'repl'@'10.0.0.%';   -- 10.5.2+
```

NEVER grant `SUPER`, `FILE`, `PROCESS`, `RELOAD`, `SHUTDOWN`, `ALL` to the replication user.

## 8. Monitoring Queries

```sql
-- single-source state
SHOW SLAVE STATUS \G                                                    -- 10.x baseline
SHOW REPLICA STATUS \G                                                  -- 10.5+ alias

-- multi-source state
SHOW ALL SLAVES STATUS \G                                               -- 10.0+
SHOW SLAVE 'analytics' STATUS \G                                        -- 10.0+, per channel
SELECT @@global.default_master_connection;                              -- which channel default commands address

-- binlog position on primary
SHOW MASTER STATUS;
SHOW BINARY LOGS;
SHOW BINLOG EVENTS IN 'primary1-bin.000001' LIMIT 10;

-- GTID positions
SELECT @@global.gtid_binlog_pos;
SELECT @@global.gtid_slave_pos;
SELECT @@global.gtid_current_pos;
SELECT @@global.gtid_strict_mode;

-- semi-sync state
SHOW STATUS LIKE 'Rpl_semi_sync_master_status';
SHOW STATUS LIKE 'Rpl_semi_sync_master_clients';
SHOW STATUS LIKE 'Rpl_semi_sync_master_no_tx';                          -- count of txns that fell back to async
SHOW STATUS LIKE 'Rpl_semi_sync_master_tx_avg_wait_time';

-- parallel applier
SHOW STATUS LIKE 'Slave_running';
SELECT @@global.slave_parallel_threads, @@global.slave_parallel_mode;
SHOW PROCESSLIST;                                                       -- look for 'Slave_worker' threads
```

## 9. Key SHOW SLAVE STATUS Columns

| Column | What it tells you |
|--------|-------------------|
| `Slave_IO_Running` | `Yes` if reading binlog from primary. `No` => check `Last_IO_Error`. |
| `Slave_SQL_Running` | `Yes` if applying events. `No` => check `Last_SQL_Error`. |
| `Seconds_Behind_Master` | Apply lag in seconds. NULL when SQL thread is stopped. |
| `Last_IO_Error`, `Last_IO_Errno` | I/O failure (network, auth, binlog gone). |
| `Last_SQL_Error`, `Last_SQL_Errno` | Apply failure (schema diff, duplicate key, missing row). |
| `Using_Gtid` | `Slave_Pos`, `Current_Pos`, or `No`. |
| `Gtid_IO_Pos` | Last GTID received. |
| `Gtid_Slave_Pos` | Last GTID applied. |
| `Connection_name` | Channel name for multi-source. |
| `Slave_SQL_Running_State` | Free-text current applier state. |
| `Master_Log_File`, `Read_Master_Log_Pos` | Position the I/O thread has read up to. |
| `Relay_Master_Log_File`, `Exec_Master_Log_Pos` | Position the SQL thread has executed up to. |

## 10. Useful Sources

- Replication overview : https://mariadb.com/kb/en/replication/
- Setting up replication : https://mariadb.com/kb/en/setting-up-replication/
- CHANGE MASTER TO : https://mariadb.com/kb/en/change-master-to/
- Semi-sync : https://mariadb.com/kb/en/semisynchronous-replication/
- Parallel replication : https://mariadb.com/kb/en/parallel-replication/
- GTID : https://mariadb.com/kb/en/gtid/
- Multi-source : https://mariadb.com/kb/en/multi-source-replication/
- Binary log formats : https://mariadb.com/kb/en/binary-log-formats/
- Replication system variables : https://mariadb.com/kb/en/replication-and-binary-log-system-variables/
