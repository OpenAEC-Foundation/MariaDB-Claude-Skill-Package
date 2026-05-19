# Galera Cluster : Working Examples

Twelve end-to-end working examples covering bootstrap, join, SST, IST, monitoring, weighted quorum, garbd, recovery, async-replica scaling, and application retry logic.

## Example 1 : 3-node cluster.cnf for node1 (10.6+)

```ini
# /etc/mysql/mariadb.conf.d/60-galera.cnf on node1 (10.0.0.11)
[mariadb]
# wsrep core
wsrep_on                  = ON
wsrep_provider            = /usr/lib/galera/libgalera_smm.so
wsrep_cluster_name        = prod_cluster
wsrep_cluster_address     = gcomm://10.0.0.11,10.0.0.12,10.0.0.13

# Per-node identity (change on node2 and node3)
wsrep_node_address        = 10.0.0.11
wsrep_node_name           = node1

# SST
wsrep_sst_method          = mariabackup
wsrep_sst_auth            = sstuser:sstpass

# Required Galera companions
binlog_format             = ROW
default_storage_engine    = InnoDB
innodb_autoinc_lock_mode  = 2

# Provider tuning : enlarge gcache for IST resilience
wsrep_provider_options    = "gcache.size=2G; gcache.page_size=128M"

# Bind addresses
bind-address              = 0.0.0.0
```

For node2 : change `wsrep_node_address=10.0.0.12` and `wsrep_node_name=node2`. For node3 : `wsrep_node_address=10.0.0.13` and `wsrep_node_name=node3`. Everything else identical.

## Example 2 : Bootstrap first node

```bash
# Step 1 : Verify wsrep is configured on node1 (parses, does not start cluster)
mariadbd --wsrep-provider --print-defaults | head -20

# Step 2 : Bootstrap the cluster (node1 only, ONE TIME EVER)
galera_new_cluster

# Step 3 : Confirm node1 is Primary and Synced
mariadb -uroot -e "
   SHOW STATUS LIKE 'wsrep_cluster_size';
   SHOW STATUS LIKE 'wsrep_cluster_status';
   SHOW STATUS LIKE 'wsrep_local_state_comment';
"

# Expected output :
# wsrep_cluster_size         1
# wsrep_cluster_status       Primary
# wsrep_local_state_comment  Synced
```

`galera_new_cluster` is a thin wrapper that sets `_WSREP_NEW_CLUSTER=--wsrep-new-cluster` and starts the service. Equivalent : `mariadbd --wsrep-new-cluster`.

## Example 3 : Join nodes 2 and 3

```bash
# On node2 (after node1 is up and Primary)
systemctl start mariadb
# node2 will SST from node1 if grastate.dat is empty (first join), then catch up

# On node3
systemctl start mariadb

# On any node : confirm cluster is now 3 nodes
mariadb -uroot -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
# wsrep_cluster_size    3

# Confirm all nodes are Synced (run on each node)
mariadb -uroot -e "SHOW STATUS LIKE 'wsrep_local_state_comment';"
# wsrep_local_state_comment    Synced
```

NEVER run `galera_new_cluster` on node2 or node3. That creates two disjoint clusters with the same name and silently corrupts data.

## Example 4 : SST user setup (mariabackup)

```sql
-- Run on node1 BEFORE node2 and node3 try to SST (replicates to all nodes once cluster is up)
CREATE USER 'sstuser'@'localhost' IDENTIFIED BY 'sstpass';

-- 10.5+ : BINLOG MONITOR replaces REPLICATION CLIENT
GRANT RELOAD, PROCESS, LOCK TABLES, BINLOG MONITOR ON *.* TO 'sstuser'@'localhost';

-- Pre-10.5 :
-- GRANT RELOAD, PROCESS, LOCK TABLES, REPLICATION CLIENT ON *.* TO 'sstuser'@'localhost';

-- For encrypted tablespaces add :
-- GRANT CREATE TABLESPACE ON *.* TO 'sstuser'@'localhost';

FLUSH PRIVILEGES;
```

