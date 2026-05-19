---
name: mariadb-errors-galera-conflicts
description: >
  Use when a Galera cluster reports certification failures, ER_LOCK_DEADLOCK fires at COMMIT in a multi-master setup, wsrep_local_cert_failures keeps climbing, a hot-row is contended across nodes, or the cluster appears to be in split-brain.
  Prevents the common mistake of treating Galera ER_LOCK_DEADLOCK as a bug, retrying without backoff, ignoring hot-row design, running a 2-node Galera without an arbitrator, or "fixing" symptoms by disabling wsrep_on.
  Covers certification-based replication semantics, ER_LOCK_DEADLOCK at COMMIT (Galera-specific path), wsrep_local_cert_failures and wsrep_local_bf_aborts status variables, hot-row detection plus sharding redesign, split-brain prevention with pc.weight and garbd, application retry strategy with exponential backoff, gcache exhaust leading to forced SST.
  Keywords: Galera, certification failure, wsrep_local_cert_failures, wsrep_local_bf_aborts, ER_LOCK_DEADLOCK at commit, ER_QUERY_INTERRUPTED, hot row, multi-master conflict, split brain, pc.weight, pc.ignore_sb, garbd, arbitrator, retry transaction, my galera keeps deadlocking, Galera-specific deadlock, write-set certification, wsrep_provider_options, gcache exhausted, forced SST, wsrep_cluster_status NON_PRIMARY, why does my commit fail on galera, how do I fix galera certification errors, what is a galera split brain, how do I stop galera deadlocks
license: MIT
compatibility: "Designed for Claude Code. Requires MariaDB 10.6-LTS, 10.11-LTS, 11.x, 12.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# MariaDB Galera Cluster Conflicts and Certification Failures

How to recognise, diagnose, and recover from Galera-specific failure modes : certification deadlocks at COMMIT, hot-row contention across nodes, split-brain and Non-Primary partitions, gcache exhaustion forcing a full SST. Galera is a synchronous multi-master cluster built on certification-based replication ; its failure surface is different from standalone InnoDB and must be reasoned about as a cluster property, not a per-node property.

## Quick Reference

- Galera `ER_LOCK_DEADLOCK` (error 1213) returned at COMMIT is NORMAL behavior, NOT a bug. The certification protocol detects write-write conflicts between nodes only at commit-time. ALWAYS retry the entire transaction.
- `wsrep_local_cert_failures` is a monotonic counter of certification failures on THIS node. Growing fast = hot-row contention across nodes. Redesign with sharded keys.
- `wsrep_local_bf_aborts` counts local transactions aborted by an incoming applier (brute-force abort). High value alongside cert-failures confirms multi-master write conflict pressure.
- ALWAYS retry with exponential backoff (50 ms base, 2x growth, max 5 attempts, optional jitter) to avoid retry-storms that compound the conflict.
- A 2-node Galera cluster is UNSAFE : it cannot form a stable quorum on a partition. ALWAYS run 3 nodes minimum, OR 2 full nodes + 1 `garbd` arbitrator.
- `pc.weight` (provider option, default 1, dynamic) biases quorum on asymmetric DC topology. Set the larger DC higher so it survives a WAN split.
- `pc.ignore_sb=ON` is DANGEROUS in multi-master : it allows writes on a disconnected partition. Use ONLY on a read-only standby or single-master topology, NEVER as a "quick fix" for split-brain alarms.
- `wsrep_cluster_status='NON_PRIMARY'` means this node lost quorum and is read-only. Find the minority partition and recover it ; do NOT force `pc.bootstrap` blindly.
- NEVER disable `wsrep_on` to "make the error go away". It de-syncs the node from the cluster and corrupts cluster consistency.
- Galera replicates ONLY write-sets (modified rows). Long SELECT-only transactions on hot data do NOT cause certification failures on the executing node.

## Galera Failure-Mode Identification

