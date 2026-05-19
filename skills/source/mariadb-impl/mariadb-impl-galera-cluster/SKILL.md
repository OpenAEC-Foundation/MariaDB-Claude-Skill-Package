---
name: mariadb-impl-galera-cluster
description: >
  Use when setting up a Galera multi-master cluster, sizing the cluster, choosing an SST method, debugging split-brain or certification failures, or scaling reads off Galera.
  Prevents the common mistake of running 2-node Galera (split-brain risk), using mysqldump SST in production (slow and blocking), running heterogeneous MariaDB versions in one cluster, or treating COMMIT-time deadlocks as application bugs instead of retrying.
  Covers 3-node minimum quorum, wsrep_provider (galera-4 since 10.4+), gcomm:// addresses, SST via mariabackup, IST incremental recovery, weighted quorum via pc.weight, garbd arbitrator, certification-based replication semantics (deadlock only at COMMIT), and wsrep_local_cert_failures monitoring.
  Keywords: Galera cluster, wsrep, wsrep_provider, gcomm, mariabackup SST, IST, multi master, 3 node cluster, split brain, quorum, pc.weight, certification failure, ER_LOCK_DEADLOCK at COMMIT, hot row, wsrep_cluster_status, wsrep_cluster_size, wsrep_local_state, Synced state, galera_new_cluster, bootstrap node, garbd arbitrator, Galera setup, multi master in mariadb, how do I set up a Galera cluster, my cluster keeps splitting, deadlock at commit, why does my transaction fail on commit, getting started with Galera
license: MIT
compatibility: "Designed for Claude Code. Requires MariaDB 10.6-LTS, 10.11-LTS, 11.x, 12.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# MariaDB Galera Cluster

Galera is a synchronous multi-master cluster built on certification-based replication. Every node can serve reads and writes, write-sets are replicated to every other node, and conflicts are detected at COMMIT time. This skill covers cluster sizing, bootstrap and join procedures, SST method choice, weighted quorum, monitoring, and the application-side retry strategy that Galera REQUIRES.

## Quick Reference

- ALWAYS run a minimum of 3 nodes. A 2-node cluster cannot form a stable quorum and either goes read-only on any partition or splits brains when each node assumes the other is dead.
- ALWAYS use `mariabackup` as the SST method on production clusters. `rsync` blocks the donor with a read lock and `mysqldump` is logical, slow, and also blocks the donor.
- ALWAYS keep every node on the SAME MariaDB minor version. Heterogeneous clusters cause replication mismatch and SST-stream incompatibility.
- ALWAYS retry transactions on `ER_LOCK_DEADLOCK` (error 1213). Galera reports certification failures as deadlocks AT COMMIT TIME, even when the local node never touched a conflicting row.
- ALWAYS set `binlog_format=ROW`, `default_storage_engine=InnoDB`, and `innodb_autoinc_lock_mode=2` on every node. Galera does not support `STATEMENT` format or non-InnoDB tables.
- ALWAYS use 3+ nodes OR add a `garbd` arbitrator (zero-data quorum-vote node) when running an even-numbered cluster across two data-centres.
- ALWAYS bootstrap the FIRST node with `galera_new_cluster` (or `mariadbd --wsrep-new-cluster`). Subsequent nodes start normally with `wsrep_cluster_address=gcomm://node1,node2,node3`.
- NEVER point application writes at multiple Galera nodes for the same hot row. Concurrent updates to one row from many nodes cause a cert-failure storm and roll back at COMMIT on every node but one.
- NEVER run `mysqldump` SST in production. Donor is locked for the entire dump duration ; on multi-GB schemas this is hours.
- NEVER scale reads by reading from every Galera node mid-SST. The donor lags during the transfer and serves stale data.
- NEVER use a 2-node cluster without `pc.weight` weighted quorum or a `garbd` arbitrator. There is no safe partition outcome.
- For read-scaling : add an ASYNC replica off ONE Galera node using standard primary-replica replication, not from every Galera node.

## Decision Trees

### Cluster size selection

