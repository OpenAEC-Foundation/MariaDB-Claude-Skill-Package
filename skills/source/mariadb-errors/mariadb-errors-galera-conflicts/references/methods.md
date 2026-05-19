# Galera Cluster Conflicts : Methods Reference

Complete reference for status variables, system variables, provider options, error codes, and the certification flow that produce or diagnose Galera-specific conflicts.

## Galera Error-Code Matrix

| Error number | Symbolic name | SQLSTATE | Triggered when | Application response |
|--------------|---------------|----------|-----------------|----------------------|
| 1213 | `ER_LOCK_DEADLOCK` | 40001 | Certification fails at COMMIT on the originating node | Retry full transaction with backoff |
| 1205 | `ER_LOCK_WAIT_TIMEOUT_EXCEEDED` | HY000 | A row-lock was held longer than `innodb_lock_wait_timeout`. NOT a Galera-specific path, but can appear under cluster write storm | Find blocker, do not blind-retry. See mariadb-errors-deadlocks. |
| 1180 | `ER_ERROR_DURING_COMMIT` | HY000 | Lower-level commit failure ; can wrap a wsrep failure in some paths | Inspect server log for wsrep context, retry if path is certification |
| 1614 | `ER_QUERY_INTERRUPTED` | 70100 | In-flight local transaction killed by an incoming applier (brute-force abort) | Treat identically to 1213 : retry full transaction |
| 1290 | `ER_OPTION_PREVENTS_STATEMENT` | HY000 | Attempting writes on a `NON_PRIMARY` node (read-only) | Recover quorum first ; do not retry |

The pair of errors that signal certification conflict is `1213` (originating node) and `1614` (local in-flight aborted by remote applier). Both are retry candidates.

## wsrep Status Variables (complete reference)

| Variable | Type | Scope | Description |
|----------|------|-------|-------------|
| `wsrep_local_cert_failures` | monotonic counter | global | Total local transactions that failed the certification test and were rolled back |
| `wsrep_local_bf_aborts` | monotonic counter | global | Total high-priority local transactions aborted by replication applier threads |
| `wsrep_cluster_status` | gauge enum | global | `PRIMARY` (quorum present), `NON_PRIMARY` (quorum lost), `DISCONNECTED` |
| `wsrep_cluster_size` | gauge | global | Number of nodes currently in the primary component |
| `wsrep_cluster_state_uuid` | gauge UUID | global | UUID identifying the cluster's current state ; mismatch across nodes = split-brain history |
| `wsrep_cluster_conf_id` | monotonic counter | global | Increments on every cluster-membership change (node join, node leave, partition) |
| `wsrep_local_state` | gauge int | global | Internal Galera FSM state number |
| `wsrep_local_state_comment` | gauge string | global | Human-readable state (`Synced`, `Donor/Desynced`, `Joining`, `Joined`, `Initialized`) |
| `wsrep_local_state_uuid` | gauge UUID | global | UUID identifying this node's state |
| `wsrep_last_committed` | gauge | global | Sequence number (seqno) of the last committed write-set on this node |
| `wsrep_local_recv_queue` | gauge | global | Current length of the received-but-not-yet-applied queue |
| `wsrep_local_recv_queue_avg` | gauge | global | Average length of the receive queue (back-pressure indicator) |
| `wsrep_local_send_queue` | gauge | global | Current length of the outgoing replication queue |
| `wsrep_flow_control_paused` | gauge | global | Fraction of time the node spent paused under flow control (0.0 to 1.0) |
| `wsrep_flow_control_sent` | monotonic counter | global | Total flow-control PAUSE messages this node has sent |
| `wsrep_flow_control_recv` | monotonic counter | global | Total flow-control messages this node has received |
| `wsrep_ready` | gauge boolean | global | `ON` if the node is ready to accept transactions, `OFF` otherwise |

ALWAYS alert on RATES, not absolute values, for monotonic counters. A baseline of cert-failures is normal under multi-master write load.

```sql
-- 10.6+ : full Galera health snapshot
SELECT
  VARIABLE_NAME, VARIABLE_VALUE
FROM information_schema.GLOBAL_STATUS
WHERE VARIABLE_NAME IN (
  'WSREP_CLUSTER_STATUS', 'WSREP_CLUSTER_SIZE', 'WSREP_LOCAL_STATE_COMMENT',
  'WSREP_LOCAL_CERT_FAILURES', 'WSREP_LOCAL_BF_ABORTS',
  'WSREP_LOCAL_RECV_QUEUE_AVG', 'WSREP_FLOW_CONTROL_PAUSED',
  'WSREP_LAST_COMMITTED', 'WSREP_READY'
);
```