The `wsrep_sst_auth=sstuser:sstpass` line in my.cnf points to this user. mariabackup uses these credentials during SST.

## Example 5 : Inspect why an SST is happening

```bash
# Watch the SST progress on the donor (node1)
tail -F /var/log/mysql/error.log | grep -i 'sst\|mariabackup'

# Typical donor log lines :
# [Note] WSREP: Initiating SST/IST transfer on DONOR side (...)
# [Note] WSREP: DONOR thread signaled with -22
# [Note] WSREP: 0.0 (node2): State transfer to 0.0 (node2) complete.

# Watch the joiner (node2)
tail -F /var/log/mysql/error.log | grep -i 'sst\|mariabackup\|wsrep'

# Typical joiner log lines :
# [Note] WSREP: Requesting state transfer: success, donor: 0
# [Note] WSREP: SST received: 6dc81a1b-...
# [Note] WSREP: Member 1 (node2) synced with group.
```

If the joiner stays in `Joining` state for hours, check : disk space on joiner, `wsrep_sst_auth` correctness, firewall on ports 4567 (gcomm), 4568 (IST), 4444 (SST).

## Example 6 : Trigger and verify IST instead of SST

```bash
# Stop one node briefly to force a small gap
systemctl stop mariadb     # on node3

# Run some writes on node1 / node2 to advance the seqno
mariadb -uroot -e "INSERT INTO test.events SELECT seq FROM seq_1_to_10000;"

# Restart node3 ; with adequate gcache, IST runs
systemctl start mariadb

# Check the joiner log : look for "Receiving IST" instead of "SST received"
grep -i 'ist\|sst' /var/log/mysql/error.log | tail -10

# Typical IST log on joiner :
# [Note] WSREP: Receiving IST: 10000 writesets, seqnos 15234-25234
# [Note] WSREP: IST received: 6dc81a1b-...:25234
```

IST runs ONLY if `gcache.size` on the donor is large enough to hold every write-set since the joiner's last seqno. If gcache rolled over, the cluster falls back to full SST.

## Example 7 : Weighted quorum for asymmetric DC deployment

```ini
# Scenario : DC-A has 2 nodes, DC-B has 1 node. Goal : DC-A survives WAN split.

# On node1 and node2 (in DC-A) :
[mariadb]
wsrep_provider_options = "pc.weight=2; gcache.size=2G"

# On node3 (in DC-B) :
[mariadb]
wsrep_provider_options = "pc.weight=1; gcache.size=2G"
```

```sql
-- Verify weighted-quorum is in effect
SHOW STATUS LIKE 'wsrep_provider_options';
-- Look for : pc.weight=2 (on DC-A nodes) and pc.weight=1 (on DC-B node)
```

WAN split : DC-A has weight 2+2=4, DC-B has weight 1. DC-A survives (4 > total/2 = 5/2 = 2.5), DC-B goes NON_PRIMARY (read-only). When the WAN heals, DC-B re-syncs via IST or SST.

## Example 8 : Garbd arbitrator (third vote for 2-node cluster)

```bash
# Install garbd on a third host (small VM, no MariaDB data)
apt install galera-arbitrator-4   # Debian/Ubuntu
# OR
dnf install galera-arbitrator-4   # RHEL/CentOS

# Configure /etc/default/garb (Debian) or /etc/sysconfig/garb (RHEL)
cat > /etc/default/garb <<'EOF'
GALERA_NODES="10.0.0.11:4567 10.0.0.12:4567"
GALERA_GROUP="prod_cluster"
LOG_FILE="/var/log/garbd.log"
EOF

# Start and enable
systemctl enable --now garb

# Verify on the data nodes (node1 / node2)
mariadb -uroot -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
# wsrep_cluster_size    3        <- 2 data nodes + 1 garbd vote
```

`garbd` stores no data but participates in quorum. Place it in a THIRD failure domain (separate rack, AZ, or DC) so a single failure cannot take all three votes.

