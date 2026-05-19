---
name: mariadb-errors-deadlocks
description: >
  Use when ER_LOCK_DEADLOCK or ER_LOCK_WAIT_TIMEOUT errors appear, designing lock-discipline for concurrent transactions, reading SHOW ENGINE INNODB STATUS deadlock section, tuning innodb_deadlock_detect, or handling Galera certification-COMMIT deadlocks.
  Prevents the common mistake of expecting deadlocks to vanish via retry-without-design-fix, missing gap-lock dynamics, or treating Galera ER_LOCK_DEADLOCK as a bug instead of expected outcome.
  Covers InnoDB deadlock detection (innodb_deadlock_detect ON default), reading deadlock section of SHOW ENGINE INNODB STATUS, lock-order discipline (always-same-table-order), gap locks vs record locks vs next-key locks, transaction isolation levels, application-layer retry strategy, Galera COMMIT-time deadlock semantics.
  Keywords: deadlock, ER_LOCK_DEADLOCK, ER_LOCK_WAIT_TIMEOUT, 1213, 1205, SHOW ENGINE INNODB STATUS, innodb_lock_wait_timeout, innodb_deadlock_detect, gap lock, next-key lock, record lock, lock order, retry transaction, REPEATABLE READ, READ COMMITTED, Galera deadlock at commit, my transaction keeps failing, lock wait timeout exceeded, transaction was deadlocked, why does my insert hang, my update is stuck
license: MIT
compatibility: "Designed for Claude Code. Requires MariaDB 10.6-LTS, 10.11-LTS, 11.x, 12.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# MariaDB Deadlocks and Lock-Wait Errors

How to diagnose, recover from, and prevent InnoDB deadlocks (error 1213) and lock-wait-timeout errors (error 1205), including the Galera certification-deadlock-at-COMMIT case that is normal behavior on a multi-master cluster.

## Quick Reference

- `ER_LOCK_DEADLOCK` (error 1213, SQLSTATE 40001) is RECOVERABLE : ALWAYS retry the ENTIRE transaction, not just the failed statement.
- `ER_LOCK_WAIT_TIMEOUT_EXCEEDED` (error 1205) means a transaction waited `innodb_lock_wait_timeout` seconds (default 50) for a row-lock and was killed. Not a deadlock ; it usually means a long-running other transaction is holding the lock.
- `innodb_deadlock_detect=ON` (default, global, dynamic) auto-aborts the smaller transaction on cycle-detection. NEVER turn this OFF on a workload with cycle-risk ; the alternative is silent stalls until `innodb_lock_wait_timeout`.
- `innodb_print_all_deadlocks=ON` (default OFF, global) writes every detected deadlock to the error log. ALWAYS turn this ON in production for forensic visibility.
- Lock-order discipline : ALWAYS acquire locks on tables in the same order across all transactions. Inconsistent order is the #1 cause of avoidable deadlocks.
- Galera certification-deadlock at `COMMIT` is NORMAL behavior, NOT a bug. The certification protocol detects write-write conflicts only at commit-time. ALWAYS retry on `ER_LOCK_DEADLOCK` returned from `COMMIT`.
- `READ COMMITTED` disables gap locks (per the InnoDB lock-modes KB), reducing deadlock count but allowing phantom-reads. Switch only after you understand the consistency trade-off.
- NEVER use `READ UNCOMMITTED` to "avoid deadlocks". It allows dirty reads of uncommitted data and is almost never the right answer.

## Error Identification

| Symptom | Error number | Symbol | SQLSTATE | Meaning |
|---------|--------------|--------|----------|---------|
| Transaction rolled back automatically | 1213 | `ER_LOCK_DEADLOCK` | 40001 | InnoDB detected a lock-cycle and aborted this transaction |
| Statement returned after ~50 seconds with "Lock wait timeout exceeded" | 1205 | `ER_LOCK_WAIT_TIMEOUT_EXCEEDED` | HY000 | A row-lock was held by another transaction longer than `innodb_lock_wait_timeout` |
| Galera `COMMIT` returned deadlock on a transaction that touched no obviously-contended row | 1213 | `ER_LOCK_DEADLOCK` (cluster) | 40001 | Certification-conflict on the cluster ; another node committed a write-set that conflicts with yours |

