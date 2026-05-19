# Galera Cluster Conflicts : Examples

Working examples for reproducing, monitoring, and resolving Galera-specific conflicts. Every example is annotated with the MariaDB version range it applies to. All examples assume a running Galera cluster with `wsrep_on=ON`.

## Example 1 : Reproduce a certification failure

Two clients connected to two DIFFERENT Galera nodes update the SAME row concurrently. One commit succeeds, the other fails certification.

```sql
-- 10.6+ : setup, run once on any node
CREATE TABLE account (
  id INT PRIMARY KEY,
  balance INT NOT NULL
) ENGINE=InnoDB;
INSERT INTO account VALUES (1, 1000);
```

```sql
-- 10.6+ : Client A on node1
START TRANSACTION;
UPDATE account SET balance = balance - 100 WHERE id = 1;
-- (do NOT commit yet)
```

```sql
-- 10.6+ : Client B on node2, runs while A is open
START TRANSACTION;
UPDATE account SET balance = balance - 200 WHERE id = 1;
COMMIT;   -- this one commits first, certifies cluster-wide
```

```sql
-- 10.6+ : back to Client A on node1
COMMIT;
-- ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
-- This is the certification failure : A's write-set conflicts with B's already-committed write-set.
```

The lesson : the error appears at `COMMIT`, not at the `UPDATE`. Client A's `UPDATE` succeeded locally with no visible problem ; the conflict only surfaced when the write-set was certified against B's committed write-set.

## Example 2 : Monitor wsrep_local_cert_failures over time

```sql
-- 10.6+ : take a baseline reading
SELECT VARIABLE_VALUE AS cert_failures_t0
FROM information_schema.GLOBAL_STATUS
WHERE VARIABLE_NAME = 'WSREP_LOCAL_CERT_FAILURES';
```

```bash
# Sample the counter every 10 seconds and print the delta (rate)
# MariaDB 10.6+, run on any Galera node
prev=0
while true; do
  cur=$(mariadb -N -e "SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS \
        WHERE VARIABLE_NAME='WSREP_LOCAL_CERT_FAILURES';")
  echo "$(date +%T)  cert_failures=$cur  delta=$((cur - prev))"
  prev=$cur
  sleep 10
done
```

ALWAYS reason about the DELTA, not the absolute number. A steady non-zero delta is normal under multi-master write load. A delta spiking by orders of magnitude indicates a new hot-row or write-storm.

## Example 3 : Identify the hot row

Find which primary keys are most contended by sampling `INNODB_TRX` and the slow log, then correlate with `wsrep_local_cert_failures`.

```sql
-- 10.6+ : list currently-running transactions and what they touch
SELECT trx_id, trx_state, trx_started, trx_query
FROM information_schema.INNODB_TRX
ORDER BY trx_started ASC;
```

```sql
-- 10.6+ : enable per-deadlock logging to catch the contended key
SET GLOBAL innodb_print_all_deadlocks = ON;
-- then tail the error log : grep for the table and WHERE clause repeating
```

When the same `UPDATE ... WHERE pk = <constant>` appears repeatedly in the deadlock log across nodes, that `pk` is the hot row. Proceed to the sharding redesign (Example 4).

## Example 4 : Hot-row sharding redesign

Replace a single contended counter row with N shards. Writes scatter ; reads aggregate.

```sql
-- 10.6+ : BEFORE - the hot-row anti-pattern
-- CREATE TABLE visit_counter (id INT PRIMARY KEY, total BIGINT);
-- INSERT INTO visit_counter VALUES (1, 0);
-- UPDATE visit_counter SET total = total + 1 WHERE id = 1;   -- hot row

-- 10.6+ : AFTER - sharded counter
CREATE TABLE visit_counter_sharded (
  shard TINYINT UNSIGNED NOT NULL,
  total BIGINT NOT NULL DEFAULT 0,
  PRIMARY KEY (shard)
) ENGINE=InnoDB;

INSERT INTO visit_counter_sharded (shard, total)
SELECT seq.seq, 0 FROM seq_0_to_31 seq;   -- 32 shards
```

```sql
-- 10.6+ : writer picks a random shard each time
UPDATE visit_counter_sharded
SET total = total + 1
WHERE shard = FLOOR(RAND() * 32);
```

```sql
-- 10.6+ : reader aggregates
SELECT SUM(total) AS visits FROM visit_counter_sharded;
```

With 32 shards and uniform random selection, the probability that two concurrent writers collide on the same shard drops to 1/32 per pair, cutting certification failures roughly proportionally.

## Example 5 : Replace a counter with a SEQUENCE

When the hot row is an ID generator, a native `SEQUENCE` removes the certification race entirely. Sequences are gap-tolerant by design.

