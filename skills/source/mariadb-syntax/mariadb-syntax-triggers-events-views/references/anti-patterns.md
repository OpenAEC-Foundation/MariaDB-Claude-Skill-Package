# Anti-Patterns : Triggers, Events, Views

Eight real anti-patterns with root cause and the correct fix. Verified 2026-05-19 against MariaDB Knowledge Base and reproduced on MariaDB 10.6, 10.11, 11.x, 12.x.

---

## AP-1 : Trigger doing N+1 lookups inside FOR EACH ROW

### The pattern (wrong)

```sql
-- 10.6+ : DO NOT DO THIS
CREATE OR REPLACE TRIGGER trg_enrich_audit
  AFTER INSERT ON balance_audit
  FOR EACH ROW
BEGIN
  DECLARE v_dept VARCHAR(64);
  -- N+1 : a SELECT against a large lookup table for every inserted row
  SELECT department INTO v_dept
    FROM employees
   WHERE id = NEW.account_id;
  UPDATE balance_audit
     SET department = v_dept
   WHERE audit_id = NEW.audit_id;
END;
```

### Why it fails

A bulk `INSERT INTO balance_audit SELECT ... FROM ...` that inserts 10 000 rows fires this trigger 10 000 times. Each fire executes one SELECT against `employees` (a point lookup that costs at minimum one index dive) and one UPDATE (a row update that costs another index dive plus log write). The 10 000-row INSERT becomes 30 000 logical operations. On a table with 10 million rows in `employees`, the effective slowdown is 50-200x.

A second-order failure : the UPDATE inside the AFTER INSERT trigger holds row locks until the outer statement commits. Concurrent bulk inserts deadlock.

### The fix

Move the lookup out of the per-row trigger into a set-based operation in the application or a stored procedure :

```sql
-- 10.6+ : correct
-- The application inserts plain audit rows :
INSERT INTO balance_audit (account_id, old_bal, new_bal)
SELECT id, prev_balance, balance FROM accounts_snapshot;

-- Then a single set-based UPDATE enriches them :
UPDATE balance_audit a
  JOIN employees e ON e.id = a.account_id
   SET a.department = e.department
 WHERE a.department IS NULL;
```

Alternatively, denormalize at write-time : if the application already knows the department, write it directly in the original INSERT.

---

## AP-2 : event_scheduler stays OFF, event never runs

### The pattern (wrong)

```sql
-- 10.6+ : created on a fresh server, scheduler is OFF
CREATE EVENT ev_purge_old_audits
  ON SCHEDULE EVERY 1 DAY
DO
  DELETE FROM balance_audit WHERE changed_at < NOW() - INTERVAL 90 DAY;

-- Wait 24 hours, run the query, audit rows are still there.
```

### Why it fails

The `event_scheduler` global variable defaults to `OFF` on most distributions (Debian / Ubuntu / RHEL stock packages). The CREATE statement succeeds, the event is registered in `mysql.event`, but the scheduler thread that fires events is not running. `INFORMATION_SCHEMA.EVENTS.LAST_EXECUTED` stays `NULL` forever.

A second variant : the admin ran `SET GLOBAL event_scheduler = ON` once at runtime, the event fired for a while, then the server was restarted and the setting reverted to OFF.

### The fix

Two steps : turn it on now, and persist for restart.

```sql
-- Runtime activation :
SET GLOBAL event_scheduler = ON;

-- Verify :
SHOW VARIABLES LIKE 'event_scheduler';
-- event_scheduler ON
```

And in `my.cnf` :

```ini
[mysqld]
event_scheduler = ON
```

If the value reads `DISABLED` instead of `OFF`, the server was started with `--event-scheduler=DISABLED`. `SET GLOBAL` cannot switch out of DISABLED ; a restart with the config change above is required.

---

## AP-3 : View with ALGORITHM = TEMPTABLE on a huge base table

### The pattern (wrong)

