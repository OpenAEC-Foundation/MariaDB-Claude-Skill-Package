# Galera Cluster : Methods Reference

Complete reference for the wsrep system variables, status variables, SST method comparison, weighted-quorum semantics, and garbd arbitrator setup.

## wsrep System Variables (configuration)

Verified against `mariadb.com/kb/en/galera-cluster-system-variables/` (2026-05-19).

| Variable | Default | Scope | Dynamic | Purpose |
|----------|---------|-------|---------|---------|
| `wsrep_on` | `OFF` | Global, Session | Yes | Master switch for wsrep replication. If `OFF`, the provider cannot load and the node cannot join the cluster. Set `ON` on every Galera node. |
| `wsrep_provider` | none | Global | No | Absolute path to the Galera provider library. Typically `/usr/lib/galera/libgalera_smm.so` (Debian/Ubuntu) or `/usr/lib64/galera/libgalera_smm.so` (RHEL/CentOS). Galera-4 ships with MariaDB 10.4+. |
| `wsrep_provider_options` | empty | Global | No | Semicolon-separated provider tuning string. Common keys : `gcache.size=2G`, `pc.weight=2`, `gcs.fc_limit=64`, `evs.send_window=4`, `evs.user_send_window=2`. |
| `wsrep_cluster_address` | none | Global | No | `gcomm://node1,node2,node3` for joining ; `gcomm://` alone bootstraps a new cluster (use `galera_new_cluster` wrapper instead in production). |
| `wsrep_cluster_name` | `my_wsrep_cluster` | Global | Yes | Logical cluster name. MUST be identical on every node in the same cluster. |
| `wsrep_node_address` | auto | Global | No | This node's IP and optional port (default 4567). Auto-detection is unreliable in cloud and container environments ; set explicitly. |
| `wsrep_node_name` | hostname | Global | Yes | Logical node identifier for logs and donor-preference settings. Set explicitly per node. |
| `wsrep_sst_method` | `rsync` | Global | Yes | SST tool. Valid : `mariabackup`, `rsync`, `rsync_wan`, `mysqldump`, `xtrabackup-v2`. Production : `mariabackup`. |
| `wsrep_sst_auth` | empty | Global | Yes | `user:password` for the SST donor authentication. Masked in `SHOW VARIABLES` output. |
| `wsrep_sst_donor` | empty | Global | Yes | Preferred donor node-name list for SST. Useful to prevent the joiner from picking a busy node. |
| `wsrep_notify_cmd` | empty | Global | Yes | Script invoked on state changes (membership, quorum, sync). Used by external monitoring. |

Required server-level companions :

| Variable | Required Value | Reason |
|----------|----------------|--------|
| `binlog_format` | `ROW` | Galera replicates write-sets, not statements. `STATEMENT` is unsafe and rejected. |
| `default_storage_engine` | `InnoDB` | Galera replicates only InnoDB tables. MyISAM and Aria writes are local-only. |
| `innodb_autoinc_lock_mode` | `2` (interleaved) | Mode 1 (consecutive) serialises AUTO_INCREMENT inserts cluster-wide ; Galera cannot guarantee this. Mode 2 lets each node allocate from its own sub-range. |
| `innodb_doublewrite` | `1` (ON, default) | Crash-safe writes. Galera nodes can crash independently ; the doublewrite buffer protects against torn pages. |
| `query_cache_size` | `0` | The query cache is single-server. With Galera, cached results on one node go stale on a write to another node. |
| `query_cache_type` | `OFF` | Same reason as above. |

## wsrep Provider Options (wsrep_provider_options)

Verified against MariaDB KB Galera-cluster pages (L-003 : upstream `galeracluster.com` is bot-blocked, MariaDB KB is canonical).

