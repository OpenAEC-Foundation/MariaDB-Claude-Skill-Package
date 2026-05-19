# Examples : MariaDB Deadlocks and Lock-Wait

Twelve worked examples covering reproduction, diagnosis, and remediation.

## Setup

All examples assume :

```sql
-- 10.6+
CREATE DATABASE IF NOT EXISTS lockdemo;
USE lockdemo;

CREATE TABLE accounts (
   id        INT PRIMARY KEY,
   balance   DECIMAL(12, 2) NOT NULL,
   updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
              ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB;

CREATE TABLE audit_log (
   id          BIGINT AUTO_INCREMENT PRIMARY KEY,
   account_id  INT NOT NULL,
   action      VARCHAR(64) NOT NULL,
   ts          TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;

INSERT INTO accounts (id, balance) VALUES (1, 1000.00), (2, 1000.00);
```

## Example 1 : Reproduce a Classic Two-Transaction Deadlock

Two sessions, opposite lock-order. Run each block in a SEPARATE client session.

```sql
-- 10.6+
-- Session A
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
-- Now switch to Session B before continuing.
```

```sql
-- 10.6+
-- Session B
START TRANSACTION;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
-- Now switch back to Session A.
```

```sql
-- 10.6+
-- Session A : tries to update id=2 which Session B holds
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
-- This session blocks.
```

```sql
-- 10.6+
-- Session B : tries to update id=1 which Session A holds
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
-- InnoDB detects the cycle. ONE session gets :
--   ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
-- The other session's UPDATE proceeds. The victim must ROLLBACK and retry.
```

The fix : both sessions MUST update id=1 first, then id=2. Same canonical order.

## Example 2 : Inspect the Deadlock via SHOW ENGINE INNODB STATUS

Immediately after reproducing Example 1 :

```sql
-- 10.6+
SHOW ENGINE INNODB STATUS\G
```

Look for the `LATEST DETECTED DEADLOCK` block. It shows both transactions, what each held, what each waited for, and which one was rolled back (`*** WE ROLL BACK TRANSACTION (N)`). See `methods.md` §3 for the line-by-line walkthrough.

## Example 3 : Enable Persistent Deadlock Logging

```sql
-- 10.6+
SET GLOBAL innodb_print_all_deadlocks = ON;
```

Now every deadlock is written to the MariaDB error log :

```bash
# 10.6+ : default error-log path on Debian/Ubuntu
tail -f /var/log/mysql/error.log | grep -A 30 'DEADLOCK'
```

To persist across restarts, add to `my.cnf` :

```ini
# /etc/mysql/mariadb.conf.d/99-deadlock-logging.cnf (10.6+)
[mariadb]
innodb_print_all_deadlocks = ON
```

## Example 4 : Python Retry Loop (mariadb connector 1.1+)

```python
# Python 3.10+, mariadb 1.1+, MariaDB 10.6+
import mariadb, time, random, logging

MAX_RETRIES = 4
DEADLOCK_ERRNO = 1213
LOCKWAIT_ERRNO = 1205

def transfer(conn, src_id: int, dst_id: int, amount):
    for attempt in range(1, MAX_RETRIES + 1):
        cur = conn.cursor()
        try:
            cur.execute("START TRANSACTION")
            # Canonical lock-order : ALWAYS lower id first.
            lo, hi = sorted([src_id, dst_id])
            cur.execute("SELECT balance FROM accounts WHERE id = ? FOR UPDATE", (lo,))
            cur.execute("SELECT balance FROM accounts WHERE id = ? FOR UPDATE", (hi,))
            cur.execute("UPDATE accounts SET balance = balance - ? WHERE id = ?", (amount, src_id))
            cur.execute("UPDATE accounts SET balance = balance + ? WHERE id = ?", (amount, dst_id))
            cur.execute("INSERT INTO audit_log (account_id, action) VALUES (?, ?)",
                        (src_id, f"transfer -{amount} to {dst_id}"))
            conn.commit()
            return
        except mariadb.OperationalError as e:
            errno = e.args[0] if e.args else None
            conn.rollback()
            if errno == DEADLOCK_ERRNO and attempt < MAX_RETRIES:
                backoff = (2 ** attempt) * 0.05 + random.uniform(0, 0.05)
                logging.warning("Deadlock attempt %d, sleep %.3fs", attempt, backoff)
                time.sleep(backoff)
                continue
            raise
        finally:
            cur.close()
```

Key points : (1) canonical lock-order via `sorted([src_id, dst_id])`, (2) retry on 1213 only (NEVER on 1205 without investigating the blocker), (3) exponential backoff with jitter, (4) bounded retry count.

## Example 5 : PHP Retry Loop (PDO mysql)

