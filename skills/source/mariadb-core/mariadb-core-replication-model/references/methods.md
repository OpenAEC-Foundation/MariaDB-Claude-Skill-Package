# methods.md : MariaDB Replication Variable Reference

Complete variable reference for replication model. Every entry is version-annotated and linked to the canonical MariaDB KB page. Sources verified 2026-05-19 against `mariadb.com/kb/en/`.

---

## 1. Semi-Synchronous Replication

Built into the server from MariaDB 10.3+. NEVER install a separate plug-in on 10.3 or later. KB : https://mariadb.com/kb/en/semisynchronous-replication/

| Variable | Scope | Default | Type | Description |
|----------|-------|---------|------|-------------|
| `rpl_semi_sync_master_enabled` | global | `OFF` | boolean | Primary-side switch. Set to `ON` to enable semi-sync waiting. (10.3+) |
| `rpl_semi_sync_master_timeout` | global | `10000` | ms | Wait time for replica ack before reverting to async. (10.3+) |
| `rpl_semi_sync_master_wait_no_slave` | global | `ON` | boolean | If `ON`, primary waits even when zero replicas are connected (up to timeout). If `OFF`, immediately falls back to async. (10.3+) |
| `rpl_semi_sync_master_wait_point` | global | `AFTER_SYNC` | enum | `AFTER_SYNC` waits for ack after binlog sync ; `AFTER_COMMIT` waits after engine commit. AFTER_SYNC is the modern recommendation. (10.3+) |
| `rpl_semi_sync_master_trace_level` | global | `32` | int | Bitmask for diagnostic logging. Increase only when debugging. (10.3+) |
| `rpl_semi_sync_slave_enabled` | global | `OFF` | boolean | Replica-side switch. Set to `ON` to allow this replica to ack the primary. (10.3+) |
| `rpl_semi_sync_slave_trace_level` | global | `32` | int | Bitmask for diagnostic logging on replica. (10.3+) |

Status variables on primary :

| Status | Healthy | Meaning |
|--------|---------|---------|
| `Rpl_semi_sync_master_status` | `ON` | Primary is currently in semi-sync mode (not fallen back to async). |
| `Rpl_semi_sync_master_clients` | `>= 1` | Number of replicas actively acknowledging. |
| `Rpl_semi_sync_master_tx_avg_wait_time` | low | Average time spent waiting for ack per committed transaction (microseconds). |
| `Rpl_semi_sync_master_no_tx` | low / 0 | Count of transactions that DID NOT receive an ack (timed out, went async). |

Status variables on replica :

| Status | Healthy | Meaning |
|--------|---------|---------|
| `Rpl_semi_sync_slave_status` | `ON` | This replica is currently acting as a semi-sync responder. |

NOTE : MariaDB uses the `slave` keyword in semi-sync variable names. NEVER expect `rpl_semi_sync_replica_*` aliases ; they do not exist as of 12.x.

---

## 2. Parallel Replication

KB : https://mariadb.com/kb/en/parallel-replication/. Requires both primary and replica MariaDB 10.0.5 or later.

| Variable | Scope | Default | Type | Description |
|----------|-------|---------|------|-------------|
| `slave_parallel_threads` | global | `0` | int | Worker-thread pool size on replica. `0` disables parallel apply (single SQL thread). Apply pool is shared across all master connections. (10.0.5+) |
| `slave_parallel_mode` | global | `optimistic` (10.5.1+), `conservative` (until 10.5.0) | enum | One of `optimistic`, `conservative`, `aggressive`, `minimal`, `none`. (10.1+ for full set) |
| `slave_parallel_max_queued` | global | `131072` | bytes | Max bytes of binlog events queued per worker. Larger queue smooths bursty workloads at the cost of memory. (10.0.5+) |
| `slave_domain_parallel_threads` | global | `0` | int | Per-domain cap inside the global worker pool. `0` means no cap (worker pool can dedicate all threads to one domain). Useful when one multi-source channel is much busier than another. (10.0+) |
| `slave_parallel_workers` | session/global | mirror of `slave_parallel_threads` | int | Compatibility alias ; prefer `slave_parallel_threads`. |