```sql
-- 10.6+ : create a SEQUENCE instead of an UPDATE-counter
CREATE SEQUENCE invoice_no_seq
  START WITH 1000
  INCREMENT BY 1
  CACHE 50
  NOCYCLE;
```

```sql
-- 10.6+ : each writer takes the next value without contention
INSERT INTO invoice (invoice_no, customer_id, amount)
VALUES (NEXTVAL(invoice_no_seq), ?, ?);
```

The `CACHE 50` clause pre-allocates 50 values per node, removing inter-node coordination for 50 inserts at a time. Sequences may produce gaps after a restart ; this is expected and acceptable for an ID generator.

## Example 6 : Application retry loop in PHP

```php
<?php
// PHP 8.1+, PDO_MySQL, MariaDB 10.6+ Galera cluster
function runWithGaleraRetry(PDO $pdo, callable $work): void {
    $maxRetries = 5;
    $baseBackoffMs = 50;
    $certErrors = [1213, 1614];   // ER_LOCK_DEADLOCK, ER_QUERY_INTERRUPTED

    for ($attempt = 1; $attempt <= $maxRetries; $attempt++) {
        try {
            $pdo->beginTransaction();
            $work($pdo);
            $pdo->commit();
            return;
        } catch (PDOException $e) {
            $errno = (int) ($e->errorInfo[1] ?? 0);
            if (in_array($errno, $certErrors, true)) {
                $pdo->rollBack();
                if ($attempt === $maxRetries) {
                    error_log("Galera retries exhausted, design review needed");
                    throw $e;
                }
                $backoffMs = $baseBackoffMs * (2 ** ($attempt - 1)) + random_int(0, 25);
                error_log("Galera conflict attempt {$attempt}, backoff {$backoffMs}ms");
                usleep($backoffMs * 1000);
                continue;
            }
            throw $e;
        }
    }
}
```

## Example 7 : Application retry loop in Node.js

```javascript
// Node.js 18+, mariadb 3.x driver, MariaDB 10.6+ Galera cluster
const CERT_ERRORS = new Set([1213, 1614]);
const MAX_RETRIES = 5;
const BASE_BACKOFF_MS = 50;

async function runWithGaleraRetry(pool, workFn) {
  for (let attempt = 1; attempt <= MAX_RETRIES; attempt++) {
    const conn = await pool.getConnection();
    try {
      await conn.beginTransaction();
      await workFn(conn);
      await conn.commit();
      return;
    } catch (err) {
      await conn.rollback().catch(() => {});
      if (CERT_ERRORS.has(err.errno) && attempt < MAX_RETRIES) {
        const backoff = BASE_BACKOFF_MS * 2 ** (attempt - 1) + Math.random() * 25;
        await new Promise((r) => setTimeout(r, backoff));
        continue;
      }
      throw err;
    } finally {
      conn.release();
    }
  }
  throw new Error("Galera retries exhausted, design review needed");
}
```

## Example 8 : Application retry loop in Java

```java
// Java 17+, MariaDB Connector/J 3.x, MariaDB 10.6+ Galera cluster
import java.sql.*;
import java.util.Set;
import java.util.concurrent.ThreadLocalRandom;

public final class GaleraRetry {
    private static final Set<Integer> CERT_ERRORS = Set.of(1213, 1614);
    private static final int MAX_RETRIES = 5;
    private static final long BASE_BACKOFF_MS = 50;

    public interface Work { void run(Connection c) throws SQLException; }

    public static void runWithRetry(DataSource ds, Work work) throws SQLException {
        for (int attempt = 1; attempt <= MAX_RETRIES; attempt++) {
            try (Connection c = ds.getConnection()) {
                c.setAutoCommit(false);
                try {
                    work.run(c);
                    c.commit();
                    return;
                } catch (SQLException e) {
                    c.rollback();
                    if (CERT_ERRORS.contains(e.getErrorCode()) && attempt < MAX_RETRIES) {
                        long backoff = BASE_BACKOFF_MS * (1L << (attempt - 1))
                                       + ThreadLocalRandom.current().nextLong(25);
                        try { Thread.sleep(backoff); }
                        catch (InterruptedException ie) {
                            Thread.currentThread().interrupt();
                            throw new SQLException("Interrupted during backoff", ie);
                        }
                        continue;
                    }
                    throw e;
                }
            }
        }
        throw new SQLException("Galera retries exhausted, design review needed");
    }
}
```

## Example 9 : pc.weight for an asymmetric data-center deployment

Two nodes in DC-A, one node in DC-B. On a WAN split, DC-A should keep accepting writes (it has more nodes), DC-B should go read-only.