## Example 9 : Health monitoring queries (cron-able)

```sql
-- Run every minute via your monitoring system (Prometheus mysqld_exporter, Telegraf, etc.)
SELECT
   (SELECT VARIABLE_VALUE FROM INFORMATION_SCHEMA.GLOBAL_STATUS
      WHERE VARIABLE_NAME='wsrep_cluster_size')           AS cluster_size,
   (SELECT VARIABLE_VALUE FROM INFORMATION_SCHEMA.GLOBAL_STATUS
      WHERE VARIABLE_NAME='wsrep_cluster_status')         AS cluster_status,
   (SELECT VARIABLE_VALUE FROM INFORMATION_SCHEMA.GLOBAL_STATUS
      WHERE VARIABLE_NAME='wsrep_local_state_comment')    AS local_state,
   (SELECT VARIABLE_VALUE FROM INFORMATION_SCHEMA.GLOBAL_STATUS
      WHERE VARIABLE_NAME='wsrep_local_cert_failures')    AS cert_failures,
   (SELECT VARIABLE_VALUE FROM INFORMATION_SCHEMA.GLOBAL_STATUS
      WHERE VARIABLE_NAME='wsrep_local_recv_queue')       AS recv_queue,
   (SELECT VARIABLE_VALUE FROM INFORMATION_SCHEMA.GLOBAL_STATUS
      WHERE VARIABLE_NAME='wsrep_flow_control_paused')    AS flow_paused;
```

Alerts to configure :
- `cluster_size != expected_node_count` (warn after 30 s, critical after 5 min)
- `cluster_status != 'Primary'` (critical immediately)
- `local_state != 'Synced'` (warn after 30 s, critical after 5 min)
- `cert_failures` rate > 10/sec (warn ; investigate hot-row pattern)
- `recv_queue > 100` sustained (warn ; appliers cannot keep up)
- `flow_paused > 0.1` (warn ; cluster overloaded)

## Example 10 : Application retry pattern (Python)

```python
import time
import mariadb

# Galera-aware commit-with-retry. Catches ER_LOCK_DEADLOCK (1213) at COMMIT
# (Galera certification failure surfaces here) and ER_LOCK_WAIT_TIMEOUT (1205).
def commit_with_retry(pool, work, max_attempts=5):
    for attempt in range(max_attempts):
        conn = pool.get_connection()
        try:
            conn.begin()
            work(conn.cursor())
            conn.commit()
            return
        except mariadb.OperationalError as e:
            conn.rollback()
            if e.errno in (1213, 1205) and attempt < max_attempts - 1:
                # Exponential back-off : 10 ms, 50 ms, 250 ms, 1.25 s, 6.25 s
                time.sleep(0.01 * (5 ** attempt))
                continue
            raise
        finally:
            conn.close()
    raise RuntimeError(f"transaction failed after {max_attempts} attempts")


# Usage
def increment_counter(cursor):
    cursor.execute("UPDATE counters SET value = value + 1 WHERE id = 1")

commit_with_retry(pool, increment_counter)
```

The retry MUST be in a NEW connection. Reusing the same connection after a deadlock-rollback is undefined behaviour.

## Example 11 : Read-scaling via async replica off Galera

```sql
-- ON ONE Galera node (node1) : note current binlog position
SHOW MASTER STATUS;
-- +--------------------+----------+
-- | File               | Position |
-- +--------------------+----------+
-- | mariadb-bin.000042 |    74591 |
-- +--------------------+----------+

-- Create replication user on the Galera cluster (replicates to all nodes)
CREATE USER 'replica'@'10.0.0.50' IDENTIFIED BY '<secret>';
GRANT REPLICATION SLAVE ON *.* TO 'replica'@'10.0.0.50';
FLUSH PRIVILEGES;
```

