# Galera Cluster Conflicts : Anti-Patterns

Real anti-patterns that produce, hide, or worsen Galera-specific conflicts. Each entry shows the broken approach, why it fails, and the correct alternative. All examples target MariaDB 10.6+ Galera clusters.

## Anti-Pattern 1 : Treating ER_LOCK_DEADLOCK at COMMIT as an application bug

### Broken

```python
# WRONG : logging the Galera certification deadlock as a fatal error
try:
    conn.commit()
except mariadb.OperationalError as e:
    if e.args[0] == 1213:
        logging.critical("DATABASE BUG: deadlock on commit, alerting on-call")
        send_pagerduty_alert("DB deadlock")
        raise SystemExit(1)
```

### Why it fails

On a Galera cluster, `ER_LOCK_DEADLOCK` (1213) returned at `COMMIT` is the certification protocol working exactly as designed. The cluster detected a write-write conflict against a write-set committed on another node. It is NOT a bug, NOT data corruption, NOT a server fault. Paging on-call for it produces constant false alarms and trains the team to ignore real incidents. Crashing the process discards work that a simple retry would have completed.

### Correct

```python
# CORRECT : a certification deadlock is an expected, recoverable outcome
try:
    conn.commit()
except mariadb.OperationalError as e:
    if e.args[0] in (1213, 1614):
        conn.rollback()
        # retry the whole transaction with backoff (see examples.md)
        retry_transaction()
    else:
        raise
```

Treat 1213-at-commit on Galera identically to a standalone InnoDB deadlock : roll back and retry the entire transaction. Alert only on the RATE of `wsrep_local_cert_failures`, never on a single occurrence.

## Anti-Pattern 2 : Retrying without backoff (the retry-storm)

### Broken

```python
# WRONG : tight retry loop with no delay
while True:
    try:
        conn.execute("START TRANSACTION")
        do_work(conn)
        conn.commit()
        break
    except mariadb.OperationalError as e:
        if e.args[0] == 1213:
            conn.rollback()
            continue   # immediately retry, no sleep, no limit
        raise
```

### Why it fails

When two transactions on different nodes conflict, both clients land here at the same instant. With no backoff, both immediately retry, conflict again at the same instant, and retry again. This is a synchronized retry-storm : the two transactions lock-step forever, each guaranteeing the other fails. CPU and replication bandwidth are consumed with zero progress. With no retry limit, the loop never terminates.

### Correct

```python
# CORRECT : exponential backoff + jitter + bounded attempts
import time, random
MAX_RETRIES = 5
for attempt in range(1, MAX_RETRIES + 1):
    try:
        conn.execute("START TRANSACTION")
        do_work(conn)
        conn.commit()
        break
    except mariadb.OperationalError as e:
        if e.args[0] in (1213, 1614):
            conn.rollback()
            if attempt == MAX_RETRIES:
                raise
            backoff = 0.05 * (2 ** (attempt - 1)) + random.uniform(0, 0.025)
            time.sleep(backoff)
            continue
        raise
```

Exponential backoff de-synchronizes the two transactions ; random jitter breaks the lock-step ; a bounded attempt count converts a hopeless hot-row into a clear error instead of an infinite loop.

## Anti-Pattern 3 : Hot-row design (single contended counter)

### Broken

```sql
-- WRONG : a single row updated by every writer on every node
CREATE TABLE global_counter (id INT PRIMARY KEY, total BIGINT NOT NULL);
INSERT INTO global_counter VALUES (1, 0);

-- every request, on every node, hits the same primary key :
UPDATE global_counter SET total = total + 1 WHERE id = 1;
```

### Why it fails

Galera certifies write-sets by primary key. Every node writing row `id=1` produces a write-set that conflicts with every other node's write-set for `id=1`. Under sustained concurrent load, certification failures become continuous : one node always loses the race. No amount of retry tuning fixes this ; the contention is structural.

### Correct

```sql
-- CORRECT : shard the hot key across N rows
CREATE TABLE counter_sharded (
  shard TINYINT UNSIGNED NOT NULL,
  total BIGINT NOT NULL DEFAULT 0,
  PRIMARY KEY (shard)
) ENGINE=InnoDB;
INSERT INTO counter_sharded (shard, total)
SELECT seq.seq, 0 FROM seq_0_to_31 seq;

-- writer scatters across shards :
UPDATE counter_sharded SET total = total + 1 WHERE shard = FLOOR(RAND() * 32);

-- reader aggregates :
SELECT SUM(total) FROM counter_sharded;
```

For an ID generator, use a native `SEQUENCE` (10.3+) which avoids the certification race entirely. The hot-row problem is a DESIGN problem ; it must be fixed at the schema level, not the retry level.

## Anti-Pattern 4 : 2-node Galera cluster with no arbitrator