```php
<?php
// PHP 8.1+, ext-pdo_mysql, MariaDB 10.6+
function transfer(PDO $pdo, int $srcId, int $dstId, string $amount): void {
    $maxRetries = 4;
    for ($attempt = 1; $attempt <= $maxRetries; $attempt++) {
        try {
            $pdo->beginTransaction();
            [$lo, $hi] = $srcId < $dstId ? [$srcId, $dstId] : [$dstId, $srcId];
            $sel = $pdo->prepare("SELECT balance FROM accounts WHERE id = ? FOR UPDATE");
            $sel->execute([$lo]);
            $sel->execute([$hi]);
            $upd = $pdo->prepare("UPDATE accounts SET balance = balance + ? WHERE id = ?");
            $upd->execute([-$amount, $srcId]);
            $upd->execute([$amount, $dstId]);
            $pdo->commit();
            return;
        } catch (PDOException $e) {
            $pdo->rollBack();
            // SQLSTATE 40001 = deadlock ; check both error code and SQLSTATE
            $isDeadlock = $e->getCode() === '40001' ||
                          (isset($e->errorInfo[1]) && $e->errorInfo[1] === 1213);
            if ($isDeadlock && $attempt < $maxRetries) {
                $sleepMs = (2 ** $attempt) * 50 + random_int(0, 50);
                usleep($sleepMs * 1000);
                continue;
            }
            throw $e;
        }
    }
}
```

## Example 6 : Node.js Retry Loop (mariadb 3.x)

```javascript
// Node.js 20+, mariadb 3.x, MariaDB 10.6+
const mariadb = require('mariadb');

const DEADLOCK_ERRNO = 1213;
const MAX_RETRIES = 4;

async function transfer(pool, srcId, dstId, amount) {
   for (let attempt = 1; attempt <= MAX_RETRIES; attempt++) {
      const conn = await pool.getConnection();
      try {
         await conn.beginTransaction();
         const [lo, hi] = srcId < dstId ? [srcId, dstId] : [dstId, srcId];
         await conn.query('SELECT balance FROM accounts WHERE id = ? FOR UPDATE', [lo]);
         await conn.query('SELECT balance FROM accounts WHERE id = ? FOR UPDATE', [hi]);
         await conn.query('UPDATE accounts SET balance = balance - ? WHERE id = ?', [amount, srcId]);
         await conn.query('UPDATE accounts SET balance = balance + ? WHERE id = ?', [amount, dstId]);
         await conn.commit();
         return;
      } catch (err) {
         await conn.rollback().catch(() => {});
         if (err.errno === DEADLOCK_ERRNO && attempt < MAX_RETRIES) {
            const backoff = Math.pow(2, attempt) * 50 + Math.random() * 50;
            await new Promise((r) => setTimeout(r, backoff));
            continue;
         }
         throw err;
      } finally {
         conn.release();
      }
   }
}
```

## Example 7 : Java Retry Loop (MariaDB Connector/J 3.x)

```java
// Java 17+, mariadb-java-client 3.x, MariaDB 10.6+
import java.sql.*;

public final class TransferService {
   private static final int DEADLOCK_ERRNO = 1213;
   private static final int MAX_RETRIES = 4;

   public void transfer(DataSource ds, int srcId, int dstId, java.math.BigDecimal amount)
         throws SQLException {
      for (int attempt = 1; attempt <= MAX_RETRIES; attempt++) {
         try (Connection conn = ds.getConnection()) {
            conn.setAutoCommit(false);
            int lo = Math.min(srcId, dstId);
            int hi = Math.max(srcId, dstId);
            try (PreparedStatement ps = conn.prepareStatement(
                  "SELECT balance FROM accounts WHERE id = ? FOR UPDATE")) {
               ps.setInt(1, lo); ps.executeQuery().close();
               ps.setInt(1, hi); ps.executeQuery().close();
            }
            try (PreparedStatement ps = conn.prepareStatement(
                  "UPDATE accounts SET balance = balance + ? WHERE id = ?")) {
               ps.setBigDecimal(1, amount.negate()); ps.setInt(2, srcId); ps.executeUpdate();
               ps.setBigDecimal(1, amount);          ps.setInt(2, dstId); ps.executeUpdate();
            }
            conn.commit();
            return;
         } catch (SQLException e) {
            if (e.getErrorCode() == DEADLOCK_ERRNO && attempt < MAX_RETRIES) {
               try {
                  Thread.sleep((long) (Math.pow(2, attempt) * 50 + Math.random() * 50));
               } catch (InterruptedException ie) {
                  Thread.currentThread().interrupt();
                  throw new SQLException("interrupted", ie);
               }
               continue;
            }
            throw e;
         }
      }
   }
}
```

## Example 8 : Switch a Session to READ COMMITTED

When an OLTP service tolerates phantom reads and you want to reduce deadlock incidence :

```sql
-- 10.6+
-- Per-connection : every session sets at connect time
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- Per-transaction only : applies to the next-started transaction
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
START TRANSACTION;
-- ...
COMMIT;
-- Session reverts to its session-level isolation here.
```

In Spring Boot (Hikari datasource) :

```yaml
spring:
  datasource:
    hikari:
      transaction-isolation: TRANSACTION_READ_COMMITTED
```

Per the SET TRANSACTION docs : omitting GLOBAL and SESSION applies the level to the next not-yet-started transaction only ; the session value applies again after that.

