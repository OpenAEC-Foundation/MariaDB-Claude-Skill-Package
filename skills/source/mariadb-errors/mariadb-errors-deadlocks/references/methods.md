# Methods Reference : MariaDB Deadlocks and Lock-Wait

Complete reference for lock-mode taxonomy, system variables, isolation-level impact, and the line-by-line reading of `SHOW ENGINE INNODB STATUS` deadlock-section.

## 1. Lock-Type Taxonomy

### 1.1 Row-Level Locks

| Mode | Symbol | Held while | Compatible with | Blocks |
|------|--------|-----------|-----------------|--------|
| Shared | S | A row is being read under a locking SELECT (`SELECT ... LOCK IN SHARE MODE`, or implicitly at SERIALIZABLE) | S | X |
| Exclusive | X | A row is being modified (`UPDATE`, `DELETE`, `INSERT`, `SELECT ... FOR UPDATE`) | nothing | S, X |

Per the InnoDB lock-modes KB :
- Shared lock : "obtained to read a row, and allows other transactions to read the locked row, but not to write to the locked row."
- Exclusive lock : "obtained to write to a row, and stops other transactions from locking the same row."

### 1.2 Intention Locks (Table-Level)

Intention locks are TABLE-level locks that signal what kind of row-level locks the transaction intends to acquire on rows in the table. They prevent another transaction from taking an incompatible TABLE-level lock (such as `LOCK TABLES ... WRITE`).

| Mode | Symbol | Meaning |
|------|--------|---------|
| Intention Shared | IS | "Indicates that a transaction intends to set a shared lock" on some row |
| Intention Exclusive | IX | "Indicates that a transaction intends to set an exclusive lock" on some row |

### 1.3 Compatibility Matrix

Per the InnoDB lock-modes KB :
- X blocks all other locks.
- S is blocked by X or IX.
- IX is blocked by X or S.
- IS is blocked only by X.

| Requested \ Held | none | IS | IX | S | X |
|------------------|------|----|----|---|---|
| IS | OK | OK | OK | OK | wait |
| IX | OK | OK | OK | wait | wait |
| S | OK | OK | wait | OK | wait |
| X | OK | wait | wait | wait | wait |

### 1.4 Record / Gap / Next-Key Locks

InnoDB row-locks are actually INDEX-record locks. The distinction matters because gaps between index entries can also be locked.

| Lock | Locks what | Purpose | When used |
|------|-----------|---------|-----------|
| Record lock | A single index record | Prevent UPDATE/DELETE of that row | All isolation levels |
| Gap lock | The open interval BETWEEN two index records | Prevent INSERT of new rows into the gap (phantom-prevention) | REPEATABLE READ, SERIALIZABLE |
| Next-key lock | Record + gap BEFORE the record | Default lock for range scans at REPEATABLE READ | REPEATABLE READ, SERIALIZABLE |
| Insert-intention lock | A gap, advertising INSERT intent | Allows concurrent INSERTs into the same gap if rows don't collide | INSERT path under any level |
| AUTO-INC lock | Table-level lock on the AUTO_INCREMENT counter | Serializes assignment of AUTO_INCREMENT values | Modulated by `innodb_autoinc_lock_mode` |

KB confirmation : "a lock is held on the gap before the index record, so that another transaction cannot insert a new index record in the gap between the record and the preceding record" and "Gap locks are disabled if the isolation level is set to READ COMMITTED".

### 1.5 AUTO-INC Lock Modes

`innodb_autoinc_lock_mode` controls AUTO_INCREMENT lock behavior :

| Value | Mode | Behavior |
|-------|------|----------|
| 0 | Traditional | Table-level AUTO-INC lock held for the duration of the statement |
| 1 | Consecutive (default before 10.6 Galera-OFF) | Bulk INSERT holds the lock briefly to pre-allocate ; simple INSERT does NOT take the lock |
| 2 | Interleaved | No AUTO-INC lock ; values may be non-monotonic across concurrent inserts. REQUIRED for Galera |

On Galera nodes set `innodb_autoinc_lock_mode=2` per the Galera Cluster KB.

## 2. System Variables

### 2.1 innodb_deadlock_detect

| Property | Value |
|----------|-------|
| Default | `ON` |
| Type | Boolean |
| Scope | Global |
| Dynamic | Yes |
| Versions | 10.6+ |

Per the InnoDB system variables docs : "By default, the InnoDB deadlock detector is enabled. If set to off, deadlock detection is disabled and MariaDB will rely on innodb_lock_wait_timeout instead."

```sql
-- 10.6+
SHOW GLOBAL VARIABLES LIKE 'innodb_deadlock_detect';
SET GLOBAL innodb_deadlock_detect = ON;
```