| Option | Default | Purpose |
|--------|---------|---------|
| `gcache.size` | `128M` | Size of the in-memory ring buffer holding recent write-sets. Larger gcache = longer IST window (joiner can do incremental transfer instead of full SST). Set 1-2 GB on busy clusters. |
| `gcache.page_size` | `128M` | Size of the on-disk gcache page files. Set above expected single-transaction write-set size. |
| `pc.weight` | `1` | This node's vote weight in the Primary Component algorithm. Set higher on the larger-DC nodes for asymmetric deployments to ensure the larger DC survives a WAN split. |
| `pc.ignore_sb` | `false` | If `true`, ignores split-brain. NEVER set true on a production cluster. |
| `pc.ignore_quorum` | `false` | If `true`, accepts NON_PRIMARY component. NEVER set true. |
| `pc.bootstrap` | `false` | Dynamically force this node to bootstrap a new Primary Component (manual recovery from total cluster failure). |
| `gcs.fc_limit` | `16` | Flow-control limit : nodes pause writes when the receive queue grows beyond this. Increase on fast networks with slow appliers. |
| `evs.suspect_timeout` | `PT5S` | How long a node is "suspect" before being declared dead. |
| `evs.inactive_timeout` | `PT15S` | How long after suspect-timeout before forced eviction. |

## wsrep Status Variables (monitoring)

Verified against `mariadb.com/kb/en/galera-cluster-status-variables/` (2026-05-19).

| Variable | Healthy Value | Alert Condition |
|----------|---------------|------------------|
| `wsrep_cluster_size` | matches deployed node count (3, 5, etc.) | Lower than expected = node(s) disconnected |
| `wsrep_cluster_status` | `Primary` | `Non-Primary` = quorum lost ; `Disconnected` = node not connected to cluster |
| `wsrep_local_state` | `4` (Synced) | Any value other than 4 sustained > 30 s |
| `wsrep_local_state_comment` | `Synced` | `Joining`, `Donor/Desynced`, `Joined`, `SST sender`, `SST receiver` are transient ; sustained = problem |
| `wsrep_local_cert_failures` | low and stable | Spike = hot-row contention ; investigate write patterns |
| `wsrep_local_recv_queue` | near 0 | Sustained > 10 = appliers cannot keep up ; flow-control will engage |
| `wsrep_local_recv_queue_avg` | near 0 | Average queue length since last query |
| `wsrep_ready` | `ON` | `OFF` = node cannot serve transactions |
| `wsrep_connected` | `ON` | `OFF` = node disconnected from cluster |
| `wsrep_local_state_uuid` | matches `wsrep_cluster_state_uuid` | Mismatch = node out of sync |
| `wsrep_apply_oooe` | low | Fraction of write-sets applied out of order ; high = network jitter |
| `wsrep_cert_deps_distance` | 50-200 typical | Average parallelism in certification ; very low = serial workload |
| `wsrep_flow_control_paused` | < 0.1 | Fraction of time the node paused for flow-control ; high = overloaded |

`wsrep_local_state` numeric values :
- `1` Joining
- `2` Donor/Desynced
- `3` Joined
- `4` Synced (healthy)

## SST Method Comparison

Verified against `mariadb.com/docs/galera-cluster/high-availability/state-snapshot-transfers-ssts-in-galera-cluster/introduction-to-state-snapshot-transfers-ssts.md` (2026-05-19).

| Method | Donor Blocking | Speed | Encryption Support | Version | Recommended Use |
|--------|----------------|-------|--------------------|---------|------------------|
| `mariabackup` | NON-BLOCKING (hot backup) | Fast | Yes (GTID + data-at-rest) | 10.1.26+ / 10.2.10+ | PRODUCTION DEFAULT |
| `rsync` | BLOCKED (read lock entire transfer) | Fastest raw throughput | Yes | All versions | Small clusters, maintenance window |
| `rsync_wan` | BLOCKED (read lock entire transfer) | Slower than rsync, lower bandwidth | Yes | All versions | Cross-WAN SST when bandwidth is constrained |
| `mysqldump` | BLOCKED (read lock entire transfer) | SLOWEST (logical dump) | Yes | All versions | Cross-major-version upgrades only |
| `xtrabackup-v2` | NON-BLOCKING | Fast | Yes | All versions | Legacy clusters with Percona XtraBackup ; prefer mariabackup |