The two errors are DIFFERENT and the fix path is DIFFERENT :
- 1213 means the cycle is GONE (server already rolled you back) : retry.
- 1205 means another transaction is STILL holding the lock : find and kill the blocker, do not just retry blindly.

## Decision Tree

```
Got a lock-related error
|
+-- Error 1213 (ER_LOCK_DEADLOCK) ?
|     |
|     +-- Is this Galera (wsrep_on=ON) and did it happen at COMMIT ?
|     |     YES : certification-conflict, NORMAL. Retry full transaction.
|     |     NO  : standard InnoDB deadlock. Retry full transaction.
|     |
|     +-- Has the same transaction deadlocked > 3 times ?
|           YES : design problem (hot-row / lock-order). Stop retrying ; fix the design.
|           NO  : exponential-backoff retry, max 3-5 attempts.
|
+-- Error 1205 (ER_LOCK_WAIT_TIMEOUT_EXCEEDED) ?
      |
      +-- Find blocking transaction :
      |   SELECT * FROM information_schema.INNODB_TRX
      |     ORDER BY trx_started ASC LIMIT 5;
      |
      +-- Is the blocker a long-running OLTP transaction ?
      |     YES : the blocker has a design problem (transaction holding locks too long).
      |           Fix the blocker, do NOT just raise innodb_lock_wait_timeout.
      |     NO  : transient ; retry once after a short backoff.
      |
      +-- Has this happened repeatedly under load ?
            YES : check innodb_buffer_pool_size, hot-row contention, missing indexes
                  causing full-scan locks. See anti-patterns.md.
```

## Key System Variables

| Variable | Default | Scope | Dynamic | Purpose |
|----------|---------|-------|---------|---------|
| `innodb_deadlock_detect` | `ON` | Global | Yes | Detects lock-cycles ; aborts smaller transaction with 1213 |
| `innodb_lock_wait_timeout` | `50` (seconds) | Global, Session | Yes | Max seconds a transaction waits for a row-lock before error 1205 |
| `innodb_print_all_deadlocks` | `OFF` | Global | Yes | When ON, every detected deadlock is written to the error log |
| `transaction_isolation` (10.6+) / `tx_isolation` (legacy) | `REPEATABLE-READ` | Global, Session | Yes | Default isolation level ; READ COMMITTED disables gap locks |

ALWAYS set `innodb_print_all_deadlocks=ON` in production. The cost is negligible and the forensic value is high.

## Lock-Type Mental Model

InnoDB lock-modes (confirmed per `innodb-lock-modes` KB) :

| Lock | Level | Purpose |
|------|-------|---------|
| Shared (S) | Row | Read-lock ; multiple S allowed on same row |
| Exclusive (X) | Row | Write-lock ; blocks all other locks |
| Intention Shared (IS) | Table | Signals intent to take S on some row in this table |
| Intention Exclusive (IX) | Table | Signals intent to take X on some row in this table |
| Record lock | Index entry | Lock on a specific index record |
| Gap lock | Between index entries | Prevents phantom inserts (REPEATABLE READ only) |
| Next-key lock | Index entry + gap before | Default REPEATABLE READ lock for range scans |
| Insert-intention lock | Gap | Special gap lock indicating an INSERT is about to happen |
| AUTO-INC lock | Table | Serializes assignment of AUTO_INCREMENT values |

Compatibility matrix (per the KB) :
- X blocks all other locks.
- S blocked by X or IX.
- IX blocked by X or S.
- IS blocked only by X.

REPEATABLE READ (the default) uses next-key locks on range scans, which is why an indexed `WHERE` range can lock more rows than the actual matches. READ COMMITTED disables gap locks (KB : "Gap locks are disabled if the isolation level is set to READ COMMITTED").