## wsrep System Variables (relevant to conflict diagnosis)

| Variable | Default | Scope | Dynamic | Purpose |
|----------|---------|-------|---------|---------|
| `wsrep_on` | `OFF` | Global, Session | Yes | Whether wsrep replication is enabled. Must be ON to participate in the cluster. |
| `wsrep_provider` | none | Global | No | Path to the wsrep provider library (typically `/usr/lib/libgalera_smm.so` or `/usr/lib/galera/libgalera_smm.so`). |
| `wsrep_provider_options` | empty | Global | Yes (most options) | Semicolon-separated provider option string. Holds `pc.weight`, `pc.ignore_sb`, `gcache.size`, etc. |
| `wsrep_cluster_address` | none | Global | No | `gcomm://host1,host2,host3` style cluster membership address. |
| `wsrep_retry_autocommit` | `1` | Global | No | Number of times an auto-committed query is internally retried on cluster conflict before returning an error to the client. Range 0 to 10000. |
| `wsrep_max_ws_size` | `2147483647` | Global | Yes | Maximum permitted write-set size in bytes. Default is roughly 2 GB. |
| `wsrep_max_ws_rows` | `0` | Global | Yes | Maximum rows per write-set. Default 0 means no limit. |
| `wsrep_sync_wait` | `0` | Session | Yes | Causality bitmask. Non-zero forces a node to wait for replication catch-up before executing the next statement. |
| `wsrep_certify_nonPK` | `ON` | Global | Yes | Certify transactions for tables with no primary key. The KB notes this is supported but not recommended ; design tables with explicit primary keys. |
| `wsrep_sst_method` | `mariabackup` (recommended) | Global | No | Method used for SST. ALWAYS use `mariabackup` in production. |
| `wsrep_sst_auth` | none | Global | No | `user:password` for SST authentication when method requires it. |

`wsrep_retry_autocommit` is silently helpful for fully-auto-committed single-statement transactions : the server itself retries up to N times before bubbling 1213 to the client. It does NOT cover multi-statement transactions ; those still require application-layer retry.

## Provider Options (via wsrep_provider_options)

The provider options control the Galera library directly. Set them via `SET GLOBAL wsrep_provider_options='key1=value1;key2=value2'`. Some are dynamic, some require a restart.

### Primary Component (PC) options

| Option | Default | Dynamic | Purpose |
|--------|---------|---------|---------|
| `pc.weight` | `1` | Yes | Node weight used in quorum calculation. Range : positive integers. Set higher on the larger DC in asymmetric layouts. |
| `pc.ignore_sb` | `false` | Yes | If true, the node continues to accept writes when disconnected from the cluster. DANGEROUS in multi-master ; can cause split-brain data divergence. |
| `pc.ignore_quorum` | `false` | Yes | If true, the node ignores quorum loss and continues operating. Even more dangerous than `pc.ignore_sb`. |
| `pc.recovery` | `true` | No | Persist primary-component state to disk for automatic recovery on cluster-wide restart. |
| `pc.wait_prim` | `true` | No | New nodes wait for the primary component to exist before joining (prevents accidental bootstrap). |
| `pc.bootstrap` | `false` (set as one-shot action) | Yes | Set to `true` to bootstrap a primary component from this node when all nodes are NON_PRIMARY. |

### gcache (write-set ring buffer) options

| Option | Default | Dynamic | Purpose |
|--------|---------|---------|---------|
| `gcache.size` | `128M` | No | Ring-buffer size for write-set caching. Determines IST window. Restart required. |
| `gcache.name` | `galera.cache` | No | File name for the gcache. |
| `gcache.dir` | datadir | No | Directory for the gcache file. |
| `gcache.recover` | `no` | No | Recover gcache contents on restart (10.4.4+). |
| `gcache.page_size` | `128M` | No | Page size for gcache overflow pages. |

Size `gcache.size` for the largest expected outage window : (write-rate in bytes/sec) × (outage budget in seconds) × safety factor of 1.5.

### EVS (Extended Virtual Synchrony) options