### IST (Incremental State Transfer)

IST applies missing write-sets from the donor's `gcache` instead of a full data copy. It runs automatically when :
1. The joiner has a `grastate.dat` with a valid `seqno`.
2. The donor's `gcache` still holds every write-set since the joiner's seqno.

If the gap is larger than `gcache.size`, the cluster falls back to SST. Size `gcache.size` to cover the longest expected outage window (network blip, restart, brief node-down). 2 GB is a common production value.

## Garbd Arbitrator

`garbd` (Galera Arbitrator Daemon) is a zero-data node that participates in quorum voting. Used to give an even-node cluster an odd vote-count without paying for a full data node.

### Installation

```bash
# Debian / Ubuntu
apt install galera-arbitrator-4

# RHEL / CentOS
dnf install galera-arbitrator-4
```

### Run as daemon

```bash
garbd --address gcomm://10.0.0.11:4567,10.0.0.12:4567 \
      --group prod_cluster \
      --log /var/log/garbd.log \
      --daemon
```

Or via systemd : edit `/etc/default/garbd` then `systemctl enable --now garbd`.

`garbd` consumes minimal CPU and RAM. Place it in a THIRD failure domain (different rack, different availability zone, different DC) so a single failure cannot take both data nodes and the arbitrator at once.

## Bootstrap and Recovery Commands

| Command | When to Use |
|---------|-------------|
| `galera_new_cluster` | Bootstrap the FIRST node of a brand-new cluster, or recover from total cluster shutdown (run on the node with the highest seqno in grastate.dat). |
| `mariadbd --wsrep-new-cluster` | Manual equivalent of `galera_new_cluster`. Same effect. |
| `systemctl start mariadb` | Standard start for nodes joining an EXISTING cluster. NEVER use `galera_new_cluster` once the cluster exists. |
| `SET GLOBAL wsrep_provider_options="pc.bootstrap=true";` | Force this node to bootstrap a new Primary Component dynamically. Recovery procedure when the cluster is stuck in NON_PRIMARY and all nodes are otherwise healthy. |
| `SET GLOBAL wsrep_on=OFF; ...; SET GLOBAL wsrep_on=ON;` | Temporarily disable replication for a local-only operation (rare, dangerous, avoid). |

## Grastate.dat Recovery

`/var/lib/mysql/grastate.dat` records the last known cluster UUID and seqno. After a crash :

| seqno value | Meaning |
|-------------|---------|
| `-1` | Node crashed mid-transaction. Run `mariadbd --wsrep-recover` to extract the real seqno from the InnoDB log, then update grastate.dat. |
| `>= 0` | Clean shutdown. Node can rejoin the cluster directly. |
| `safe_to_bootstrap: 1` | This node is safe to bootstrap from (highest seqno at shutdown). |
| `safe_to_bootstrap: 0` | Bootstrapping from this node would lose data ; pick another node with `safe_to_bootstrap: 1`. |

After a total cluster shutdown : compare grastate.dat across all nodes, find the highest seqno with `safe_to_bootstrap: 1`, run `galera_new_cluster` on THAT node, start the rest normally.

## Privilege Requirements

| Privilege | Used By | On |
|-----------|---------|-----|
| `RELOAD` | mariabackup SST | `*.*` |
| `PROCESS` | mariabackup SST | `*.*` |
| `LOCK TABLES` | mariabackup SST | `*.*` |
| `BINLOG MONITOR` (10.5+) | mariabackup SST | `*.*` (replaces legacy `REPLICATION CLIENT`) |
| `REPLICATION CLIENT` | mariabackup SST (pre-10.5) | `*.*` |
| `CREATE TABLESPACE` | mariabackup with encryption | `*.*` |