```sql
-- 10.6+ : DO NOT USE TEMPTABLE FOR LARGE TABLES
CREATE OR REPLACE
  ALGORITHM = TEMPTABLE
VIEW v_recent_orders AS
  SELECT id, customer_id, total, order_at
    FROM orders
   WHERE order_at > NOW() - INTERVAL 7 DAY;

-- Application queries :
SELECT * FROM v_recent_orders WHERE customer_id = 12345;
```

### Why it fails

`TEMPTABLE` forces materialization of the entire view body into a per-query temp table on EVERY reference. The outer `WHERE customer_id = 12345` cannot be pushed down into the view body. Even if `orders` has a perfect index on `(customer_id, order_at)`, the engine scans the last 7 days of orders, materializes them into a temp table, then filters that temp table by `customer_id`. On a table with 100k orders per day, that is 700k rows scanned per query.

The view is also non-updatable because TEMPTABLE breaks updatability.

### The fix

Default to `UNDEFINED` and let the optimizer pick MERGE when possible :

```sql
-- 10.6+ : correct
ALTER VIEW v_recent_orders
  ALGORITHM = UNDEFINED
  AS
  SELECT id, customer_id, total, order_at
    FROM orders
   WHERE order_at > NOW() - INTERVAL 7 DAY;
```

EXPLAIN now shows the outer `WHERE customer_id = 12345` pushed through, hitting the `(customer_id, order_at)` index directly. Only use TEMPTABLE when the view body has a non-mergeable construct AND you specifically want the result frozen for the duration of the query (which is rare).

---

## AP-4 : Expecting CREATE MATERIALIZED VIEW to work

### The pattern (wrong)

```sql
-- This syntax DOES NOT EXIST in MariaDB
CREATE MATERIALIZED VIEW mv_daily_revenue AS
  SELECT DATE(order_at) AS d, SUM(total) AS revenue
    FROM orders
   GROUP BY DATE(order_at);
-- ERROR 1064 (42000) : You have an error in your SQL syntax near 'MATERIALIZED'
```

### Why it fails

MariaDB has NO materialized-view object. The keyword does not parse. Users coming from PostgreSQL, Oracle, or any DBMS with native materialized views will type this and hit a syntax error.

Closely related confusion : `EXPLAIN` output contains `select_type = MATERIALIZED` for derived-table materialization. This refers to query-time materialization of subqueries, NOT to materialized-view objects.

### The fix

Use the documented workaround : a real table seeded by the expensive query, plus an event-scheduled refresh.

```sql
-- 10.6+ : correct
CREATE OR REPLACE TABLE mv_daily_revenue AS
  SELECT DATE(order_at) AS d, SUM(total) AS revenue, COUNT(*) AS orders
    FROM orders
   GROUP BY DATE(order_at);

CREATE INDEX ix_mv_daily_revenue_d ON mv_daily_revenue (d);

CREATE OR REPLACE EVENT ev_refresh_mv_daily_revenue
  ON SCHEDULE EVERY 1 HOUR
  ON COMPLETION PRESERVE
  DISABLE ON SLAVE
DO
  BEGIN
    TRUNCATE TABLE mv_daily_revenue;
    INSERT INTO mv_daily_revenue
      SELECT DATE(order_at), SUM(total), COUNT(*)
        FROM orders
       GROUP BY DATE(order_at);
  END;
```

Trade-offs the user must accept : eventual consistency (lag equal to refresh interval), full-refresh cost per cycle, and brief unavailability during TRUNCATE (mitigated by RENAME TABLE swap).

---

## AP-5 : View / trigger / event whose DEFINER account was dropped

### The pattern (wrong)

```sql
-- A previous DBA created a view :
CREATE
  DEFINER = 'old_dba'@'localhost'
  SQL SECURITY DEFINER
VIEW v_sensitive AS
  SELECT * FROM hr.salaries;

-- Months later, the account is cleaned up :
DROP USER 'old_dba'@'localhost';

-- Now SELECT fails for everyone :
SELECT * FROM v_sensitive;
-- ERROR 1449 (HY000) : The user specified as a definer ('old_dba'@'localhost') does not exist
```