### Broken

```ini
# WRONG : a 2-node Galera cluster with nothing to break a tie
# node1 my.cnf
wsrep_cluster_address = gcomm://node1,node2
# node2 my.cnf
wsrep_cluster_address = gcomm://node1,node2
```

### Why it fails

Two nodes cannot form a stable quorum. On a network partition, each node sees exactly one node (itself) and cannot tell whether the peer crashed or the link broke. Galera resolves this conservatively : BOTH nodes go `NON_PRIMARY` and read-only, so a single network blip takes the entire cluster down for writes. Without conservative resolution, both would accept writes and diverge (true split-brain). Either way a 2-node cluster has no safe failure mode.

### Correct

```bash
# CORRECT : add a garbd arbitrator on a third host
# on host3 (NOT a data node) :
apt install galera-arbitrator-4
# /etc/default/garbd
GALERA_GROUP="prod_cluster"
GALERA_NODES="node1:4567,node2:4567"
systemctl enable --now garbd
```

`garbd` is a quorum-only daemon : it votes but stores no data. With 2 data nodes + garbd, a partition that isolates one data node still leaves a 2-vote majority that keeps writes available. The minimum safe Galera topology is 3 nodes, OR 2 nodes + 1 garbd.

## Anti-Pattern 5 : Setting pc.ignore_sb=ON in a multi-master cluster

### Broken

```sql
-- WRONG : disabling split-brain protection to silence NON_PRIMARY alarms
SET GLOBAL wsrep_provider_options = 'pc.ignore_sb=ON';
```

### Why it fails

`pc.ignore_sb=ON` tells the node to keep accepting writes even when it is disconnected from the cluster and has lost quorum. In a multi-master cluster, this is exactly the condition that causes a true split-brain : both partitions accept conflicting writes, and when the network heals the two divergent histories cannot be merged. The data is permanently inconsistent. The MariaDB documentation explicitly cautions that this option "could lead to data inconsistency in a multi-master setup."

### Correct

```sql
-- CORRECT : keep split-brain protection ON ; fix the real quorum problem
SET GLOBAL wsrep_provider_options = 'pc.ignore_sb=OFF';   -- the safe default
```

If `NON_PRIMARY` alarms are frequent, the fix is more quorum participants (a 3rd node or garbd) and `pc.weight` tuning for asymmetric topologies, NOT disabling the protection. `pc.ignore_sb=ON` is acceptable ONLY on a single-master deployment or a read-only standby where divergence is impossible.

## Anti-Pattern 6 : Disabling wsrep_on to "make the error go away"

### Broken

```sql
-- WRONG : turning off cluster replication on a node that throws conflicts
SET GLOBAL wsrep_on = OFF;
-- "the certification deadlocks stopped, problem solved"
```

### Why it fails

`wsrep_on=OFF` removes the node from the cluster's replication stream. Local writes on that node are no longer broadcast to the cluster, and incoming write-sets are no longer applied. The node silently diverges from every other node : its data is now wrong and getting worse with every write. The certification deadlocks "stopped" only because the node stopped participating in the consistency protocol. Re-enabling `wsrep_on` later forces a full SST to discard the divergent state, and any writes made while disconnected are lost.

### Correct

Leave `wsrep_on=ON` always. Certification deadlocks are resolved at the application layer with retry plus, for sustained failures, hot-row redesign. If a node genuinely must be removed from the cluster (maintenance), stop the MariaDB service cleanly so it leaves the primary component gracefully, then rejoin it via the normal IST/SST path. NEVER use `wsrep_on=OFF` as a runtime workaround on a live cluster node.

## Anti-Pattern 7 : gcache too small for the realistic outage window

### Broken

```ini
# WRONG : leaving the default gcache.size on a high-write cluster
[mariadb]
wsrep_provider_options = "gcache.size=128M"   # the default
```

### Why it fails

`gcache.size` controls how far back a donor can serve incremental state transfer (IST). On a cluster producing 10 MB/s of write-sets, a 128M gcache holds roughly 13 seconds of history. Any node down longer than that, even a brief restart for a config change, falls back to a full SST : a complete data copy that can take hours on a large dataset and (under the legacy `rsync` method) read-locks the donor. Routine maintenance becomes a multi-hour cluster-degradation event.

### Correct

```ini
# CORRECT : size gcache for the realistic worst-case outage
# write rate ~10 MB/s, outage budget 1 hour, 1.5x safety :
# 10 MB/s * 3600 s * 1.5 = 54 GB
[mariadb]
wsrep_provider_options = "gcache.size=54G"
```

Measure the actual write-set rate (`wsrep_replicated_bytes` delta over time), multiply by the longest outage you want IST to cover, add a safety factor. `gcache.size` is not dynamic ; set it in `my.cnf` and restart. A correctly-sized gcache keeps node rejoins fast and incremental.