| Symptom | Where it surfaces | What it means |
|---------|-------------------|---------------|
| `ER_LOCK_DEADLOCK` (1213, SQLSTATE 40001) returned at the `COMMIT` line | Application client on any node | Certification conflict : another node committed a write-set that conflicts with this one. Retry full transaction. |
| `wsrep_local_cert_failures` rising fast | `SHOW STATUS LIKE 'wsrep_local_cert_failures'` | This node is losing the certification race repeatedly. Indicates hot-row contention or cross-node write-storm. |
| `wsrep_local_bf_aborts` rising fast | `SHOW STATUS LIKE 'wsrep_local_bf_aborts'` | Local in-flight transactions are being killed by remote applier threads (brute-force abort). Same root cause as cert-failures from the other direction. |
| Application sees `ER_QUERY_INTERRUPTED` | Application client | Mid-flight transaction killed by an incoming higher-priority write-set. Treat identically to 1213 : retry. |
| `wsrep_cluster_status='NON_PRIMARY'` on a subset of nodes | `SHOW STATUS LIKE 'wsrep_cluster_status'` | Quorum lost on those nodes. They are read-only. Identify and recover the minority partition. |
| All nodes reachable but cluster-size shrunk | `SHOW STATUS LIKE 'wsrep_cluster_size'` returns less than expected node count | One or more nodes have left the primary component. Check their `wsrep_local_state_comment` for cause. |
| Joining node never finishes IST, falls back to full SST | Joiner log, `wsrep_local_state_comment` cycles `Joining` to `Donor/Desynced` | The donor's `gcache` no longer holds the write-sets the joiner needs. Forced full SST. |

The most common confusion : a transaction that touched no obviously-contended row on the LOCAL node still gets `ER_LOCK_DEADLOCK` at `COMMIT`. That is the certification protocol working as designed. The conflict was detected against a write-set that ANOTHER node committed during this transaction's lifetime.

## Decision Tree

```
Galera cluster reporting errors
|
+-- ER_LOCK_DEADLOCK / ER_QUERY_INTERRUPTED at COMMIT ?
|     |
|     +-- Is wsrep_on=ON on this node ? (confirm with SHOW VARIABLES)
|     |     NO  : this is standalone InnoDB. See mariadb-errors-deadlocks instead.
|     |     YES : continue.
|     |
|     +-- Has wsrep_local_cert_failures jumped > 10x baseline ?
|     |     YES : hot-row contention. Identify the hot key (see methods.md).
|     |           Redesign : shard the row or use a sequence.
|     |     NO  : transient cross-node write race. Retry transaction with backoff.
|     |
|     +-- Has the same logical transaction failed > 5 retries ?
|           YES : design problem. Stop retrying, fix the root cause (hot-row, lock-order).
|           NO  : exponential backoff retry, max 5 attempts.
|
+-- wsrep_cluster_status='NON_PRIMARY' ?
|     |
|     +-- How many nodes total in the deployment ?
|     |     2 : you have no quorum protection. Add garbd or a 3rd node BEFORE recovery.
|     |     3+ : a minority partition lost quorum. Find the majority, validate its data.
|     |
|     +-- Is this an asymmetric DC layout (2 DC-A, 1 DC-B) ?
|     |     YES : verify pc.weight is biased to the larger DC. See examples.md.
|     |     NO  : continue to recovery.
|     |
|     +-- Recover by joining the minority node BACK to the primary component.
|           Do NOT force pc.bootstrap unless ALL nodes are NON_PRIMARY and you confirmed
|           which node has the latest committed data (via wsrep_last_committed).
|
+-- Joining node loops Joining/Donor/Desynced ?
      |
      +-- Check joiner log for IST failure : gcache exhausted on donor.
      +-- Increase gcache.size on a healthy node, or restart the joiner with SST forced.
      +-- See anti-patterns.md : gcache too small under write-storm.
```

## Certification-Based Replication Mental Model

A Galera transaction is local-only until `COMMIT`. At COMMIT, the modified rows are bundled as a write-set and broadcast to all nodes. Each node certifies the write-set against every in-flight and recently-committed write-set on that node. If a conflict is found (same primary key modified concurrently), certification fails and the transaction is aborted on the originating node with `ER_LOCK_DEADLOCK`.

Three consequences :

1. The error always lands at `COMMIT`, NEVER mid-transaction. Standalone InnoDB deadlocks land on the offending statement ; Galera certification deadlocks land on commit. This changes where the retry-handler must sit in the application code.
2. The originating node sees the failure even though no local lock was held by another local transaction. The conflicting transaction can be on a different node entirely.
3. Long-running READ-ONLY transactions do NOT cause certification failures. Only WRITE-sets are certified. A reporting transaction can run for hours without triggering this path (it has other risks ; see `mariadb-impl-galera-cluster`).

