# Galera Cluster : Anti-Patterns

Eight anti-patterns observed in real Galera deployments, with the symptom, root cause, and corrected approach for each.

---

## 1 : 2-node Galera cluster

### Anti-pattern

```ini
# my.cnf on both nodes
wsrep_cluster_address = gcomm://10.0.0.11,10.0.0.12
# Only 2 data nodes, no garbd arbitrator
```

### Symptom

- Random read-only outages when one node briefly disconnects (both nodes go NON_PRIMARY because neither has majority).
- Split-brain after a network partition : each node assumes the other is dead, both accept writes, data diverges, cluster cannot heal without manual conflict resolution.

### Root cause

Galera quorum requires a strict majority of votes. With 2 nodes, no single-node partition can hold majority (1 vote is exactly half, not more). The Primary Component algorithm has no safe outcome.

### Correct approach

Run 3 data nodes minimum. If a third data node is too expensive, add a `garbd` arbitrator on a third host. `garbd` consumes minimal resources and stores no data, but provides the third vote that makes quorum decisions deterministic.

```bash
# On a third host (small VM, separate failure domain)
apt install galera-arbitrator-4
cat > /etc/default/garb <<'EOF'
GALERA_NODES="10.0.0.11:4567 10.0.0.12:4567"
GALERA_GROUP="prod_cluster"
EOF
systemctl enable --now garb

# wsrep_cluster_size now reports 3 (2 data nodes + garbd vote)
```

---

## 2 : mysqldump SST in production

### Anti-pattern

```ini
wsrep_sst_method = mysqldump
wsrep_sst_auth   = sstuser:sstpass
```

### Symptom

- Donor node holds a global read lock for the ENTIRE duration of the dump.
- On a 100 GB schema, the donor is unwritable for HOURS.
- Application requests queue up against the donor or fail with `ER_LOCK_WAIT_TIMEOUT`.

### Root cause

`mysqldump` SST is logical : it runs `FLUSH TABLES WITH READ LOCK`, dumps every table to SQL, sends to the joiner, joiner loads. The donor cannot accept writes during the dump. The other Galera nodes continue, but write-sets pile up in the donor's receive queue.

### Correct approach

Use `mariabackup` SST. It is a hot-backup tool that does NOT lock the donor.

```ini
wsrep_sst_method = mariabackup
wsrep_sst_auth   = sstuser:sstpass
```

```sql
-- Create the SST user with the right privileges (10.5+)
CREATE USER 'sstuser'@'localhost' IDENTIFIED BY 'sstpass';
GRANT RELOAD, PROCESS, LOCK TABLES, BINLOG MONITOR ON *.* TO 'sstuser'@'localhost';
FLUSH PRIVILEGES;
```

`mariabackup` SST is non-blocking and the production standard since MariaDB 10.1.26+ / 10.2.10+.

---

## 3 : Heterogeneous MariaDB versions in one cluster

### Anti-pattern

```
node1 : MariaDB 10.6.16
node2 : MariaDB 10.11.5
node3 : MariaDB 11.4.2
```

### Symptom

- SST fails with version-mismatch errors during data-dictionary load.
- Replication stalls on DDL statements that one version cannot parse (for example `INSTANT ADD COLUMN` syntax differences).
- Galera protocol negotiation falls back to lowest-common-denominator, losing features the newer nodes support.

### Root cause

Galera replicates write-sets at the storage-engine level, but the data dictionary, table format, and SQL parser must be compatible across nodes. Different minor versions can have different on-disk formats for InnoDB system tables.

### Correct approach

Keep every node on the SAME minor version. Upgrade one node at a time via rolling upgrade :

```bash
# Rolling upgrade procedure (one node at a time)
# Step 1 : stop node3
systemctl stop mariadb        # on node3

# Step 2 : upgrade packages on node3
apt install mariadb-server=10.11.6+maria~ubu2204

# Step 3 : restart node3, let it IST or SST back into the cluster
systemctl start mariadb

# Step 4 : run mariadb-upgrade on node3
mariadb-upgrade

# Step 5 : confirm node3 is Synced before moving to node2
mariadb -e "SHOW STATUS LIKE 'wsrep_local_state_comment';"
# Synced

# Repeat for node2, then node1
```