## Anti-Pattern 8 : Forcing pc.bootstrap on the wrong node

### Broken

```sql
-- WRONG : bootstrapping a primary component from whatever node responds first
-- (all nodes are NON_PRIMARY after a multi-DC outage)
-- operator runs this on node1 without checking seqno :
SET GLOBAL wsrep_provider_options = 'pc.bootstrap=true';
```

### Why it fails

When every node is `NON_PRIMARY`, one node must bootstrap a new primary component, and the other nodes then rejoin and adopt ITS data as the cluster truth. If the bootstrapped node is not the one with the latest committed data, every write that landed only on a higher-seqno node is permanently discarded the moment those nodes rejoin via SST. There is no recovery without a backup. Picking "the first node that responds" is a coin-flip on data loss.

### Correct

```sql
-- CORRECT : check wsrep_last_committed on EVERY node first
-- run on each node :
SHOW STATUS LIKE 'wsrep_last_committed';
-- node1 : 184529
-- node2 : 184601   <- highest seqno, has the latest committed data
-- node3 : 184529

-- bootstrap from node2 ONLY :
SET GLOBAL wsrep_provider_options = 'pc.bootstrap=true';   -- on node2
```

ALWAYS bootstrap from the node with the highest `wsrep_last_committed`. The other nodes then rejoin and converge to that node's state without losing committed writes. When in doubt, take a backup of every node before bootstrapping anything.

## Anti-Pattern 9 : Long-running write transaction on hot data

### Broken

```sql
-- WRONG : a long transaction holding modified hot rows until COMMIT
START TRANSACTION;
UPDATE inventory SET qty = qty - 1 WHERE sku = 'POPULAR-SKU';   -- hot row, modified early
-- ... 30 seconds of other work : external API calls, report generation ...
INSERT INTO order_line (sku, qty) VALUES ('POPULAR-SKU', 1);
COMMIT;   -- write-set certified only here, 30 seconds later
```

### Why it fails

Galera certifies the write-set at `COMMIT`. The longer a transaction stays open after modifying a hot row, the larger the window in which another node can commit a conflicting write-set for that same row. A 30-second transaction that touched `POPULAR-SKU` early is almost guaranteed to fail certification at commit because dozens of other nodes' transactions for that SKU committed during those 30 seconds. The transaction also blocks nothing on other nodes (Galera is optimistic) but is itself near-certain to lose.

### Correct

```sql
-- CORRECT : keep write transactions short ; do slow work OUTSIDE the transaction
-- 1. do the slow work first (API calls, report generation) with no transaction open
-- 2. open the transaction, touch hot rows, commit immediately :
START TRANSACTION;
UPDATE inventory SET qty = qty - 1 WHERE sku = 'POPULAR-SKU';
INSERT INTO order_line (sku, qty) VALUES ('POPULAR-SKU', 1);
COMMIT;   -- write-set certified within milliseconds of touching the hot row
```

Minimise the time between modifying a hot row and committing. Move every non-database operation (network calls, computation, file IO) outside the transaction boundary. A short transaction has a tiny conflict window and certifies successfully far more often.

## Anti-Pattern 10 : Skipping wsrep_local_cert_failures monitoring

### Broken

The cluster runs with no observability on `wsrep_local_cert_failures` or `wsrep_local_bf_aborts`. The first signal of trouble is users reporting failed checkouts.

### Why it fails

Certification failures are a leading indicator of a hot-row problem that will eventually saturate retry budgets and surface as user-visible errors. Without monitoring the rate of `wsrep_local_cert_failures`, the operator has no way to catch a developing hot-row before it becomes an incident, and no way to confirm a redesign actually reduced contention.

### Correct

Scrape `wsrep_local_cert_failures` and `wsrep_local_bf_aborts` into the metrics system and alert on the RATE of change, not the absolute value.

```sql
-- CORRECT : expose the counters for a metrics agent to scrape
SELECT VARIABLE_NAME, VARIABLE_VALUE
FROM information_schema.GLOBAL_STATUS
WHERE VARIABLE_NAME IN (
  'WSREP_LOCAL_CERT_FAILURES', 'WSREP_LOCAL_BF_ABORTS',
  'WSREP_CLUSTER_STATUS', 'WSREP_CLUSTER_SIZE',
  'WSREP_FLOW_CONTROL_PAUSED', 'WSREP_LOCAL_RECV_QUEUE_AVG'
);
```

A baseline rate is normal under multi-master write load. Alert when the rate jumps by an order of magnitude, or when `wsrep_cluster_status` is anything other than `PRIMARY`, or when `wsrep_flow_control_paused` rises (the cluster is throttling itself under back-pressure).