ALWAYS leave at default `ON`. The only justification for `OFF` is a workload with NO cycle-risk (read-mostly), and even then the cost of detection is negligible.

### 2.2 innodb_lock_wait_timeout

| Property | Value |
|----------|-------|
| Default | `50` (seconds) |
| Type | Numeric |
| Scope | Global, Session |
| Range | `1` to `1073741824` |
| Dynamic | Yes |
| Versions | 10.6+ |

Controls how long a transaction waits for a row-lock before timing out with error 1205 `ER_LOCK_WAIT_TIMEOUT_EXCEEDED`.

```sql
-- 10.6+ : per-session override for a long-running maintenance script
SET SESSION innodb_lock_wait_timeout = 300;
```

ALWAYS prefer a per-session override for long-running scripts. NEVER raise the GLOBAL value as a "fix" for a chronic blocker ; that just delays the symptom.

### 2.3 innodb_print_all_deadlocks

| Property | Value |
|----------|-------|
| Default | `OFF` |
| Type | Boolean |
| Scope | Global |
| Dynamic | Yes |
| Versions | 10.6+ |

When ON, every detected deadlock is logged to the MariaDB error log (not just the latest one visible in `SHOW ENGINE INNODB STATUS`).

```sql
-- 10.6+
SET GLOBAL innodb_print_all_deadlocks = ON;
```

ALWAYS turn ON in production. The cost is negligible (one error-log write per deadlock) and the forensic value is high. Without this, you only ever see the MOST RECENT deadlock per `SHOW ENGINE INNODB STATUS` invocation, which loses history.

### 2.4 transaction_isolation

| Property | Value |
|----------|-------|
| Default | `REPEATABLE-READ` |
| Type | Enum |
| Values | `READ-UNCOMMITTED`, `READ-COMMITTED`, `REPEATABLE-READ`, `SERIALIZABLE` |
| Scope | Global, Session |
| Dynamic | Yes |
| Versions | 10.6+ (renamed from `tx_isolation` for MySQL 8 compatibility) |

```sql
-- 10.6+
SET SESSION transaction_isolation = 'READ-COMMITTED';
SET GLOBAL transaction_isolation = 'REPEATABLE-READ';
```

Equivalent SQL-standard form :

```sql
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
SET GLOBAL TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

Per the SET TRANSACTION docs, omitting GLOBAL and SESSION applies the level to the NEXT (not-yet-started) transaction only, after which the session value applies again.

## 3. Reading SHOW ENGINE INNODB STATUS Deadlock-Section

The output of `SHOW ENGINE INNODB STATUS\G` contains a section starting with `LATEST DETECTED DEADLOCK`. Annotated walkthrough :

```
------------------------
LATEST DETECTED DEADLOCK
------------------------
2026-05-19 12:34:56 0x7f8a2c0c1700               <-- timestamp + thread id
*** (1) TRANSACTION:                              <-- the first transaction
TRANSACTION 12345, ACTIVE 3 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 4 lock struct(s), heap size 1136, 3 row lock(s)
MariaDB thread id 42, OS thread handle 140234..., query id 100 localhost user
UPDATE accounts SET balance = balance - 100 WHERE id = 1   <-- the statement that waited
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:      <-- what (1) wanted
RECORD LOCKS space id 5 page no 4 n bits 72 index PRIMARY of table `app`.`accounts`
trx id 12345 lock_mode X locks rec but not gap waiting

*** (2) TRANSACTION:                              <-- the second transaction
TRANSACTION 12346, ACTIVE 4 sec starting index read
mysql tables in use 1, locked 1
6 lock struct(s), heap size 1136, 5 row lock(s)
MariaDB thread id 43, OS thread handle 140234..., query id 101 localhost user
UPDATE accounts SET balance = balance + 100 WHERE id = 2
*** (2) HOLDS THE LOCK(S):                        <-- what (2) is holding
RECORD LOCKS space id 5 page no 4 n bits 72 index PRIMARY of table `app`.`accounts`
trx id 12346 lock_mode X locks rec but not gap
*** (2) WAITING FOR THIS LOCK TO BE GRANTED:      <-- what (2) wants from (1)
RECORD LOCKS space id 5 page no 4 n bits 72 index PRIMARY of table `app`.`accounts`
trx id 12346 lock_mode X locks rec but not gap waiting

