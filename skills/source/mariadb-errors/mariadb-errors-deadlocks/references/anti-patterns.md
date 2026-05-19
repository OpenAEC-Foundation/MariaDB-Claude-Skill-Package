# Anti-Patterns : MariaDB Deadlocks and Lock-Wait

Ten real-world anti-patterns observed in production MariaDB / InnoDB workloads. Each entry has the SYMPTOM, the WRONG implementation, WHY it fails, and the CORRECT alternative.

---

## A-1 : Retrying a Single Statement Instead of the Whole Transaction

### Wrong

```python
# Python 3.10+, MariaDB 10.6+ : WRONG
cur.execute("START TRANSACTION")
cur.execute("UPDATE a SET x = 1 WHERE id = 1")
try:
   cur.execute("UPDATE b SET y = 2 WHERE id = 1")
except mariadb.OperationalError as e:
   if e.args[0] == 1213:
      cur.execute("UPDATE b SET y = 2 WHERE id = 1")  # WRONG
```

### Why it fails

When MariaDB detects a deadlock, it ROLLS BACK the ENTIRE transaction server-side and returns error 1213 to the client. The first `UPDATE a` is GONE. Retrying only the second `UPDATE b` runs it in a new implicit autocommit transaction, silently violating atomicity. The two updates are no longer atomic ; an observer can see only the `UPDATE b` change.

### Correct

Retry the ENTIRE transaction from `START TRANSACTION` :

```python
# Python 3.10+, MariaDB 10.6+ : CORRECT
for attempt in range(1, MAX_RETRIES + 1):
   try:
      cur.execute("START TRANSACTION")
      cur.execute("UPDATE a SET x = 1 WHERE id = 1")
      cur.execute("UPDATE b SET y = 2 WHERE id = 1")
      conn.commit()
      return
   except mariadb.OperationalError as e:
      conn.rollback()
      if e.args[0] == 1213 and attempt < MAX_RETRIES:
         time.sleep((2 ** attempt) * 0.05 + random.uniform(0, 0.05))
         continue
      raise
```

---

## A-2 : Disabling innodb_deadlock_detect on a High-Load Workload

### Wrong

```ini
# my.cnf : WRONG on a workload with cycle-risk
[mariadb]
innodb_deadlock_detect = OFF
```

### Why it fails

The InnoDB system-variables docs are explicit : "If set to off, deadlock detection is disabled and MariaDB will rely on innodb_lock_wait_timeout instead." That means a cycle is no longer detected ; both transactions just sit there waiting for each other until `innodb_lock_wait_timeout` (default 50 seconds) fires and ONE side gets error 1205. During those 50 seconds, the entire OLTP service can stall behind those two transactions, since every subsequent transaction that needs either lock also queues.

The original justification ("deadlock detection is expensive at high concurrency") was a 2016-era observation that was largely fixed in InnoDB ; the cost on modern hardware is negligible.

### Correct

ALWAYS leave at default `ON`. Fix the design problem (lock-order, hot-row, missing index) instead.

```ini
# my.cnf : CORRECT
[mariadb]
innodb_deadlock_detect = ON
innodb_print_all_deadlocks = ON
```

---

## A-3 : Using READ UNCOMMITTED to "Avoid Deadlocks"

### Wrong

```sql
-- 10.6+ : WRONG
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
```

### Why it fails

READ UNCOMMITTED allows DIRTY READS : your session sees data from OTHER transactions that have not yet committed and might roll back. You read values that never existed in any consistent state. This is almost never an acceptable trade for "fewer deadlocks".

Worse : READ UNCOMMITTED does NOT eliminate deadlocks on the WRITE side. UPDATE / DELETE / INSERT still take row-locks. The "avoidance" is only for the SELECT side, which was rarely the deadlock cause to begin with.

### Correct

If you need fewer deadlocks AND can tolerate phantom reads, use READ COMMITTED (gap locks disabled per the InnoDB lock-modes KB) :