See `references/methods.md` for the complete lock-mode taxonomy and the `SHOW ENGINE INNODB STATUS` deadlock-section read-guide.

## Reading SHOW ENGINE INNODB STATUS

When a deadlock occurs, the engine records the most recent one in the `LATEST DETECTED DEADLOCK` block. Run :

```sql
SHOW ENGINE INNODB STATUS\G
```

The block shows TWO transactions, what each held, what each waited for, and which one was chosen as the victim (auto-rolled-back). To capture EVERY deadlock (not only the latest), enable `innodb_print_all_deadlocks=ON` and tail the MariaDB error log.

```sql
-- 10.6+
SET GLOBAL innodb_print_all_deadlocks = ON;
```

The block names a victim at the end :

```
*** WE ROLL BACK TRANSACTION (1)
```

Transaction (1) was killed. The other (transaction (2)) succeeded. The victim is typically the transaction that has modified the FEWEST rows so far (cheapest to roll back).

See `references/methods.md` for a labelled walkthrough of every line of the deadlock-block.

## Lock-Order Discipline

The #1 cause of avoidable deadlocks : two transactions acquire locks on the same set of tables in DIFFERENT order. Two transactions doing `UPDATE accounts ; UPDATE audit_log` and `UPDATE audit_log ; UPDATE accounts` will deadlock under any concurrent load.

ALWAYS pick a canonical order (alphabetical, foreign-key dependency order, whatever) and enforce it across the entire codebase. Code review should reject any transaction that locks tables in a different order than its peers.

```sql
-- 10.6+ : canonical order across ALL business transactions
START TRANSACTION;
SELECT id FROM accounts     WHERE id = ? FOR UPDATE;   -- always FIRST
INSERT INTO audit_log (account_id, action, ts) VALUES (?, ?, NOW());  -- always SECOND
UPDATE accounts SET balance = balance + ? WHERE id = ?;
COMMIT;
```

See `references/examples.md` for a reproducer of a 2-transaction deadlock and the fix.

## Application-Layer Retry Strategy

When a deadlock occurs, retry the ENTIRE transaction (not just the failed statement). Use exponential backoff with a maximum retry count. Log every retry.

```python
# Python 3.10+, mariadb 1.1+, MariaDB 10.6+
import mariadb, time, random, logging

MAX_RETRIES = 4

def run_with_deadlock_retry(conn, work_fn):
    for attempt in range(1, MAX_RETRIES + 1):
        try:
            cur = conn.cursor()
            cur.execute("START TRANSACTION")
            work_fn(cur)
            conn.commit()
            return
        except mariadb.OperationalError as e:
            errno = e.args[0] if e.args else None
            if errno == 1213:   # ER_LOCK_DEADLOCK
                conn.rollback()
                if attempt == MAX_RETRIES:
                    logging.error("Deadlock retries exhausted")
                    raise
                backoff = (2 ** attempt) * 0.05 + random.uniform(0, 0.05)
                logging.warning("Deadlock attempt %d, backoff %.3fs", attempt, backoff)
                time.sleep(backoff)
                continue
            raise
```

Retry budget is 3-5 attempts. If a transaction deadlocks more than that, the root cause is a DESIGN problem (hot-row, bad lock-order, missing index) and retries will just mask it.

See `references/examples.md` for PHP, Node.js, and Java retry implementations.

## Galera Certification-Deadlock

On a Galera cluster (`wsrep_on=ON`), write-write conflicts between nodes are detected at CERTIFICATION time, which is `COMMIT`. The cluster returns `ER_LOCK_DEADLOCK` (1213) on the local node even if the local transaction never touched a conflicting row on this node directly.

This is NORMAL behavior, NOT a bug. The application MUST handle it identically to a standalone-InnoDB deadlock : retry the transaction.

```sql
-- 10.6+ Galera node : check certification failure count
SHOW STATUS LIKE 'wsrep_local_cert_failures';
```

A non-zero `wsrep_local_cert_failures` is expected on a healthy multi-master cluster under write load. Alert ONLY on sudden multi-order-of-magnitude jumps, not on the absolute number. See `references/anti-patterns.md` for the "treating Galera commit-deadlock as a bug" anti-pattern.