```bash
# Take a mariabackup from node1, restore on the async replica (10.0.0.50)
mariabackup --backup --target-dir=/backup/from-node1 \
            --user=root --password=<root-pw>
# ... ship the backup to 10.0.0.50, prepare, copy-back, start mariadb ...
```

```sql
-- ON the async replica (10.0.0.50, NOT in the Galera cluster)
CHANGE MASTER TO
   MASTER_HOST='10.0.0.11',
   MASTER_USER='replica',
   MASTER_PASSWORD='<secret>',
   MASTER_LOG_FILE='mariadb-bin.000042',
   MASTER_LOG_POS=74591,
   MASTER_SSL=1,
   MASTER_SSL_VERIFY_SERVER_CERT=1;

START SLAVE;

-- Confirm replication is running
SHOW SLAVE STATUS\G
-- Look for : Slave_IO_Running: Yes ; Slave_SQL_Running: Yes ; Seconds_Behind_Master: 0
```

The async replica is OUTSIDE the Galera cluster. It serves read-only traffic without consuming Galera certification bandwidth. NEVER point CHANGE MASTER at multiple Galera nodes ; the source-position state becomes ambiguous.

## Example 12 : Total cluster recovery from full shutdown

```bash
# Scenario : power outage took all nodes down. Restart procedure :

# Step 1 : on every node, inspect grastate.dat
for n in node1 node2 node3 ; do
   echo "=== $n ==="
   ssh $n cat /var/lib/mysql/grastate.dat
done

# Typical output :
# === node1 ===
# version: 2.1
# uuid:    6dc81a1b-7afe-11ee-95b3-7b8c4a1d2c5e
# seqno:   25234
# safe_to_bootstrap: 0

# === node2 ===
# version: 2.1
# uuid:    6dc81a1b-7afe-11ee-95b3-7b8c4a1d2c5e
# seqno:   25234
# safe_to_bootstrap: 1            <- THIS node is safe to bootstrap

# === node3 ===
# version: 2.1
# uuid:    6dc81a1b-7afe-11ee-95b3-7b8c4a1d2c5e
# seqno:   25230
# safe_to_bootstrap: 0

# Step 2 : pick the node with safe_to_bootstrap: 1 AND highest seqno (node2)
# Bootstrap from node2 :
ssh node2 'galera_new_cluster'

# Step 3 : start the other nodes normally
ssh node1 'systemctl start mariadb'
ssh node3 'systemctl start mariadb'

# Step 4 : verify cluster reformed
mariadb -uroot -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
# wsrep_cluster_size    3
```

If NO node has `safe_to_bootstrap: 1`, the highest-seqno node can still be bootstrapped manually after editing grastate.dat. This loses any writes made after that seqno on other nodes. Investigate WHY the cluster lost quorum before bootstrapping to avoid data loss.

## Example 13 : Recover a single failed node

```bash
# node3 crashed mid-transaction ; rejoin procedure :

# Step 1 : check grastate.dat on node3
cat /var/lib/mysql/grastate.dat
# If seqno: -1, run wsrep-recover to extract real seqno from InnoDB log
mariadbd --wsrep-recover
# Output : ... Recovered position: 6dc81a1b-...:25240
# Update grastate.dat with seqno: 25240

# Step 2 : start node3 normally (NEVER galera_new_cluster on a single failed node)
systemctl start mariadb

# Step 3 : node3 will IST from a donor (node1 or node2) if gcache covers the gap,
#          otherwise SST. Watch the error log :
tail -F /var/log/mysql/error.log

# Healthy rejoin sequence :
# [Note] WSREP: Requesting state transfer
# [Note] WSREP: IST received: ...    OR    SST received: ...
# [Note] WSREP: Member 2 (node3) synced with group.

# Step 4 : confirm
mariadb -uroot -e "SHOW STATUS LIKE 'wsrep_local_state_comment';"
# wsrep_local_state_comment    Synced
```

NEVER run `galera_new_cluster` to recover a single node from an otherwise-healthy cluster. That command creates a new cluster and SPLITS your existing cluster in two.