## Key Status Variables

| Variable | Type | Meaning |
|----------|------|---------|
| `wsrep_local_cert_failures` | monotonic counter | Local transactions that failed certification on this node and were rolled back |
| `wsrep_local_bf_aborts` | monotonic counter | Local transactions aborted by an incoming applier (brute-force abort) |
| `wsrep_cluster_status` | gauge | `PRIMARY` (quorum present), `NON_PRIMARY` (quorum lost), `DISCONNECTED` |
| `wsrep_local_state` | gauge | Internal FSM state number |
| `wsrep_local_state_comment` | gauge | Human-readable state (e.g. `Synced`, `Donor/Desynced`, `Joining`, `Joined`) |
| `wsrep_cluster_size` | gauge | Number of nodes currently in the primary component |
| `wsrep_last_committed` | gauge | Sequence number of the last committed write-set on this node |
| `wsrep_provider_options` | system variable | Active provider-option string (includes `pc.weight`, `pc.ignore_sb`, `gcache.size`, etc.) |

ALWAYS alert on RATE OF CHANGE of `wsrep_local_cert_failures` and `wsrep_local_bf_aborts`, NEVER on the absolute value. A non-zero counter is normal on any healthy multi-master cluster.

```sql
-- 10.6+ : Galera health snapshot
SHOW STATUS LIKE 'wsrep_local_cert_failures';
SHOW STATUS LIKE 'wsrep_local_bf_aborts';
SHOW STATUS LIKE 'wsrep_cluster_status';
SHOW STATUS LIKE 'wsrep_cluster_size';
SHOW STATUS LIKE 'wsrep_local_state_comment';
```

See `references/methods.md` for the complete wsrep status-variable reference and a Galera error-code matrix.

## Application Retry Strategy

Retry the ENTIRE transaction, not the failing statement. Use exponential backoff with random jitter to avoid synchronized retries (retry-storm). Log every retry. Treat the retry budget as a circuit-breaker : exhausting it means a design problem, not a transient failure.

```python
# Python 3.10+, mariadb 1.1+, MariaDB 10.6+ Galera cluster
import mariadb, time, random, logging

MAX_RETRIES = 5
BASE_BACKOFF_SECONDS = 0.05  # 50 ms

GALERA_CERT_ERRORS = {1213, 1614}  # ER_LOCK_DEADLOCK, ER_QUERY_INTERRUPTED equivalents

def run_with_galera_retry(conn, work_fn):
    for attempt in range(1, MAX_RETRIES + 1):
        try:
            cur = conn.cursor()
            cur.execute("START TRANSACTION")
            work_fn(cur)
            conn.commit()
            return
        except mariadb.OperationalError as e:
            errno = e.args[0] if e.args else None
            if errno in GALERA_CERT_ERRORS:
                conn.rollback()
                if attempt == MAX_RETRIES:
                    logging.error("Galera retries exhausted ; design review needed")
                    raise
                backoff = BASE_BACKOFF_SECONDS * (2 ** (attempt - 1)) + random.uniform(0, 0.025)
                logging.warning("Galera conflict attempt %d, backoff %.3fs", attempt, backoff)
                time.sleep(backoff)
                continue
            raise
```

Retry budget is 3 to 5 attempts. If a transaction needs more than that under normal load, the workload has a hot-row, missing index, or bad lock-order. Retries mask the problem ; they do not fix it.

See `references/examples.md` for retry implementations in PHP, Node.js, and Java.

## Hot-Row Redesign

A single primary key contended by writers on multiple nodes will deadlock under any sustained write load. Galera cannot make this go away ; the certification protocol detects the conflict and one node always loses. Common hot-row patterns :

- Global counter table : `UPDATE counter SET v = v + 1` on the only row.
- Single-row inventory : `UPDATE stock SET qty = qty - 1 WHERE sku = ?` on a popular SKU.
- Single-row rate-limit bucket : `UPDATE rl SET count = count + 1 WHERE key = ?`.
- Sequence-emulated-as-row : a single-row `current_value` table.

The fix is design-level, not tuning-level. Shard the hot key across N rows, aggregate at read time.