```
Production cluster, single data-centre ?
   YES : 3 nodes minimum. 5 for higher fault tolerance (tolerates 2 simultaneous failures).
   NO  : continue

Two data-centres with WAN between them ?
   Symmetric (2+2) : add a 5th node OR a garbd arbitrator in a third location.
                     Without a third site, any WAN split takes both DCs read-only.
   Asymmetric (2+1) : set pc.weight=2 on the larger-DC nodes so the larger DC survives
                      a WAN split. The smaller DC goes read-only.

Single-node development / test ?
   YES : run standalone MariaDB, not Galera. wsrep_provider=none.
   NO  : 3 nodes minimum.
```

NEVER deploy 2 nodes. The only safe 2-node deployment uses `garbd` as a third vote, which makes it effectively 3-node.

### SST method selection

```
Production cluster, joiner needs full state transfer ?
   YES : wsrep_sst_method=mariabackup
         Donor stays writable during transfer. Modern default since MariaDB 10.1.26+ / 10.2.10+.

Joiner has recent state in gcache from a brief disconnect ?
   YES : IST runs automatically (no SST). Faster, no full data copy.
         Requires gcache.size large enough to hold writesets since joiner's last seqno.

Small dataset (< 10 GB), maintenance window acceptable ?
   ACCEPTABLE FALLBACK : wsrep_sst_method=rsync. Locks donor with read lock during transfer.

Logical SST for cross-major-version cluster upgrade ?
   ACCEPTABLE ONLY THEN : wsrep_sst_method=mysqldump. Slow, blocks donor. Rare use-case.

Production cluster, never want donor blocked ?
   ALWAYS : wsrep_sst_method=mariabackup.
```

`mariabackup` is the only SST method that does NOT block the donor with a read lock. On any cluster that takes production traffic, mariabackup is the only correct choice.

### Bootstrap vs join

```
Brand-new cluster, no node has ever been part of this cluster ?
   FIRST node  : galera_new_cluster   (creates cluster UUID, becomes Primary)
   OTHER nodes : systemctl start mariadb  (join via wsrep_cluster_address)

Cluster lost quorum, all nodes shut down, restarting ?
   Pick the node with the highest seqno (check grastate.dat).
   That node : galera_new_cluster   (re-bootstraps from latest state)
   OTHER nodes : systemctl start mariadb  (join, IST or SST as needed)

Single node failed and rejoining ?
   Failed node : systemctl start mariadb  (NEVER galera_new_cluster after the cluster exists)
                 IST runs automatically if gcache has the missing writesets.
```

NEVER run `galera_new_cluster` on more than one node simultaneously. That creates two disjoint clusters with the same name.

### Application retry strategy

```
Transaction returns ER_LOCK_DEADLOCK (1213) at COMMIT ?
   YES : this is a Galera certification failure, NOT an application bug.
         Retry the ENTIRE transaction in a new connection. Retry budget : 3-5 attempts
         with exponential back-off (10 ms, 50 ms, 250 ms).

Transaction returns ER_LOCK_WAIT_TIMEOUT (1205) mid-transaction ?
   YES : standard InnoDB row-lock timeout. Retry from BEGIN.

Transaction succeeds on local node but rolled back at COMMIT ?
   YES : Galera-specific. Local node optimistically commits, write-set is rejected
         on remote node during certification. Application receives 1213.
         Retry.

Same row updated thousands of times per second from every Galera node ?
   ROOT CAUSE : hot-row pattern, certification storm.
   FIX : route all writes to that row through ONE node (sticky routing),
         or shard the hot key across multiple rows.
```

In Galera, ALL write contention surfaces as deadlock-at-COMMIT, not row-lock-wait. Applications MUST have retry logic. No retry logic means random transaction failures under load.

## Patterns

### Minimal 3-node my.cnf (per node, identical except node_address and node_name)