```sql
-- 10.6+ : on each node in DC-A, give it weight 2
SET GLOBAL wsrep_provider_options = 'pc.weight=2';
```

```sql
-- 10.6+ : the node in DC-B keeps the default weight 1
SET GLOBAL wsrep_provider_options = 'pc.weight=1';
```

```ini
# Persist in my.cnf so the weight survives a restart
# DC-A nodes :
[mariadb]
wsrep_provider_options = "pc.weight=2"

# DC-B node :
[mariadb]
wsrep_provider_options = "pc.weight=1"
```

After a WAN split : DC-A has total weight 4, DC-B has total weight 1. DC-A retains quorum (`PRIMARY`), DC-B becomes `NON_PRIMARY` and read-only. Without `pc.weight`, a 2-versus-1 split already favours DC-A by node count ; `pc.weight` makes the bias explicit and survives partial-node loss within DC-A.

## Example 10 : garbd arbitrator setup for a 2-node cluster

A 2-node cluster cannot form a stable quorum. Add `garbd` on a third host so there are 3 quorum votes.

```bash
# On a third host (NOT one of the two data nodes)
# MariaDB 10.6+ compatible galera-4
apt install galera-arbitrator-4
```

```ini
# /etc/default/garbd
GALERA_GROUP="prod_cluster"
GALERA_NODES="10.0.0.11:4567,10.0.0.12:4567"
LOG_FILE="/var/log/garbd.log"
```

```bash
# Start and enable
systemctl enable --now garbd
systemctl status garbd
```

```sql
-- 10.6+ : verify quorum vote count from a data node
SHOW STATUS LIKE 'wsrep_cluster_size';
-- garbd participates in quorum ; a 2-node + garbd setup survives the loss of one data node.
```

`garbd` stores no data, consumes a few MB of RAM, and exists purely to break the tie. NEVER place it on the same physical host as a data node.

## Example 11 : Recover a NON_PRIMARY partition

All nodes show `wsrep_cluster_status='NON_PRIMARY'` after a multi-DC outage. Bootstrap from the node with the latest committed data.

```sql
-- 10.6+ : run on EVERY node, record the seqno
SHOW STATUS LIKE 'wsrep_last_committed';
-- node1 : 184529
-- node2 : 184529
-- node3 : 184601   <- highest, has the latest data
```

```sql
-- 10.6+ : on node3 ONLY (the highest seqno)
SET GLOBAL wsrep_provider_options = 'pc.bootstrap=true';
```

```sql
-- 10.6+ : confirm node3 is now PRIMARY
SHOW STATUS LIKE 'wsrep_cluster_status';   -- PRIMARY
```

The other nodes then rejoin via IST or SST automatically. NEVER bootstrap from a node with a lower `wsrep_last_committed` ; that discards the writes that landed on the higher node and is unrecoverable without a backup.

## Example 12 : Size gcache to avoid forced SST

A node was down for 30 minutes ; on rejoin it falls back to a full SST because the donor's `gcache` no longer holds 30 minutes of write-sets. Size `gcache.size` for the expected outage window.

```sql
-- 10.6+ : measure the write-set production rate on a healthy node
-- sample wsrep_replicated_bytes twice, 60s apart
SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS
WHERE VARIABLE_NAME = 'WSREP_REPLICATED_BYTES';
-- delta over 60s / 60 = bytes per second
```

```ini
# my.cnf : if write rate is ~10 MB/s and outage budget is 1 hour :
# 10 MB/s * 3600 s * 1.5 safety = 54 GB
[mariadb]
wsrep_provider_options = "gcache.size=54G"
```

```bash
# gcache.size is NOT dynamic : restart the node to apply
systemctl restart mariadb
```

A node that rejoins WITHIN the gcache window uses IST (fast, incremental). A node beyond the window falls back to full SST (slow, full data copy). Sizing the gcache to the realistic worst-case outage keeps rejoins fast.

## Example 13 : Confirm a transaction failed certification (not standalone deadlock)

```sql
-- 10.6+ : after catching ER_LOCK_DEADLOCK, confirm it was a Galera path
SHOW VARIABLES LIKE 'wsrep_on';                    -- ON  -> cluster node
SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS
WHERE VARIABLE_NAME = 'WSREP_LOCAL_CERT_FAILURES'; -- did this just increment ?
```

If `wsrep_on=ON` AND the error landed on the `COMMIT` statement AND `wsrep_local_cert_failures` incremented, it was a certification failure. The fix is retry plus, if recurring, hot-row redesign. If the error landed mid-statement, it is a standalone InnoDB deadlock instead ; see `mariadb-errors-deadlocks`.