```sql
-- 10.6+ : sharded counter, N=16 shards keyed by hash
CREATE TABLE counter_sharded (
  shard TINYINT UNSIGNED NOT NULL,
  v BIGINT NOT NULL DEFAULT 0,
  PRIMARY KEY (shard)
);

INSERT INTO counter_sharded (shard, v)
SELECT seq.seq, 0 FROM seq_0_to_15 seq;

-- Increment : pick a random shard
UPDATE counter_sharded SET v = v + 1
  WHERE shard = FLOOR(RAND() * 16);

-- Read : sum across shards
SELECT SUM(v) AS total FROM counter_sharded;
```

For sequences, use the native `SEQUENCE` object (MariaDB 10.3+) instead of an `UPDATE counter`. Sequences are gap-tolerant by design and avoid the certification race entirely.

```sql
-- 10.6+ : SEQUENCE avoids hot-row entirely
CREATE SEQUENCE order_id_seq START WITH 1 INCREMENT BY 1;
INSERT INTO orders (id, ...) VALUES (NEXTVAL(order_id_seq), ...);
```

See `references/examples.md` for additional hot-row redesigns (stock, rate-limit, queue table).

## Split-Brain Prevention

A split-brain occurs when a network partition divides the cluster into groups that each believe the other is dead. With a 2-node cluster this is unavoidable on any partition. Galera prevents both sides from accepting writes via the Primary Component (PC) algorithm : only the partition with the majority of nodes (by count, weighted by `pc.weight`) accepts writes ; the minority becomes `NON_PRIMARY` and goes read-only.

Three deployment patterns prevent split-brain :

1. **3 full nodes** (preferred) : any single partition leaves a 2-node majority.
2. **2 full nodes + 1 garbd arbitrator** : `garbd` is a quorum-only daemon that participates in voting but stores no data. Place garbd on a third host so that a 2-host cluster has 3 quorum participants.
3. **Asymmetric DC with `pc.weight`** : 2 nodes in DC-A with `pc.weight=2`, 1 node in DC-B with `pc.weight=1`. On a WAN split, DC-A keeps writes ; DC-B goes read-only.

```sql
-- 10.6+ : set pc.weight at runtime on a node
SET GLOBAL wsrep_provider_options = 'pc.weight=2';
```

```bash
# Start garbd on a third host (no MariaDB needed, just galera-arbitrator package)
garbd --group prod_cluster \
      --address gcomm://node1.example.com,node2.example.com \
      --log-file /var/log/garbd.log
```

See `references/methods.md` for the full `pc.*` and `gcs.*` provider-option reference, and `references/examples.md` for asymmetric-DC weighted-quorum deployments.

## Recovering a Non-Primary Partition

If `wsrep_cluster_status='NON_PRIMARY'` on every node (rare ; usually the result of a multi-DC outage), one node must bootstrap a new primary component. ALWAYS pick the node with the highest `wsrep_last_committed` value (the latest committed data).

```sql
-- 10.6+ : query the seqno on each NON_PRIMARY node BEFORE bootstrapping
SHOW STATUS LIKE 'wsrep_last_committed';
```

```sql
-- 10.6+ : on the node with the highest seqno only
SET GLOBAL wsrep_provider_options = 'pc.bootstrap=true';
```

The other nodes then rejoin via IST (if their `gcache` is still in range) or SST (full state transfer otherwise). Forcing `pc.bootstrap` on the wrong node loses committed data and is unrecoverable without a backup.

See `references/anti-patterns.md` for the "force pc.bootstrap blindly" anti-pattern.

## gcache Exhaustion and Forced SST

When a node falls behind and tries to rejoin via IST (incremental state transfer), the donor node must still hold the missing write-sets in its `gcache` ring buffer. Default `gcache.size=128M`. On a write-heavy cluster, 128M holds only seconds of write-sets ; a node that was down for minutes will need a full SST instead.

Symptoms of forced SST : joiner log cycles between `Joining` and `Donor/Desynced` ; joiner `wsrep_local_state_comment` shows SST in progress ; donor goes read-only during SST under the legacy `rsync` method. The fix : size `gcache.size` for the expected outage window. For a 10 MB/s sustained write rate and a 1-hour outage budget : 36 GB gcache.

```sql
-- 10.6+ : check active provider options including gcache.size
SHOW VARIABLES LIKE 'wsrep_provider_options';
```