| Option | Default | Dynamic | Purpose |
|--------|---------|---------|---------|
| `evs.suspect_timeout` | `PT5S` | No | Time after which a non-responding node is suspected dead. ISO 8601 duration format. |
| `evs.inactive_timeout` | `PT15S` | No | Time after which an unresponsive node is removed from the cluster. |
| `evs.keepalive_period` | `PT1S` | No | Keepalive ping interval between nodes. |
| `evs.join_retrans_period` | `PT1S` | No | Retransmit interval during cluster-join. |

### GCS (Group Communication System) options

| Option | Default | Dynamic | Purpose |
|--------|---------|---------|---------|
| `gcs.fc_limit` | `16` | Yes | Flow-control threshold. When `wsrep_local_recv_queue` reaches this, the node sends FC_PAUSE to slow other nodes. |
| `gcs.fc_factor` | `1.0` | Yes | Multiplier applied to `gcs.fc_limit` to determine when to resume after pause. |
| `gcs.fc_master_slave` | `no` | No | If `yes`, only the master sends flow-control messages (master-slave deployments). |

```sql
-- 10.6+ : set multiple provider options at runtime
SET GLOBAL wsrep_provider_options =
  'pc.weight=2;pc.ignore_sb=OFF;gcs.fc_limit=32;evs.suspect_timeout=PT10S';

-- 10.6+ : inspect current values
SHOW VARIABLES LIKE 'wsrep_provider_options';
```

Non-dynamic options (`gcache.size`, `pc.recovery`, EVS timeouts) must go in `my.cnf` and require a node restart.

## Certification Flow (mental model)

```
Client                   Local Node                Other Nodes
  |                          |                          |
  |--BEGIN; ...statements--->|                          |
  |                          |  (local-only, no replication yet)
  |                          |                          |
  |--COMMIT----------------->|                          |
  |                          |--write-set broadcast---->|
  |                          |                          |
  |                          |<--certification result---|
  |                          |  (each node certifies against its in-flight
  |                          |   and recently-committed write-sets)
  |                          |                          |
  |                          | If all nodes certify OK :
  |                          |   apply locally, commit, return 0 to client
  |                          |                          |
  |                          | If any node rejects :
  |                          |   rollback locally, return ER_LOCK_DEADLOCK (1213)
  |<--OK or 1213-------------|                          |
```

Key invariants :

1. The application sees the error only at the `COMMIT` boundary.
2. The error originates from the LOCAL node, not from the conflicting peer.
3. A successful `COMMIT` on Galera means EVERY node has certified and accepted the write-set ; the cluster is synchronously consistent at this point.

The brute-force abort path (`wsrep_local_bf_aborts`) is the symmetric case : an incoming applier wants to apply a write-set that conflicts with a local IN-FLIGHT (not yet committed) transaction. The local in-flight is killed mid-statement and the client sees `ER_QUERY_INTERRUPTED` (1614).

## Hot-Row Redesign Patterns

A single primary key contended across nodes is the #1 cause of sustained certification failures. The redesign options :

### 1. Sharded counter (N-row replacement)

```sql
-- 10.6+ : sharded counter
CREATE TABLE counter_sharded (
  shard TINYINT UNSIGNED NOT NULL,
  v BIGINT NOT NULL DEFAULT 0,
  PRIMARY KEY (shard)
) ENGINE=InnoDB;

-- Initialize : 16 shards
INSERT INTO counter_sharded (shard, v)
SELECT seq.seq, 0 FROM seq_0_to_15 seq;

-- Writer : pick a shard at random
UPDATE counter_sharded SET v = v + 1 WHERE shard = FLOOR(RAND() * 16);

-- Reader : SUM across shards
SELECT SUM(v) AS total FROM counter_sharded;
```

### 2. SEQUENCE object (gap-tolerant, no certification race)

```sql
-- 10.6+ : SEQUENCE for monotonically-increasing IDs
CREATE SEQUENCE order_id_seq START WITH 1 INCREMENT BY 1 CACHE 100;

-- Writer : NEXTVAL has no certification contention
INSERT INTO orders (id, customer, ts) VALUES (NEXTVAL(order_id_seq), ?, NOW());
```

### 3. Insert-only audit table

```sql
-- 10.6+ : audit_log as insert-only replaces UPDATE counter
CREATE TABLE audit_log (
  id BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY,
  account_id BIGINT NOT NULL,
  action VARCHAR(64) NOT NULL,
  ts DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
) ENGINE=InnoDB;

-- Writer : INSERT only, never UPDATE
INSERT INTO audit_log (account_id, action) VALUES (?, ?);

-- Reader : COUNT or aggregate
SELECT account_id, COUNT(*) FROM audit_log GROUP BY account_id;
```