For major version jumps (10.x to 11.x), use `wsrep_sst_method=mysqldump` for the upgrade SST (logical dump survives format changes), then switch back to `mariabackup` after the cluster is on the new major version.

---

## 4 : Retrying inside the same transaction

### Anti-pattern

```python
# WRONG : retry inside the already-aborted transaction
cursor.execute("BEGIN")
try:
    cursor.execute("UPDATE accounts SET balance = balance - 100 WHERE id = 1")
    cursor.execute("COMMIT")
except mariadb.OperationalError as e:
    if e.errno == 1213:
        cursor.execute("UPDATE accounts SET balance = balance - 100 WHERE id = 1")  # WRONG
        cursor.execute("COMMIT")
```

### Symptom

- "Cannot execute statement in transaction" errors.
- Application silently runs queries OUTSIDE any transaction after the rollback.
- Data corruption : partial updates committed without isolation.

### Root cause

When `ER_LOCK_DEADLOCK` (1213) fires, the server has already rolled the transaction back. The connection is in autocommit mode again. Any subsequent statement runs without transactional protection.

### Correct approach

Open a NEW transaction (new BEGIN) when retrying. Wrap the retry loop around the ENTIRE transaction, including BEGIN.

```python
for attempt in range(5):
    try:
        cursor.execute("BEGIN")
        cursor.execute("UPDATE accounts SET balance = balance - 100 WHERE id = 1")
        cursor.execute("COMMIT")
        break
    except mariadb.OperationalError as e:
        if e.errno in (1213, 1205) and attempt < 4:
            time.sleep(0.01 * (5 ** attempt))
            continue
        raise
```

In a connection pool, return the connection to the pool after each failed attempt and acquire a fresh one. The connection state after a deadlock-rollback should not be reused.

---

## 5 : Treating ER_LOCK_DEADLOCK as an application bug

### Anti-pattern

```python
try:
    do_transaction()
except mariadb.OperationalError as e:
    if e.errno == 1213:
        logger.error("deadlock detected, transaction failed permanently")
        send_alert("deadlock in production")
        raise UserVisibleError("our database had a problem, please try again")
```

### Symptom

- Constant alerts about "deadlocks" in production under any non-trivial load.
- End users see "please try again" errors for legitimate operations.
- Engineering team spends time hunting non-existent locking bugs.

### Root cause

In Galera, `ER_LOCK_DEADLOCK` at COMMIT time is the NORMAL signal for a certification failure. It happens whenever two nodes commit write-sets that conflict (touching the same row, or row-ranges, or unique-key values) within the same certification interval. This is not a bug ; it is how Galera enforces consistency.

### Correct approach

Retry the transaction automatically with exponential back-off. Only surface a user-visible error after the retry budget is exhausted.

```python
def commit_with_retry(work, max_attempts=5):
    for attempt in range(max_attempts):
        try:
            return work()
        except mariadb.OperationalError as e:
            if e.errno in (1213, 1205) and attempt < max_attempts - 1:
                time.sleep(0.01 * (5 ** attempt))
                continue
            raise
```

Monitor `wsrep_local_cert_failures` over time. A steady rate is normal in a multi-master cluster. A SUDDEN spike indicates a hot-row pattern that needs sharding or sticky routing.

---

## 6 : Hot-row update from many nodes simultaneously

### Anti-pattern

```sql
-- Application runs this from every Galera node, hundreds of times per second
UPDATE inventory_summary SET total = total + 1 WHERE id = 1;
```

### Symptom

- `wsrep_local_cert_failures` spikes into the thousands per second.
- Throughput collapses : application sees > 50% of UPDATE statements rolled back at COMMIT with 1213.
- Adding more application servers makes the problem WORSE, not better.

### Root cause

Every UPDATE on the same row generates a write-set targeting the same row. Certification rejects all but one of them per certification interval, across all nodes. The more concurrent writers, the higher the failure rate. This is the classic Galera hot-row pattern.

### Correct approach

Option A : route all writes for this row through ONE node (sticky routing in the load balancer or application).