```sql
-- 10.6+ : CORRECT for OLTP workloads that tolerate phantoms
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

If you need full serializability, accept that REPEATABLE READ produces more deadlocks and design your retry loop accordingly.

---

## A-4 : Inconsistent Lock Order Across Procedures

### Wrong

```sql
-- 10.6+ : WRONG : two procedures with opposite lock-order
DELIMITER //
CREATE PROCEDURE debit_then_audit(IN aid INT, IN amt DECIMAL(12,2))
BEGIN
   START TRANSACTION;
   UPDATE accounts   SET balance = balance - amt WHERE id = aid;
   INSERT INTO audit_log (account_id, action) VALUES (aid, 'debit');
   COMMIT;
END //
CREATE PROCEDURE audit_then_debit(IN aid INT, IN amt DECIMAL(12,2))
BEGIN
   START TRANSACTION;
   INSERT INTO audit_log (account_id, action) VALUES (aid, 'debit');
   UPDATE accounts   SET balance = balance - amt WHERE id = aid;
   COMMIT;
END //
DELIMITER ;
```

### Why it fails

Two concurrent sessions, one calling each procedure on the same `aid`, can deadlock : the first session holds the accounts row-X and waits for audit_log auto-inc, while the second session holds audit_log auto-inc and waits for the accounts row-X. Even with `innodb_autoinc_lock_mode=2` (no AUTO-INC lock), the implicit index-locks on the new audit_log row can still produce a cycle.

### Correct

Pick a canonical order (alphabetical, foreign-key dependency, whatever) and enforce it across the entire codebase :

```sql
-- 10.6+ : CORRECT : both procedures use accounts-first order
DELIMITER //
CREATE PROCEDURE debit_with_audit(IN aid INT, IN amt DECIMAL(12,2))
BEGIN
   START TRANSACTION;
   UPDATE accounts   SET balance = balance - amt WHERE id = aid;   -- ALWAYS first
   INSERT INTO audit_log (account_id, action) VALUES (aid, 'debit'); -- ALWAYS second
   COMMIT;
END //
DELIMITER ;
```

Code review MUST reject any new transaction that touches the same tables in a different order than its peers.

---

## A-5 : Treating a Galera Commit-Deadlock as a Bug

### Wrong

```python
# Python 3.10+, MariaDB 10.6+ Galera : WRONG
try:
   conn.commit()
except mariadb.OperationalError as e:
   if e.args[0] == 1213:
      send_alert("DATABASE DEADLOCK BUG, INVESTIGATE")  # WRONG
      raise
```

### Why it fails

Per vooronderzoek Cluster-3 §2 : Galera certification-based replication detects write-write conflicts between cluster nodes only at COMMIT time. The local node has no way to know during the transaction body that another node committed a conflicting write-set. So `ER_LOCK_DEADLOCK` (1213) on a Galera `COMMIT` is the EXPECTED outcome of the certification protocol, NOT a bug. The `wsrep_local_cert_failures` status variable tracks the cumulative count and is non-zero on any healthy multi-master cluster under write load.

Alerting on every certification-conflict produces hundreds or thousands of false-positive alerts per day. The team learns to ignore the alerts. When a REAL problem occurs, it is buried in noise.

### Correct

Handle Galera commit-deadlocks identically to standalone-InnoDB deadlocks : retry the transaction. Alert only on sudden rate-of-change in `wsrep_local_cert_failures`, never on absolute count.

```python
# Python 3.10+, MariaDB 10.6+ Galera : CORRECT
# Reuse the same retry loop as standalone InnoDB ; errno 1213 means RETRY
# whether the deadlock came from InnoDB cycle-detection or Galera certification.
```

---

## A-6 : Hot-Row Counter Table

### Wrong

```sql
-- 10.6+ : WRONG : single-row counter under high write-rate
CREATE TABLE global_counter (
   name   VARCHAR(64) PRIMARY KEY,
   value  BIGINT NOT NULL DEFAULT 0
) ENGINE=InnoDB;

INSERT INTO global_counter (name, value) VALUES ('page_views', 0);