*** WE ROLL BACK TRANSACTION (1)                  <-- the chosen victim
```

Section meanings :

| Section | Meaning |
|---------|---------|
| `*** (N) TRANSACTION:` | Header for transaction N (always 2 transactions on a deadlock cycle) |
| `LOCK WAIT` | This transaction is currently waiting on a lock |
| `lock struct(s)` | Internal lock structures held |
| `row lock(s)` | Actual rows locked |
| `WAITING FOR THIS LOCK TO BE GRANTED` | The lock this transaction needed but could not get |
| `HOLDS THE LOCK(S)` | Locks this transaction has already acquired (incompatible with the other side's wait) |
| `*** WE ROLL BACK TRANSACTION (N)` | Which transaction was chosen as victim |

Lock-mode encoding :

| String | Meaning |
|--------|---------|
| `lock_mode X` | Exclusive |
| `lock_mode S` | Shared |
| `locks rec but not gap` | Pure record lock (no gap) |
| `locks gap before rec` | Pure gap lock |
| `locks rec but not gap insert intention` | Insert-intention lock |
| `locks rec` (no modifier) | Next-key lock (record + gap before) |

### 3.1 Victim Selection

InnoDB picks the transaction with the FEWEST modified rows as the victim (cheapest rollback). This means a small new transaction will often be killed in favor of a long-running batch. The application MUST be ready to retry on either side.

## 4. Isolation-Level Lock Implications

| Level | Reads use locks ? | Gap locks ? | Next-key locks ? | Phantom reads ? | Dirty reads ? |
|-------|-------------------|-------------|------------------|-----------------|---------------|
| READ UNCOMMITTED | No | No | No | Yes | YES |
| READ COMMITTED | No (snapshot per stmt) | NO (disabled per KB) | NO | Yes | No |
| REPEATABLE READ (default) | No (snapshot per txn) | Yes | Yes (on range scans) | No | No |
| SERIALIZABLE | YES (plain SELECT becomes locking when autocommit OFF) | Yes (broad) | Yes (broad) | No | No |

Per the SET TRANSACTION docs : "SERIALIZABLE - Converts plain SELECT statements to locked reads when autocommit is disabled". This means SELECT can deadlock at SERIALIZABLE, which is rarely intended.

## 5. Inspecting Live Locks

```sql
-- 10.6+ : list active InnoDB transactions
SELECT trx_id, trx_state, trx_started, trx_query
  FROM information_schema.INNODB_TRX
  ORDER BY trx_started ASC;

-- 10.6+ : list InnoDB locks (deprecated in newer versions ; check version)
SELECT * FROM information_schema.INNODB_LOCKS;

-- 10.6+ : list lock-waits
SELECT * FROM information_schema.INNODB_LOCK_WAITS;
```

`INNODB_LOCKS` and `INNODB_LOCK_WAITS` were moved to `performance_schema.data_locks` and `performance_schema.data_lock_waits` in newer MariaDB versions ; verify per version when scripting.

## 6. Killing a Blocker

```sql
-- 10.6+ : find the blocker thread id from INNODB_TRX
SELECT trx_mysql_thread_id, trx_started, trx_query
  FROM information_schema.INNODB_TRX
  ORDER BY trx_started ASC LIMIT 1;

-- Kill the blocker (forces transaction rollback)
KILL <thread-id>;
```

`KILL` rolls back the target transaction. The application connected via that thread receives a connection-loss error and must reconnect. NEVER `KILL` a long-running OLTP transaction without checking what it is doing ; you may abort legitimate work.

## 7. Galera-Specific Status

```sql
-- Galera node : certification-failure counter (cumulative since node start)
SHOW STATUS LIKE 'wsrep_local_cert_failures';

-- Local-node send/recv queue lengths (high values mean cluster congestion)
SHOW STATUS LIKE 'wsrep_local_send_queue%';
SHOW STATUS LIKE 'wsrep_local_recv_queue%';

-- BF-abort counter : transactions aborted by an incoming write-set
SHOW STATUS LIKE 'wsrep_local_bf_aborts';
```

A non-zero `wsrep_local_cert_failures` is EXPECTED on a healthy multi-master cluster under write load. The application MUST handle `ER_LOCK_DEADLOCK` at COMMIT as a normal retry case.

## 8. References (verified via WebFetch)

- InnoDB lock modes : `mariadb.com/docs/server/server-usage/storage-engines/innodb/innodb-lock-modes`
- InnoDB system variables (deadlock detect, lock wait timeout, print all deadlocks) : `mariadb.com/docs/server/server-usage/storage-engines/innodb/innodb-system-variables` and `mariadb.com/kb/en/innodb-system-variables/`
- SET TRANSACTION ISOLATION LEVEL : `mariadb.com/docs/server/reference/sql-statements/administrative-sql-statements/set-commands/set-transaction`
- Galera certification semantics : vooronderzoek Cluster-3 §2