### Why it fails

`SQL SECURITY DEFINER` (the default) means the view runs with the DEFINER's privileges. When the DEFINER account is dropped, MariaDB cannot resolve those privileges and refuses to run the view. The same applies to triggers, events, and stored routines.

### The fix

Re-point the object at a stable service account that exists and has the required base-table privileges :

```sql
-- 10.6+ : correct
CREATE USER 'service_views'@'localhost' IDENTIFIED BY 'long-random';
GRANT SELECT ON hr.* TO 'service_views'@'localhost';

ALTER
  DEFINER = 'service_views'@'localhost'
  SQL SECURITY DEFINER
VIEW v_sensitive AS
  SELECT * FROM hr.salaries;
```

Prevention : create a dedicated service account for every DEFINER and document it. NEVER use a personal DBA account as the DEFINER for long-lived objects.

To audit which objects depend on which DEFINER :

```sql
SELECT 'VIEW' AS kind, TABLE_SCHEMA, TABLE_NAME, DEFINER
  FROM INFORMATION_SCHEMA.VIEWS
UNION ALL
SELECT 'TRIGGER', TRIGGER_SCHEMA, TRIGGER_NAME, DEFINER
  FROM INFORMATION_SCHEMA.TRIGGERS
UNION ALL
SELECT 'EVENT', EVENT_SCHEMA, EVENT_NAME, DEFINER
  FROM INFORMATION_SCHEMA.EVENTS
UNION ALL
SELECT 'ROUTINE', ROUTINE_SCHEMA, ROUTINE_NAME, DEFINER
  FROM INFORMATION_SCHEMA.ROUTINES;
```

---

## AP-6 : Non-deterministic trigger with binlog_format = STATEMENT or MIXED

### The pattern (wrong)

```sql
-- 10.6+ on a replication primary with binlog_format = STATEMENT
CREATE OR REPLACE TRIGGER trg_set_random_id
  BEFORE INSERT ON jobs
  FOR EACH ROW
SET NEW.job_uuid = UUID();
```

### Why it fails

With `binlog_format = STATEMENT` (or `MIXED` falling back to STATEMENT for this case), the trigger body is re-executed on the replica. `UUID()` is non-deterministic : the replica generates a DIFFERENT UUID than the primary. The replica's data drifts from the primary's, eventually breaking foreign-key references and JOIN semantics.

Same applies to `NOW(6)` with sub-second precision, `RAND()`, `CONNECTION_ID()`, `CURRENT_USER()`, and any function whose value depends on the executing server.

With `binlog_format = ROW`, the trigger fires only on the primary and the resulting row is replicated verbatim. The replica does NOT re-fire the trigger. ROW is safe for non-deterministic triggers.

### The fix

Three options, in preference order :

1. **Set `binlog_format = ROW`** server-wide. This is the modern default and the recommended setting for any topology with triggers.

   ```ini
   [mysqld]
   binlog_format = ROW
   ```

2. **Make the trigger deterministic** by sourcing the value from the application (pass UUID as a column value in the INSERT) instead of generating it inside the trigger.

3. **Move the logic out of the trigger** into the application or into a stored procedure called explicitly, where the non-determinism is intentional.

Verify the format with `SHOW VARIABLES LIKE 'binlog_format'` ; verify per-session with `SHOW SESSION VARIABLES LIKE 'binlog_format'`.

---

## AP-7 : INSERT IGNORE bypassing WITH CHECK OPTION

### The pattern (wrong)