-- Every page-view :
UPDATE global_counter SET value = value + 1 WHERE name = 'page_views';
```

### Why it fails

Every writer needs an X-lock on the SAME row. At 1000 RPS, that row becomes the bottleneck for the entire workload. Lock-waits pile up, `innodb_lock_wait_timeout` fires intermittently, and the application sees a flood of error 1205. No amount of lock-order discipline or isolation tuning helps : there is only ONE row, every writer wants it, only ONE writer can have it at a time.

### Correct

Three working alternatives, pick one :

```sql
-- Option A : sharded counter (see examples.md Example 9 for the full pattern)
CREATE TABLE global_counter_sharded (
   name   VARCHAR(64) NOT NULL,
   shard  TINYINT UNSIGNED NOT NULL,
   value  BIGINT NOT NULL DEFAULT 0,
   PRIMARY KEY (name, shard)
) ENGINE=InnoDB;

-- Option B : sequence object (10.3+ : MariaDB-only feature)
CREATE SEQUENCE page_view_seq START WITH 1 INCREMENT BY 1;
SELECT NEXT VALUE FOR page_view_seq;

-- Option C : append-only log + periodic roll-up
CREATE TABLE page_view_events (
   id  BIGINT AUTO_INCREMENT PRIMARY KEY,
   ts  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;
-- Hot writers do INSERT (no lock contention) ; nightly roll-up to summary.
```

---

## A-7 : Raising innodb_lock_wait_timeout to Mask a Long-Running Blocker

### Wrong

```sql
-- 10.6+ : WRONG : raising the timeout instead of fixing the blocker
SET GLOBAL innodb_lock_wait_timeout = 600;
```

### Why it fails

If transactions are timing out because OTHER transactions hold locks too long, raising the timeout just lets the SAME long blocker hold the locks even longer. The symptom shifts from "error 1205 every 50 seconds" to "10-minute application stalls". Throughput collapses ; user requests pile up in connection pools ; the upstream load balancer marks the DB host unhealthy.

The default 50 seconds was chosen by the InnoDB authors as a reasonable upper bound for a healthy OLTP transaction. If your transactions need more time, they are doing too much work in one transaction.

### Correct

Identify the blocker and SHORTEN it :

```sql
-- 10.6+ : find the oldest active transaction (the likely blocker)
SELECT trx_id, trx_mysql_thread_id, trx_started,
       TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS age_sec,
       SUBSTRING(trx_query, 1, 200) AS query
  FROM information_schema.INNODB_TRX
  ORDER BY trx_started ASC
  LIMIT 5;
```

Then : (a) split the blocker into smaller commits, (b) remove unnecessary `FOR UPDATE` clauses, (c) add indexes so the blocker touches fewer rows, (d) move read-only work outside the transaction.

ONLY raise `innodb_lock_wait_timeout` as a PER-SESSION override for a deliberate long-running maintenance script :

```sql
-- 10.6+ : CORRECT : session-scoped override for one script
SET SESSION innodb_lock_wait_timeout = 600;
```

---

## A-8 : No innodb_print_all_deadlocks in Production

### Wrong

```ini
# my.cnf : WRONG (default OFF)
[mariadb]
# innodb_print_all_deadlocks not set
```

### Why it fails

Without `innodb_print_all_deadlocks=ON`, the only way to see a deadlock is to run `SHOW ENGINE INNODB STATUS` and read the `LATEST DETECTED DEADLOCK` section. That shows ONLY the most-recent one. Earlier deadlocks are LOST. When you investigate a production incident hours later, the evidence is gone.

The cost of enabling the variable is one error-log write per deadlock, which is negligible even on the most write-heavy workloads.

### Correct

```ini
# my.cnf : CORRECT
[mariadb]
innodb_print_all_deadlocks = ON
```

Then tail the error log and feed deadlocks into your observability stack :

```bash
# 10.6+ : Debian/Ubuntu default path
tail -F /var/log/mysql/error.log | grep -A 50 'TRANSACTION:'
```

---

## A-9 : Optimistic Locking Without Retry on the App Side

### Wrong

```python
# Python 3.10+, MariaDB 10.6+ : WRONG : optimistic UPDATE without verifying affected-rows
cur.execute(
   "UPDATE accounts SET balance = ?, version = version + 1 WHERE id = ? AND version = ?",
   (new_balance, account_id, expected_version)
)
conn.commit()
return  # WRONG : did not check whether UPDATE affected 0 rows or 1 row
```

### Why it fails

Optimistic-concurrency relies on the application detecting the "lost update" case where `affected_rows = 0` (another writer bumped the version first). If the app ignores `affected_rows`, the silent failure looks like success to the user, the new balance is NEVER written, but no error is raised.

This is not technically a deadlock issue, but it appears in the same incident triage : "the transaction returned without error but the balance is wrong".

### Correct

Check `cur.rowcount` (mariadb connector) / `pdo->rowCount()` (PHP) / `executeUpdate()` return value (JDBC). If zero, the version-check failed ; retry by re-reading the row, re-computing the new value, and trying again.

```python
# Python 3.10+, MariaDB 10.6+ : CORRECT
for attempt in range(1, MAX_RETRIES + 1):
   cur.execute("SELECT balance, version FROM accounts WHERE id = ?", (account_id,))
   bal, ver = cur.fetchone()
   new_bal = compute_new(bal)
   cur.execute(
      "UPDATE accounts SET balance = ?, version = version + 1 WHERE id = ? AND version = ?",
      (new_bal, account_id, ver)
   )
   if cur.rowcount == 1:
      conn.commit()
      return
   # version mismatch ; retry with fresh read
```

---

## A-10 : Holding a Transaction Open Across a Network Round-Trip to an External Service

### Wrong

```python
# Python 3.10+, MariaDB 10.6+ : WRONG : external HTTP call inside transaction
conn.begin()
cur.execute("UPDATE accounts SET balance = balance - 100 WHERE id = 1")
response = requests.post("https://payments.example.com/charge", json={...})  # WRONG
if response.ok:
   cur.execute("INSERT INTO payments (status) VALUES ('ok')")
conn.commit()
```

### Why it fails

The transaction holds row-locks on `accounts` while waiting for the external HTTP response. If the external service is slow (1-5 seconds is typical for a payment gateway), every concurrent writer to `accounts` queues behind your transaction. At any reasonable load, this triggers `innodb_lock_wait_timeout` (default 50s) for OTHER transactions long before your slow service even responds.

This pattern also creates a correctness problem : if the network drops after the external service committed but before the local COMMIT lands, the local rollback leaves the systems out of sync.

### Correct

Use the "outbox pattern" : commit a row to a local outbox table, return success, then process the external call asynchronously.

```python
# Python 3.10+, MariaDB 10.6+ : CORRECT
conn.begin()
cur.execute("UPDATE accounts SET balance = balance - 100 WHERE id = 1")
cur.execute(
   "INSERT INTO payment_outbox (account_id, amount, status) VALUES (?, ?, 'pending')",
   (1, 100)
)
conn.commit()  # FAST : transaction holds locks for milliseconds

# Async worker picks up payment_outbox rows and calls the external service
# OUTSIDE any database transaction. On success, the worker updates the outbox row.
```

Result : transactions stay short, lock-waits disappear, retries on the external call are explicit and idempotent.

---

## Source Notes

- A-1, A-3, A-7, A-10 : standard InnoDB transaction-handling patterns confirmed against the InnoDB system-variables docs.
- A-2 : `innodb_deadlock_detect` semantics confirmed via WebFetch against `mariadb.com/docs/server/server-usage/storage-engines/innodb/innodb-system-variables`.
- A-4 : lock-order discipline is the canonical InnoDB advice ; confirmed via lock-modes KB.
- A-5 : Galera certification-conflict semantics from vooronderzoek Cluster-3 §2 and the Galera cluster KB.
- A-6 : hot-row contention pattern is universal across InnoDB-style engines.
- A-8 : `innodb_print_all_deadlocks` default OFF confirmed via WebFetch against `mariadb.com/kb/en/innodb-system-variables/`.
- A-9 : optimistic-concurrency pattern is application-side ; the InnoDB MVCC docs back the version-check approach.