```python
# Application picks one node for all "counter" writes
counter_node_conn = pool_for("node1")   # always node1
counter_node_conn.execute("UPDATE inventory_summary SET total = total + 1 WHERE id = 1")
```

Option B : shard the hot key across multiple rows, sum at read time.

```sql
-- Shard the counter into 64 rows ; writes are random per-node, reads aggregate
CREATE TABLE inventory_summary_shard (
   id        BIGINT NOT NULL,
   shard_id  TINYINT NOT NULL,
   total     BIGINT NOT NULL DEFAULT 0,
   PRIMARY KEY (id, shard_id)
);

-- Application picks a random shard for each write
UPDATE inventory_summary_shard SET total = total + 1
   WHERE id = 1 AND shard_id = FLOOR(RAND() * 64);

-- Reads aggregate across shards
SELECT id, SUM(total) AS total FROM inventory_summary_shard
   WHERE id = 1 GROUP BY id;
```

Option C : use an external counter store (Redis INCR, dedicated counter service) and reconcile to MariaDB periodically.

---

## 7 : Disabling foreign-key checks during SST

### Anti-pattern

```ini
# WRONG : trying to "speed up" SST by relaxing constraints
[mariadb]
wsrep_sst_method = mariabackup
init_connect     = "SET FOREIGN_KEY_CHECKS=0;"
```

### Symptom

- SST completes "successfully" but the joiner has orphan rows (FK references to non-existent parents).
- Subsequent writes on the joiner fail with `ER_NO_REFERENCED_ROW` (1452).
- Cluster appears Synced but queries return inconsistent results across nodes.

### Root cause

`mariabackup` SST is atomic and consistent BY DESIGN. It uses InnoDB's MVCC snapshot to capture a transactionally-consistent point-in-time. Disabling FK checks does not speed up the SST and instead allows the joiner to load data in an inconsistent order, corrupting referential integrity.

### Correct approach

NEVER disable FK checks on a Galera node. Let `mariabackup` handle consistency. If SST is too slow, the fix is to enlarge `gcache.size` so IST becomes possible, not to weaken SST.

```ini
[mariadb]
wsrep_sst_method        = mariabackup
wsrep_provider_options  = "gcache.size=4G; gcache.page_size=128M"
# FOREIGN_KEY_CHECKS stays at its default (ON)
```

---

## 8 : Reading from a Galera node during its SST

### Anti-pattern

```python
# Application connects round-robin to all Galera nodes
# Joiner node accepts the connection during SST and returns stale or empty data
conn = pool.get_random_node()  # might be the joiner during SST
results = conn.execute("SELECT * FROM products WHERE id = 1")
```

### Symptom

- Intermittent "row not found" errors during cluster recovery operations.
- Customers see partial cart contents or missing orders for the duration of an SST.
- Reads return data from BEFORE the SST started.

### Root cause

A node receiving SST is in `Joiner` state. Its `wsrep_local_state_comment` is `Joining` or `SST receiver`. The node may accept connections but has incomplete or stale data until SST finishes and the node reaches `Synced`. Some workloads stay connectable during SST but serve inconsistent reads.

### Correct approach

Application connection pool MUST check `wsrep_local_state_comment` before sending queries to a node, and exclude any node not in `Synced` state.

```python
def is_node_ready(conn):
    cursor = conn.cursor()
    cursor.execute("SHOW STATUS LIKE 'wsrep_local_state_comment'")
    state = cursor.fetchone()[1]
    return state == 'Synced'

def get_ready_connection(pool):
    for node in pool.nodes:
        conn = node.get_connection()
        if is_node_ready(conn):
            return conn
        conn.close()
    raise RuntimeError("no Galera node in Synced state")
```

Better : use a load-balancer with built-in Galera awareness (MaxScale with the `galeramon` monitor, HAProxy with `mysql-check`, or ProxySQL with `mysql_galera_hostgroups`). These check `wsrep_local_state_comment=Synced` automatically and route only to ready nodes.

```ini
# MaxScale snippet : galeramon excludes non-Synced nodes from the routing pool
[Galera-Monitor]
type            = monitor
module          = galeramon
servers         = node1,node2,node3
user            = maxuser
password        = <secret>
monitor_interval = 2s
disable_master_failback = true
```