## Isolation-Level Impact on Locks

| Level | Gap locks | Phantom reads | Deadlock incidence |
|-------|-----------|---------------|--------------------|
| READ UNCOMMITTED | No | Yes (and dirty reads) | Low (almost no locking) |
| READ COMMITTED | NO (disabled) | Yes | Lower than RR |
| REPEATABLE READ (default) | Yes (next-key) | No (until 11.4 limits) | Highest |
| SERIALIZABLE | Yes (broadest) | No | Highest plus contention |

Switching from REPEATABLE READ to READ COMMITTED reduces deadlocks but introduces phantom reads. This is a SEMANTIC change, not a tuning knob. Verify your application can tolerate phantom-reads before switching.

```sql
-- 10.6+ : per-session change for an OLTP service
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

NEVER switch the GLOBAL default without auditing every read path. NEVER use SERIALIZABLE on an OLTP workload (it converts plain `SELECT` into locking reads when autocommit is OFF).

## Hot-Row Contention

A single row contended by many writers will deadlock or lock-wait under load no matter how clean your lock-order discipline is. Symptoms : a single counter table, a single sequence row, an inventory row for a popular SKU, a single rate-limit bucket.

Redesign options :
- Shard the hot key across N sub-keys, sum at read time.
- Use a sequence object (`CREATE SEQUENCE`, 10.3+) instead of `UPDATE counter SET v = v + 1`.
- Use `INSERT ... ON DUPLICATE KEY UPDATE` with a sharded primary-key prefix.
- Use a queue table with rows-per-message instead of an UPDATE-in-place.

See `references/examples.md` for a hot-row redesign and `references/anti-patterns.md` for the single-counter-table anti-pattern.

## What This Skill Does NOT Cover

- General InnoDB tuning : see `mariadb-impl-performance-tuning`.
- Galera cluster setup : see `mariadb-impl-galera-cluster`.
- Replication conflicts (non-Galera) : see `mariadb-core-replication-model`.
- Optimistic-concurrency patterns (versioned rows) : application-layer concern, out of scope.

## Reference Links

- `references/methods.md` : lock-types matrix, SHOW ENGINE INNODB STATUS deadlock-section read-guide, system-variable details, isolation-level impact on locks.
- `references/examples.md` : 10+ working examples (reproduce a deadlock, read INNODB STATUS, retry loops in Python/PHP/Node.js/Java, switch to READ COMMITTED, hot-row redesign, Galera retry pattern).
- `references/anti-patterns.md` : 8+ real anti-patterns (single-statement retry, disabling deadlock detection, READ UNCOMMITTED escape, inconsistent lock order, Galera-bug treatment, hot-row design, long-running transaction, no deadlock logging).

## Source Verification

All facts in this skill were verified via WebFetch against :
- `mariadb.com/docs/server/server-usage/storage-engines/innodb/innodb-system-variables` : `innodb_deadlock_detect` default ON / global / dynamic ; fall-back to `innodb_lock_wait_timeout`.
- `mariadb.com/docs/server/server-usage/storage-engines/innodb/innodb-lock-modes` : S / X / IS / IX lock-modes, compatibility matrix, gap locks disabled at READ COMMITTED.
- `mariadb.com/kb/en/innodb-system-variables/` : `innodb_lock_wait_timeout` default 50 / range 1-1073741824 / global+session / dynamic ; `innodb_print_all_deadlocks` default OFF / global ; error 1205 `ER_LOCK_WAIT_TIMEOUT_EXCEEDED`.
- `mariadb.com/docs/server/reference/sql-statements/administrative-sql-statements/set-commands/set-transaction` : four isolation levels, default REPEATABLE READ, SET TRANSACTION syntax with GLOBAL/SESSION/no-scope semantics.
- Vooronderzoek Cluster-3 §2 : Galera certification-conflict at COMMIT semantics, `wsrep_local_cert_failures` status variable.