Modes :

| Mode | Conflict handling | Speed | Risk |
|------|-------------------|-------|------|
| `none` | single-threaded | slowest | none |
| `minimal` | parallel commit-phase only | small gain | none |
| `conservative` | parallel only for transactions the primary marked as group-commit-safe | medium | safe (preserves order) |
| `optimistic` | parallel for all DML ; row-level conflicts roll back and retry | high | retry storms under heavy contention |
| `aggressive` | parallel without conflict-avoidance heuristics | highest | data corruption risk if dependencies are misjudged |

ALWAYS test mode changes on a non-production replica. The default `optimistic` is correct for most OLTP workloads ; only switch when you have a measured reason.

---

## 3. Global Transaction IDs (GTID)

KB : https://mariadb.com/kb/en/gtid/. Format : `domain-server-sequence` (three integers).

| Variable | Scope | Default | Type | Description |
|----------|-------|---------|------|-------------|
| `gtid_domain_id` | global / session | `0` | uint32 | Identifies the logical replication stream this server writes to. Multi-source / multi-writer topologies use distinct domains. |
| `gtid_strict_mode` | global | `OFF` | boolean | When `ON`, the server rejects : (1) GTIDs with lower sequence than already binlogged ; (2) manual binlog entries with lower sequence ; (3) replicating missing GTIDs from the primary. ALWAYS enable on long-lived replicas. |
| `gtid_slave_pos` | global | (replica-side state) | string | Last GTID the SQL thread applied. Authoritative pointer for `MASTER_USE_GTID=slave_pos`. |
| `gtid_binlog_pos` | global | (server-side state) | string | Last GTID written to this server's binary log. Equals `gtid_slave_pos` on a pure replica that is also writing its own binlog. |
| `gtid_current_pos` | global | (computed) | string | Per-domain max of `gtid_slave_pos` and `gtid_binlog_pos`. Used by `MASTER_USE_GTID=current_pos`. |
| `gtid_binlog_state` | global | (computed) | string | The set of GTIDs seen in this server's binlog (one per domain). Useful for failover diagnostics. |
| `gtid_seq_no` | session | `0` | uint64 | Session-level override to set the sequence number of the next transaction. Used by tools (mariabackup, dump/load) when restoring GTID-marked dumps. |

`CHANGE MASTER ... MASTER_USE_GTID` accepts :