```ini
# 10.6+ : 3-node Galera cluster, mariabackup SST
[mariadb]
# wsrep core
wsrep_on                = ON
wsrep_provider          = /usr/lib/galera/libgalera_smm.so
wsrep_cluster_name      = prod_cluster
wsrep_cluster_address   = gcomm://10.0.0.11,10.0.0.12,10.0.0.13

# Per-node identity
wsrep_node_address      = 10.0.0.11
wsrep_node_name         = node1

# SST
wsrep_sst_method        = mariabackup
wsrep_sst_auth          = sstuser:sstpass

# Galera-required InnoDB and binlog settings
binlog_format           = ROW
default_storage_engine  = InnoDB
innodb_autoinc_lock_mode = 2

# Optional : enlarge gcache to allow IST instead of SST after long disconnects
wsrep_provider_options  = "gcache.size=2G"
```

`innodb_autoinc_lock_mode=2` (interleaved) is REQUIRED for Galera. Mode 1 (consecutive) serialises AUTO_INCREMENT inserts across the cluster, which Galera cannot guarantee.

### Bootstrap first node + join others

```bash
# On node1 ONLY (creates the cluster)
galera_new_cluster

# Confirm node1 is Primary
mariadb -e "SHOW STATUS LIKE 'wsrep_cluster_status';"
# +----------------------+---------+
# | wsrep_cluster_status | Primary |
# +----------------------+---------+

# On node2 and node3 (join the existing cluster)
systemctl start mariadb

# On any node : confirm cluster size
mariadb -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
# +--------------------+-------+
# | wsrep_cluster_size | 3     |
# +--------------------+-------+
```

NEVER run `galera_new_cluster` on node2 or node3 after node1 has bootstrapped. That fragments the cluster.

### SST user setup (mariabackup)

```sql
-- Run on the donor node BEFORE joiners need SST
CREATE USER 'sstuser'@'localhost' IDENTIFIED BY 'sstpass';
GRANT RELOAD, PROCESS, LOCK TABLES, BINLOG MONITOR ON *.* TO 'sstuser'@'localhost';
-- 10.5+ : BINLOG MONITOR replaces the legacy REPLICATION CLIENT privilege
```

`wsrep_sst_auth=sstuser:sstpass` in my.cnf points to this user. mariabackup uses it during SST.

### Weighted quorum for asymmetric deployment

```ini
# DC-A has 2 nodes, DC-B has 1 node. Without weighting, WAN split takes everyone read-only.
# Setting pc.weight=2 on DC-A nodes lets DC-A survive a WAN split (quorum 2+2=4 vs 1).
[mariadb]
# DC-A nodes :
wsrep_provider_options = "pc.weight=2; gcache.size=2G"

# DC-B node :
wsrep_provider_options = "pc.weight=1; gcache.size=2G"
```

The cluster sums weights of reachable nodes vs total cluster weight. Reachable > total/2 means Primary, otherwise NON_PRIMARY (read-only).

### Garbd arbitrator (zero-data third vote)

```bash
# garbd runs on a third host with no data, providing a quorum vote.
# Useful for 2-data-node deployments where a third full node is too expensive.
garbd --address gcomm://10.0.0.11,10.0.0.12 \
      --group prod_cluster \
      --log /var/log/garbd.log \
      --daemon
```

garbd is part of the `galera-arbitrator` package on most distributions. It participates in quorum but stores no data, so it can run on a small VM.

### Health monitoring queries

```sql
-- The four queries to run every minute via your monitoring system
SHOW STATUS LIKE 'wsrep_cluster_size';        -- 3 (expected) ; alert if < 3
SHOW STATUS LIKE 'wsrep_cluster_status';      -- Primary (expected) ; alert if not Primary
SHOW STATUS LIKE 'wsrep_local_state_comment'; -- Synced (expected) ; alert if not Synced
SHOW STATUS LIKE 'wsrep_local_cert_failures'; -- Watch trend ; spike = hot-row contention

-- Verify gcache headroom so IST stays possible after short outages
SHOW STATUS LIKE 'wsrep_local_recv_queue';    -- Should stay near 0 ; rising = overload
```

A healthy node reports `wsrep_cluster_status=Primary`, `wsrep_local_state_comment=Synced`, `wsrep_ready=ON`, `wsrep_connected=ON`.