`gcache.size` is NOT dynamic ; setting it via `SET GLOBAL wsrep_provider_options` does not resize an active ring buffer. Restart the node with the new value in `my.cnf`.

## SST Method Selection

| Method | Status | Donor blocked | Use when |
|--------|--------|---------------|----------|
| `mariabackup` | Recommended default | NO (non-blocking, InnoDB-aware) | Always, for production |
| `rsync` | Deprecated for SST | YES (donor read-locks during transfer) | Only for legacy setups ; avoid |
| `mysqldump` | Deprecated for SST | YES (logical dump, very slow) | Never on production |

ALWAYS set `wsrep_sst_method=mariabackup` in `my.cnf`. The `rsync` and `mysqldump` methods are kept for backward compatibility, not for new deployments.

See `references/methods.md` for the SST method comparison and `mariadb-impl-galera-cluster` for full setup.

## What This Skill Does NOT Cover

- Initial Galera cluster setup and `my.cnf` configuration : see `mariadb-impl-galera-cluster`.
- Standalone InnoDB deadlocks (not Galera) : see `mariadb-errors-deadlocks`.
- Asynchronous replication lag (non-Galera) : see `mariadb-errors-replication-lag`.
- Backup and restore including `mariabackup` standalone use : see `mariadb-impl-backup-restore`.
- General performance tuning (buffer pool, IO, query cache) : see `mariadb-impl-performance-tuning`.

## Reference Links

- `references/methods.md` : Galera error-code matrix, complete wsrep_* status variable reference, certification flow diagram, hot-row redesign patterns, `pc.*` / `gcs.*` / `evs.*` provider-option reference.
- `references/examples.md` : 10+ working examples (reproduce certification failure, monitor wsrep_local_cert_failures, hot-row sharding redesign, application retry loop, pc.weight asymmetric DC, garbd setup, gcache sizing, SST recovery, NON_PRIMARY bootstrap).
- `references/anti-patterns.md` : 8+ real anti-patterns (treating commit-deadlock as bug, retry without backoff, hot-row design, 2-node without garbd, pc.ignore_sb in multi-master, disabling wsrep_on, gcache too small, force pc.bootstrap blindly).

## Source Verification

All facts in this skill were verified via WebFetch against MariaDB official documentation :

- `mariadb.com/kb/en/galera-cluster-status-variables/` : `wsrep_local_cert_failures` (monotonic counter, total local transactions that failed certification), `wsrep_local_bf_aborts` (monotonic counter, local transactions aborted by replication applier threads), `wsrep_cluster_status` (`PRIMARY` / `NON_PRIMARY` / `DISCONNECTED`), `wsrep_local_state`, `wsrep_local_state_comment`.
- `mariadb.com/kb/en/galera-cluster-system-variables/` : `wsrep_on` (default OFF, must be ON to join cluster), `wsrep_provider` (path to libgalera_smm.so), `wsrep_cluster_address` (`gcomm://node1,node2,node3`), `wsrep_retry_autocommit` (default 1, range 0-10000), `wsrep_max_ws_size` (default 2 GB), `wsrep_sync_wait` (session, bitmask 0-15), `wsrep_certify_nonPK` (default ON).
- `mariadb.com/docs/galera-cluster/reference/wsrep-variable-details/wsrep_provider_options` : `pc.weight` (default 1, dynamic, integer), `pc.ignore_sb` (default false, dynamic, dangerous in multi-master), `pc.recovery` (default true, not dynamic), `gcache.size` (default 128M, not dynamic), `evs.suspect_timeout` (default PT5S).
- `mariadb.com/docs/server/reference/error-codes/mariadb-error-codes-1200-to-1299` : 1213 `ER_LOCK_DEADLOCK` "Deadlock found when trying to get lock; try restarting transaction", 1205 `ER_LOCK_WAIT_TIMEOUT` "Lock wait timeout exceeded; try restarting transaction".
- Vooronderzoek Cluster-3 §2 : certification-based replication semantics, `wsrep_local_cert_failures` as the canonical indicator, `pc.weight` for asymmetric DC quorum, 2-node Galera plus `garbd` as the minimum-viable HA shape.
- LESSONS L-003 : Galera upstream `galeracluster.com` is bot-blocked ; MariaDB KB pages are the canonical source.