| Value | Behavior |
|-------|----------|
| `no` | binlog-position replication, no GTID tracking |
| `slave_pos` | resume from `gtid_slave_pos` (replica's applied position) |
| `current_pos` | resume from `gtid_current_pos` (max of slave_pos and binlog_pos) |

`gtid_strict_mode=ON` is the modern recommendation per the KB ; it catches divergence early instead of letting it propagate.

NOTE on cross-vendor : per the KB, "MariaDB can be a replica for a MySQL primary, but MySQL cannot be a replica for a MariaDB primary." The format `domain-server-sequence` is NOT interconvertible with MySQL's `uuid:seqno`. Treat MariaDB-to-MySQL as one-way dump-and-load.

---

## 4. Binary Log Variables

KB : https://mariadb.com/kb/en/binary-log-formats/ and https://mariadb.com/kb/en/replication-and-binary-log-system-variables/.

| Variable | Scope | Default | Type | Description |
|----------|-------|---------|------|-------------|
| `log_bin` | startup | OFF | path | Enables binary logging and sets file basename. Replication primaries MUST have it ON. |
| `server_id` | startup | `1` (10.x), pre-set | uint32 | Server identity in replication chains. ALWAYS unique across the topology. |
| `log_slave_updates` | startup | `OFF` | boolean | Write replicated events to this server's binlog too. Required on intermediate replicas in chains and on Galera nodes that need to feed downstream async replicas. |
| `binlog_format` | global / session | `MIXED` (10.2.4+), `STATEMENT` until 10.2.3 | enum | One of `STATEMENT`, `ROW`, `MIXED`. |
| `binlog_row_image` | global / session | `FULL` | enum | One of `FULL`, `MINIMAL`, `NOBLOB`. MariaDB 11.4+ adds `FULL_NODUP`. |
| `binlog_row_event_max_size` | startup | `8192` | bytes | Soft cap on per-row event size before splitting. |
| `sync_binlog` | global | `0` | int | `0` = OS decides ; `1` = fsync after every transaction (crash-safe) ; `N` = fsync every N writes. ALWAYS `1` paired with `innodb_flush_log_at_trx_commit=1` for crash-safe primary. |
| `expire_logs_days` | global | `0` (no expiry) | days | Binlog auto-expiry. Replaced by `binlog_expire_logs_seconds` on 10.5+ for finer control. |
| `binlog_expire_logs_seconds` | global | `0` | seconds | Modern fine-grained binlog expiry (10.5+). |
| `max_binlog_size` | startup | `1G` | bytes | Rotate when current binlog exceeds this size. |
| `binlog_checksum` | global | `CRC32` | enum | One of `NONE`, `CRC32`. Detects binlog corruption. Keep `CRC32`. |

Binary log format choice :

| Format | Behavior | Use when |
|--------|----------|----------|
| `STATEMENT` | Logs the SQL statement. Smallest but UNSAFE with non-deterministic functions. | Legacy compatibility only. ALWAYS migrate off in new topologies. |
| `ROW` | Logs the actual row changes. Largest but always safe. Required for Galera. | Galera ; any topology with non-deterministic SQL ; PITR with full row reconstruction. |
| `MIXED` | Statement by default, auto-switches to row for unsafe statements. Default since 10.2.4. | General OLTP ; balance of size and safety. |

`binlog_row_image` choice :

| Value | Per-row data logged | Use case |
|-------|---------------------|----------|
| `FULL` | All columns, before + after image | PITR, downstream replication, debugging |
| `NOBLOB` | All columns before + after EXCEPT unchanged BLOB/TEXT | reduce binlog volume on tables with large BLOBs |
| `MINIMAL` | Only the PK in before image, only changed columns in after image | minimum binlog volume ; LOSES information needed for full PITR |
| `FULL_NODUP` (11.4+) | Same as FULL but skips duplicate before-image columns | size reduction without losing reconstruction capability |

ALWAYS keep `FULL` for any topology that does PITR. `MINIMAL` is for high-volume write workloads with no PITR requirement.

---

## 5. Multi-Source Replication

KB : https://mariadb.com/kb/en/multi-source-replication/. Available since the 10.0 series.

Syntax uses NAMED CONNECTIONS, not `FOR CHANNEL` :

```sql
CHANGE MASTER ['connection_name'] TO ...
START SLAVE  ['connection_name']
STOP SLAVE   ['connection_name']
RESET SLAVE  ['connection_name'] [ALL]
SHOW SLAVE  ['connection_name'] STATUS
SHOW ALL SLAVES STATUS
```

Connection names are case-insensitive. The default unnamed connection uses an empty string `''`.

Per-channel session variables :

| Variable | Scope | Default | Description |
|----------|-------|---------|-------------|
| `default_master_connection` | session | `''` | Selects which named connection subsequent commands operate on without explicit naming. |

Status command :

| Command | Output | When to use |
|---------|--------|-------------|
| `SHOW SLAVE STATUS` | Default connection only | Single-source topology. |
| `SHOW SLAVE 'name' STATUS` | One specific channel | Targeted diagnostics. |
| `SHOW ALL SLAVES STATUS` | All channels in one resultset, with `Connection_name` column | Multi-source dashboard. |

`SHOW REPLICA` / `SHOW ALL REPLICAS` aliases exist from 10.5+.

---

## 6. SHOW SLAVE STATUS Key Columns

KB : https://mariadb.com/kb/en/show-slave-status/.

| Column | Type | Healthy | Failure indicator |
|--------|------|---------|-------------------|
| `Slave_IO_State` | string | "Waiting for master to send event" | "Connecting to master" stuck, or empty |
| `Slave_IO_Running` | enum | `Yes` | `No` (connection broken), `Connecting` (in retry) |
| `Slave_SQL_Running` | enum | `Yes` | `No` (apply error, see Last_SQL_Error) |
| `Seconds_Behind_Master` | int | `0` or small steady-state | rising trend (lag) or `NULL` (IO thread down) |
| `Last_IO_Errno` / `Last_IO_Error` | int / string | `0` / empty | non-zero with error message |
| `Last_SQL_Errno` / `Last_SQL_Error` | int / string | `0` / empty | non-zero with error message |
| `Master_Log_File` / `Read_Master_Log_Pos` | string / int | progressing | static when primary is writing |
| `Relay_Master_Log_File` / `Exec_Master_Log_Pos` | string / int | catching up to Read_Master | gap = lag |
| `Gtid_IO_Pos` | string | tracks primary `gtid_binlog_pos` | gap = IO-thread lag |
| `Using_Gtid` | enum | `Slave_Pos` / `Current_Pos` | `No` if attached by binlog position |
| `Connection_name` | string | name of channel | only present in `SHOW ALL SLAVES STATUS` |

`Seconds_Behind_Master` measures the timestamp on the event the SQL thread is APPLYING relative to the replica's wall clock. It drops to `0` between transactions and lies about true lag. Cross-check with `pt-heartbeat` or an equivalent token-based heartbeat table.

---

## 7. TLS Variables for Replication

KB : https://mariadb.com/kb/en/change-master-to/.

| Variable / clause | Where | Purpose |
|-------------------|-------|---------|
| `MASTER_SSL=1` | `CHANGE MASTER TO` | Enable TLS on this connection. |
| `MASTER_SSL_CA` | `CHANGE MASTER TO` | Path to CA certificate. |
| `MASTER_SSL_CERT` | `CHANGE MASTER TO` | Client certificate path. |
| `MASTER_SSL_KEY` | `CHANGE MASTER TO` | Client private key path. |
| `MASTER_SSL_VERIFY_SERVER_CERT=1` | `CHANGE MASTER TO` | ALWAYS set this with TLS ; otherwise TLS gives no MITM protection. |
| `MASTER_SSL_CIPHER` | `CHANGE MASTER TO` | Restrict to specific cipher suite. |
| `MASTER_SSL_CRL` | `CHANGE MASTER TO` | Certificate revocation list path. |

TLS is configured on the REPLICA side. The primary just needs `ssl=ON` and a server cert.

---

## Source Verification (2026-05-19)

| KB page | Verified facts |
|---------|----------------|
| https://mariadb.com/kb/en/semisynchronous-replication/ | semi-sync built-in 10.3+, variable names, default timeout 10000 ms, async fallback on timeout |
| https://mariadb.com/kb/en/parallel-replication/ | mode names + defaults, 10.0.5 minimum, slave_domain_parallel_threads behavior |
| https://mariadb.com/kb/en/gtid/ | format domain-server-sequence, variable names, gtid_strict_mode behavior, MariaDB-replica-to-MySQL not supported |
| https://mariadb.com/kb/en/binary-log-formats/ | three formats, MIXED auto-switches |
| https://mariadb.com/kb/en/replication-and-binary-log-system-variables/ | binlog_format default MIXED since 10.2.4, binlog_row_image FULL/MINIMAL/NOBLOB, FULL_NODUP 11.4+, sync_binlog default 0 |
| https://mariadb.com/kb/en/multi-source-replication/ | named-connection syntax, case-insensitive, SHOW ALL SLAVES STATUS |
| https://mariadb.com/kb/en/show-slave-status/ | column names, REPLICA aliases 10.5+ |