```sql
-- 10.6+
CREATE OR REPLACE VIEW v_premium_accounts AS
  SELECT id, owner, balance
    FROM accounts
   WHERE balance > 10000
  WITH CHECK OPTION;

-- This silently succeeds and INSERTs a row that the view itself cannot see :
INSERT IGNORE INTO v_premium_accounts (id, owner, balance)
VALUES (999, 'eve', 50);

-- Now :
SELECT * FROM v_premium_accounts WHERE id = 999;
-- empty set : the row is in accounts, but invisible to the view.
SELECT * FROM accounts WHERE id = 999;
-- one row : id = 999, balance = 50 (which should have been rejected).
```

### Why it fails

`WITH CHECK OPTION` raises a CHECK-violation error when an INSERT or UPDATE through the view would produce a row outside the view's predicate. `INSERT IGNORE` downgrades the error to a warning and DOES NOT roll back. The row lands in the base table, satisfying neither the view's predicate nor the developer's intent.

Same anti-pattern applies to `UPDATE ... IGNORE` against a view with WITH CHECK OPTION.

### The fix

Two options :

1. **Forbid `INSERT IGNORE` against views with WITH CHECK OPTION** in your code-review process. Treat it as a build-blocking lint.

2. **Add a CHECK constraint on the base table** that mirrors the view's predicate, so the constraint applies regardless of the access path :

   ```sql
   -- 10.6+ : enforce at the base table
   ALTER TABLE accounts
     ADD CONSTRAINT chk_premium_only CHECK (balance > 10000);
   ```

   This rejects the row at the base-table level. `INSERT IGNORE` then still downgrades to a warning, but the row is still rejected because base-table CHECK constraints are not bypassable by IGNORE in the same way (the row simply does not pass the check and is skipped, NOT silently accepted).

Verify via :

```sql
SELECT id FROM accounts WHERE id = 999;
-- empty : the constraint blocked the insert.
```

---

## AP-8 : Transaction control statements inside a trigger body

### The pattern (wrong)

```sql
-- 10.6+ : creation succeeds, runtime fails
CREATE OR REPLACE TRIGGER trg_rollback_on_negative
  BEFORE INSERT ON accounts
  FOR EACH ROW
BEGIN
  IF NEW.balance < 0 THEN
    ROLLBACK;  -- INVALID
  END IF;
END;

INSERT INTO accounts (id, owner, balance, status) VALUES (1, 'bob', -5, 'active');
-- ERROR 1422 (HY000) : Explicit or implicit commit is not allowed in stored function or trigger
```

### Why it fails

Triggers run inside the calling statement's transaction and cannot manage it. The forbidden statements are `COMMIT`, `ROLLBACK`, `SAVEPOINT`, `RELEASE SAVEPOINT`, and `START TRANSACTION`. The CREATE statement parses successfully because the parser does not enforce the limitation, but the runtime executor raises `Explicit or implicit commit is not allowed in stored function or trigger` (error 1422 / SQLSTATE HY000) when the trigger fires.

A second variant : DDL inside a trigger body (`CREATE TABLE`, `DROP TABLE`, `ALTER TABLE`) implicitly commits and so triggers the same error.

### The fix

Use `SIGNAL` to raise a user-defined exception. The calling statement (and its transaction) is rolled back by the application's standard error-handling :

```sql
-- 10.6+ : correct
CREATE OR REPLACE TRIGGER trg_reject_negative
  BEFORE INSERT ON accounts
  FOR EACH ROW
BEGIN
  IF NEW.balance < 0 THEN
    SIGNAL SQLSTATE '45000'
      SET MESSAGE_TEXT = 'balance must be non-negative';
  END IF;
END;

-- The INSERT now fails cleanly :
INSERT INTO accounts (id, owner, balance, status) VALUES (1, 'bob', -5, 'active');
-- ERROR 1644 (45000) : balance must be non-negative
```

The application catches the error and decides whether to retry, log, or rollback its larger transaction. The trigger itself never touches transaction control. The same pattern applies when business logic requires aborting an UPDATE or DELETE : raise SIGNAL in BEFORE UPDATE / BEFORE DELETE.