### Application retry pattern (Python pseudo-code)

```python
import time, mariadb

def commit_with_retry(conn, work, max_attempts=5):
    for attempt in range(max_attempts):
        try:
            conn.begin()
            work(conn.cursor())
            conn.commit()
            return
        except mariadb.OperationalError as e:
            # 1213 = ER_LOCK_DEADLOCK (Galera cert-failure surfaces as this at COMMIT)
            # 1205 = ER_LOCK_WAIT_TIMEOUT (standard InnoDB row-lock timeout)
            if e.errno in (1213, 1205) and attempt < max_attempts - 1:
                conn.rollback()
                time.sleep(0.01 * (5 ** attempt))   # 10 ms, 50 ms, 250 ms, 1.25 s
                continue
            raise
    raise RuntimeError("transaction failed after retries")
```

NEVER catch the deadlock and proceed without retrying. NEVER retry inside the SAME transaction (the transaction is already aborted by the server).

### Read-scaling via async replica

```sql
-- On any ONE Galera node (the "replication source") :
SHOW MASTER STATUS;
-- +--------------------+----------+--------------+------------------+
-- | File               | Position | Binlog_Do_DB | Binlog_Ignore_DB |
-- +--------------------+----------+--------------+------------------+
-- | mariadb-bin.000042 |    74591 |              |                  |
-- +--------------------+----------+--------------+------------------+

-- On the read-only async replica (NOT part of the Galera cluster) :
CHANGE MASTER TO
   MASTER_HOST='10.0.0.11',
   MASTER_USER='replica',
   MASTER_PASSWORD='<secret>',
   MASTER_LOG_FILE='mariadb-bin.000042',
   MASTER_LOG_POS=74591,
   MASTER_SSL=1,
   MASTER_SSL_VERIFY_SERVER_CERT=1;
START SLAVE;
```

NEVER replicate FROM more than one Galera node concurrently to the same async replica. The replica's GTID/position state becomes ambiguous if the source switches mid-stream.

## Cross-References

- `mariadb-impl-replication-setup` for the async-replica side of read-scaling, GTID semantics, and TLS for replication.
- `mariadb-impl-backup-restore` for `mariabackup` usage in non-cluster contexts (the SST is a thin wrapper over the same tool).
- `mariadb-syntax-transactions` for `ER_LOCK_DEADLOCK` semantics, isolation levels, and standalone InnoDB retry patterns.
- `mariadb-errors-replication` for SST-failure diagnosis, gcache exhaustion, and node-stuck-in-Joiner-state recovery.
- `mariadb-core-architecture` for the wsrep API layering, when Galera plug-in loads, and how it intercepts the InnoDB commit path.
- `mariadb-impl-query-optimization` for hot-row design avoidance and sharded-key patterns.

## Reference Files

- `references/methods.md` : full wsrep_* variable reference, SST-method comparison matrix, monitoring query catalogue, garbd setup syntax.
- `references/examples.md` : 10+ working configurations covering 3-node cluster.cnf, bootstrap first node, join additional nodes, SST via mariabackup, IST recovery, monitoring queries, retry-loop application pattern, garbd arbitrator setup.
- `references/anti-patterns.md` : 8 anti-patterns with cause, symptom, and corrected approach.

## Sources

Verified against : `mariadb.com/kb/en/galera-cluster/`, `mariadb.com/kb/en/galera-cluster-system-variables/`, `mariadb.com/kb/en/galera-cluster-status-variables/`, `mariadb.com/docs/galera-cluster/high-availability/state-snapshot-transfers-ssts-in-galera-cluster/introduction-to-state-snapshot-transfers-ssts.md`, `mariadb.com/docs/galera-cluster/galera-management/installation-and-deployment/getting-started-with-mariadb-galera-cluster.md`, `mariadb.com/kb/en/mariabackup/`. Last verified : 2026-05-19. Galera upstream `galeracluster.com` is bot-blocked (L-003) ; all upstream variable references cross-verified against MariaDB KB and the MariaDB/server GitHub source.