NEVER change the GLOBAL isolation without auditing every read path. The semantic shift (gap locks dropped, phantoms allowed) can silently corrupt invariants that depended on REPEATABLE READ.

## Example 9 : Redesign a Hot-Row Counter with Sharding

PROBLEM :

```sql
-- 10.6+ : hot-row design that deadlocks under load
CREATE TABLE site_visit_counter (
   site_id  INT PRIMARY KEY,
   visits   BIGINT NOT NULL DEFAULT 0
) ENGINE=InnoDB;

-- Every page-view :
UPDATE site_visit_counter SET visits = visits + 1 WHERE site_id = 1;
-- Under 1000 RPS, all writers contend on the same row.
```

FIX : shard the counter across N rows, sum at read time.

```sql
-- 10.6+
CREATE TABLE site_visit_counter_sharded (
   site_id  INT NOT NULL,
   shard    TINYINT UNSIGNED NOT NULL,
   visits   BIGINT NOT NULL DEFAULT 0,
   PRIMARY KEY (site_id, shard)
) ENGINE=InnoDB;

-- Insert 16 shard rows per site_id at site creation time :
INSERT INTO site_visit_counter_sharded (site_id, shard, visits)
SELECT 1, n, 0 FROM (
   SELECT 0 n UNION SELECT 1 UNION SELECT 2 UNION SELECT 3
   UNION SELECT 4 UNION SELECT 5 UNION SELECT 6 UNION SELECT 7
   UNION SELECT 8 UNION SELECT 9 UNION SELECT 10 UNION SELECT 11
   UNION SELECT 12 UNION SELECT 13 UNION SELECT 14 UNION SELECT 15
) s;

-- Every page-view picks a random shard :
-- Pseudocode for application : shard = random() % 16
UPDATE site_visit_counter_sharded
   SET visits = visits + 1
   WHERE site_id = 1 AND shard = 7;

-- Read total :
SELECT SUM(visits) AS total
  FROM site_visit_counter_sharded
  WHERE site_id = 1;
```

The 16x sharding cuts contention by 16x. Choose N proportional to peak concurrent-writer count.

## Example 10 : Galera Retry Pattern

On a Galera cluster, the same retry loop as Example 4 works for both standalone deadlocks AND certification-deadlocks-at-COMMIT. The application does NOT need to distinguish them.

```python
# Python 3.10+, mariadb 1.1+, MariaDB 10.6+ Galera-4
# Identical to Example 4 ; the loop catches errno 1213 whether the deadlock came from
# InnoDB cycle-detection or Galera certification.
```

For observability, watch the Galera-specific status :

```sql
SHOW STATUS LIKE 'wsrep_local_cert_failures';
SHOW STATUS LIKE 'wsrep_local_bf_aborts';
```

ALWAYS alert on rate-of-change (sudden spike), NEVER on absolute count. A healthy multi-master cluster has a non-zero baseline.

## Example 11 : Diagnose a Lock-Wait Timeout (Not a Deadlock)

When you get error 1205 instead of 1213, there is no cycle ; another transaction is just holding the lock too long.

```sql
-- 10.6+ : find currently-active transactions, oldest first
SELECT trx_id, trx_mysql_thread_id, trx_started,
       TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS age_sec,
       SUBSTRING(trx_query, 1, 200) AS query
  FROM information_schema.INNODB_TRX
  ORDER BY trx_started ASC
  LIMIT 10;
```

If the oldest transaction is older than your `innodb_lock_wait_timeout` (default 50s), it is the prime suspect. Inspect what it is doing :

```sql
-- 10.6+ : full process-list, look for the matching thread id
SHOW FULL PROCESSLIST;
```

The fix is to make the BLOCKING transaction shorter (split into smaller commits, remove unnecessary `FOR UPDATE`, add indexes so it touches fewer rows), NOT to raise `innodb_lock_wait_timeout`.

## Example 12 : Single-Statement Retry Is WRONG, Whole-Transaction Retry Is RIGHT

WRONG :

```python
# Python 3.10+, MariaDB 10.6+ : WRONG pattern
cur.execute("START TRANSACTION")
cur.execute("UPDATE a SET x = 1 WHERE id = 1")
try:
   cur.execute("UPDATE b SET y = 2 WHERE id = 1")     # deadlocks
except mariadb.OperationalError as e:
   if e.args[0] == 1213:
      cur.execute("UPDATE b SET y = 2 WHERE id = 1")  # WRONG : transaction was rolled back
                                                       # by the server, this runs in a NEW
                                                       # implicit autocommit context
```

After a deadlock, the SERVER has already rolled back the whole transaction. Retrying just the failed statement runs it OUTSIDE the original transaction, which silently violates atomicity.

RIGHT : retry the ENTIRE transaction from `START TRANSACTION` (see Example 4).

## Notes on Source Verification

All variable defaults, error codes, lock-mode semantics, and isolation-level behavior were verified via WebFetch against `mariadb.com/docs/server/server-usage/storage-engines/innodb/*` and `mariadb.com/kb/en/innodb-system-variables/`. Galera retry semantics confirmed against vooronderzoek Cluster-3 §2.