### 4. INSERT ... ON DUPLICATE KEY UPDATE with hashed prefix

```sql
-- 10.6+ : sharded INSERT...ODKU
CREATE TABLE rl_buckets (
  shard TINYINT UNSIGNED NOT NULL,
  rl_key VARCHAR(128) NOT NULL,
  cnt BIGINT NOT NULL DEFAULT 0,
  ts DATETIME(6) NOT NULL,
  PRIMARY KEY (shard, rl_key)
) ENGINE=InnoDB;

INSERT INTO rl_buckets (shard, rl_key, cnt, ts)
  VALUES (CRC32(?) MOD 16, ?, 1, NOW(6))
  ON DUPLICATE KEY UPDATE cnt = cnt + 1, ts = NOW(6);
```

The pattern : convert one contended row into N rows keyed by a deterministic shard. Writes scatter across shards ; reads aggregate. The trade-off is one extra GROUP BY at read time.

## SST Method Comparison

| Method | Donor-blocking | Compression | InnoDB-aware | Use case |
|--------|----------------|-------------|--------------|----------|
| `mariabackup` | NO | Optional via `--compress` | Yes, native | All production deployments |
| `rsync` | YES (read-lock during transfer) | NO | No | Avoid ; only for legacy compatibility |
| `mysqldump` | YES (logical, very slow) | NO | No | Never on production |

```ini
# /etc/my.cnf.d/galera.cnf : recommended SST configuration
[mariadb]
wsrep_sst_method = mariabackup
wsrep_sst_auth = sstuser:strong-password-here
```

The SST user requires `RELOAD`, `PROCESS`, `LOCK TABLES`, and `REPLICATION CLIENT` privileges to enable `mariabackup` to read the donor state without ALL PRIVILEGES.

```sql
-- 10.6+ : create the minimum-privilege SST user
CREATE USER 'sstuser'@'localhost' IDENTIFIED BY 'strong-password-here';
GRANT RELOAD, PROCESS, LOCK TABLES, REPLICATION CLIENT ON *.* TO 'sstuser'@'localhost';
FLUSH PRIVILEGES;
```

## garbd (Galera Arbitrator) Reference

`garbd` is a quorum-only daemon. It participates in voting but stores no data. Use it on a third host to give a 2-node MariaDB Galera cluster a 3-vote quorum.

```bash
# Install (Debian/Ubuntu)
apt install galera-arbitrator-4

# /etc/default/garbd
GALERA_GROUP="prod_cluster"
GALERA_NODES="node1.example.com:4567,node2.example.com:4567"
LOG_FILE="/var/log/garbd.log"

# Start
systemctl enable --now garbd
```

Key properties :

- Stores NO data ; cannot be promoted to a full node.
- Does NOT count toward `wsrep_cluster_size` in the same way as a full node ; it adds to the quorum vote.
- Resource cost : tiny (a few MB of RAM, negligible CPU).
- Placement : a host independent of both data nodes. NEVER on the same physical host as a Galera node.

## Sources

- `mariadb.com/kb/en/galera-cluster-status-variables/` : `wsrep_local_cert_failures`, `wsrep_local_bf_aborts`, `wsrep_cluster_status`, `wsrep_local_state`, `wsrep_local_state_comment`.
- `mariadb.com/kb/en/galera-cluster-system-variables/` : `wsrep_on`, `wsrep_provider`, `wsrep_provider_options`, `wsrep_cluster_address`, `wsrep_retry_autocommit`, `wsrep_max_ws_size`, `wsrep_sync_wait`, `wsrep_certify_nonPK`.
- `mariadb.com/docs/galera-cluster/reference/wsrep-variable-details/wsrep_provider_options` : `pc.weight`, `pc.ignore_sb`, `pc.recovery`, `gcache.size`, `evs.suspect_timeout`.
- `mariadb.com/docs/server/reference/error-codes/mariadb-error-codes-1200-to-1299` : 1213 `ER_LOCK_DEADLOCK`, 1205 `ER_LOCK_WAIT_TIMEOUT`, 1290 `ER_OPTION_PREVENTS_STATEMENT`.
- Vooronderzoek Cluster-3 §2 : certification-based replication semantics.
- LESSONS L-003 : `galeracluster.com` upstream bot-blocked ; MariaDB KB pages are canonical.
