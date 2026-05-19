# Examples : Triggers, Events, Views

Ten working examples. Every snippet verified against MariaDB 10.6-LTS, 10.11-LTS, 11.x, and 12.x unless annotated otherwise. Version annotations use SQL comments (`-- 10.6+`).

---

## Example 1 : audit-log trigger on UPDATE

The canonical valid use case for a trigger : write a history row whenever a sensitive column changes.

```sql
-- 10.6+
CREATE OR REPLACE TABLE accounts (
  id        BIGINT        PRIMARY KEY,
  owner     VARCHAR(64)   NOT NULL,
  balance   DECIMAL(20,2) NOT NULL,
  status    VARCHAR(20)   NOT NULL
);

CREATE OR REPLACE TABLE balance_audit (
  audit_id   BIGINT PRIMARY KEY AUTO_INCREMENT,
  account_id BIGINT       NOT NULL,
  old_bal    DECIMAL(20,2),
  new_bal    DECIMAL(20,2),
  changed_by VARCHAR(128),
  changed_at TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6)
);

CREATE OR REPLACE TRIGGER trg_audit_balance
  AFTER UPDATE ON accounts
  FOR EACH ROW
INSERT INTO balance_audit (account_id, old_bal, new_bal, changed_by)
VALUES (NEW.id, OLD.balance, NEW.balance, CURRENT_USER());
```

Verify :

```sql
UPDATE accounts SET balance = 100 WHERE id = 1;
SELECT * FROM balance_audit;
-- one row : old_bal = previous value, new_bal = 100, changed_by = '<user>@<host>'.
```

---

## Example 2 : BEFORE INSERT trigger with SIGNAL validation

Reject inserts that violate a business rule. Use this pattern when the rule cannot be expressed as a CHECK constraint (for example, it depends on another table).

```sql
-- 10.6+
CREATE OR REPLACE TABLE blocked_owners (
  owner VARCHAR(64) PRIMARY KEY
);

CREATE OR REPLACE TRIGGER trg_reject_blocked_owner
  BEFORE INSERT ON accounts
  FOR EACH ROW
BEGIN
  IF EXISTS (SELECT 1 FROM blocked_owners WHERE owner = NEW.owner) THEN
    SIGNAL SQLSTATE '45000'
      SET MESSAGE_TEXT = 'owner is blocked from creating accounts';
  END IF;
END;
```

`SIGNAL SQLSTATE '45000'` is the user-defined-exception class. The INSERT fails with the supplied MESSAGE_TEXT visible to the application.

---

## Example 3 : multi-trigger ordering with FOLLOWS

Since 10.2.3+ MariaDB supports multiple triggers per (table, timing, event). Order is explicit via FOLLOWS or PRECEDES.

```sql
-- 10.2.3+
CREATE OR REPLACE TABLE login_log (
  log_id     BIGINT PRIMARY KEY AUTO_INCREMENT,
  account_id BIGINT      NOT NULL,
  at         TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6)
);

CREATE OR REPLACE TRIGGER trg_audit_login
  AFTER UPDATE ON accounts
  FOR EACH ROW
INSERT INTO login_log (account_id) VALUES (NEW.id);

CREATE OR REPLACE TRIGGER trg_audit_balance
  AFTER UPDATE ON accounts
  FOR EACH ROW
  FOLLOWS trg_audit_login
INSERT INTO balance_audit (account_id, old_bal, new_bal)
VALUES (NEW.id, OLD.balance, NEW.balance);

-- Inspect order :
SELECT TRIGGER_NAME, ACTION_ORDER
  FROM INFORMATION_SCHEMA.TRIGGERS
 WHERE EVENT_OBJECT_TABLE = 'accounts'
   AND ACTION_TIMING      = 'AFTER'
   AND EVENT_MANIPULATION = 'UPDATE'
 ORDER BY ACTION_ORDER;
-- trg_audit_login    1
-- trg_audit_balance  2
```

---

## Example 4 : daily-cleanup event

Recurring event that purges old audit rows nightly.

```sql
-- 10.6+
-- Step 1 : enable scheduler (and persist in [mysqld] of my.cnf)
SET GLOBAL event_scheduler = ON;

-- Step 2 : event
CREATE OR REPLACE EVENT ev_purge_old_audits
  ON SCHEDULE EVERY 1 DAY
    STARTS CURRENT_TIMESTAMP + INTERVAL 1 HOUR
  ON COMPLETION PRESERVE
  ENABLE
  COMMENT 'nightly audit retention (90 day window)'
DO
  DELETE FROM balance_audit
   WHERE changed_at < NOW() - INTERVAL 90 DAY;

-- Verify
SELECT EVENT_NAME, STATUS, LAST_EXECUTED
  FROM INFORMATION_SCHEMA.EVENTS
 WHERE EVENT_NAME = 'ev_purge_old_audits';
```

---

## Example 5 : hourly refresh event with DISABLE ON SLAVE

The standard recurring-refresh pattern for the materialized-view workaround.

```sql
-- 10.6+
CREATE OR REPLACE EVENT ev_refresh_mv_daily_revenue
  ON SCHEDULE EVERY 1 HOUR
  ON COMPLETION PRESERVE
  DISABLE ON SLAVE
DO
  BEGIN
    TRUNCATE TABLE mv_daily_revenue;
    INSERT INTO mv_daily_revenue
      SELECT DATE(order_at) AS d,
             SUM(total)     AS revenue,
             COUNT(*)       AS orders
        FROM orders
       GROUP BY DATE(order_at);
  END;
```

`DISABLE ON SLAVE` ensures the replica does NOT run the event ; the binlog of the primary already replicates the TRUNCATE and INSERT statements.

---

## Example 6 : view with MERGE algorithm (the fast default)

A simple updatable view that filters to active accounts.

```sql
-- 10.6+
CREATE OR REPLACE
  ALGORITHM = MERGE
  DEFINER   = 'reporting_owner'@'localhost'
  SQL SECURITY DEFINER
VIEW v_active_accounts AS
  SELECT id, owner, balance
    FROM accounts
   WHERE status = 'active'
  WITH CHECK OPTION;

-- Confirm MERGE chosen :
SELECT TABLE_NAME, ALGORITHM, IS_UPDATABLE
  FROM INFORMATION_SCHEMA.VIEWS
 WHERE TABLE_NAME = 'v_active_accounts';
-- v_active_accounts | MERGE | YES
```

EXPLAIN against `SELECT * FROM v_active_accounts WHERE id = 1` shows `type = const` (or `eq_ref`) and `key = PRIMARY`, identical to selecting from `accounts` directly.

---

## Example 7 : view with TEMPTABLE (and why to avoid it)

Demonstration only. NEVER use TEMPTABLE on a large base table in production.

```sql
-- 10.6+ : DO NOT USE FOR LARGE TABLES
CREATE OR REPLACE
  ALGORITHM = TEMPTABLE
  SQL SECURITY DEFINER
VIEW v_active_accounts_slow AS
  SELECT id, owner, balance
    FROM accounts
   WHERE status = 'active';

-- Compare EXPLAIN to Example 6 :
EXPLAIN SELECT * FROM v_active_accounts_slow WHERE id = 1;
-- select_type = DERIVED on the view, full materialization of all 'active' rows.
-- type = ALL on the inner SELECT.
-- Even though the outer query filters by id, the optimizer cannot push id = 1
-- into the materialized result. Every reference re-materializes the entire view.
```

The fix is `ALTER VIEW v_active_accounts_slow ALGORITHM = UNDEFINED ;`.

---

## Example 8 : view with WITH CHECK OPTION CASCADED

Demonstrate how CASCADED protects against bypassing the predicate through a chained view.

```sql
-- 10.6+
CREATE OR REPLACE VIEW v_premium_accounts AS
  SELECT id, owner, balance
    FROM accounts
   WHERE balance > 10000
  WITH CASCADED CHECK OPTION;

CREATE OR REPLACE VIEW v_premium_active AS
  SELECT id, owner, balance
    FROM v_premium_accounts
   WHERE id < 1000
  WITH CASCADED CHECK OPTION;

-- This succeeds : id = 500 and balance = 15000 satisfies both predicates.
INSERT INTO v_premium_active (id, owner, balance) VALUES (500, 'alice', 15000);

-- This fails : balance = 50 violates the v_premium_accounts predicate.
-- CASCADED walks the chain and rejects.
INSERT INTO v_premium_active (id, owner, balance) VALUES (501, 'bob', 50);
-- ERROR : CHECK OPTION failed
```

With `LOCAL CHECK OPTION` on the inner view only, the second INSERT would succeed (predicate of the outer view is not checked).

---

## Example 9 : materialized-view workaround (complete recipe)

Real-world pattern. The base query `SUM(total) GROUP BY DATE(order_at)` is too expensive for every dashboard read.

```sql
-- 10.6+
-- Step 1 : create the orders table (presumed existing in real systems)
CREATE OR REPLACE TABLE orders (
  id        BIGINT PRIMARY KEY AUTO_INCREMENT,
  order_at  TIMESTAMP(6) NOT NULL,
  total     DECIMAL(20,2) NOT NULL
);

-- Step 2 : create the snapshot table from the expensive aggregate
CREATE OR REPLACE TABLE mv_daily_revenue AS
  SELECT DATE(order_at) AS d,
         SUM(total)     AS revenue,
         COUNT(*)       AS orders
    FROM orders
   GROUP BY DATE(order_at);

-- Step 3 : index the snapshot for the typical dashboard query
CREATE INDEX ix_mv_daily_revenue_d ON mv_daily_revenue (d);

-- Step 4 : scheduled refresh (every hour)
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

-- Step 5 : application queries hit the snapshot, not orders
SELECT d, revenue
  FROM mv_daily_revenue
 WHERE d BETWEEN '2026-05-01' AND '2026-05-19'
 ORDER BY d;
```

For zero-downtime refresh, swap the snapshot via RENAME TABLE :

```sql
-- 10.6+
-- Build new snapshot, then atomic swap
CREATE OR REPLACE TABLE mv_daily_revenue_new AS
  SELECT DATE(order_at) AS d, SUM(total) AS revenue, COUNT(*) AS orders
    FROM orders
   GROUP BY DATE(order_at);

RENAME TABLE mv_daily_revenue     TO mv_daily_revenue_old,
             mv_daily_revenue_new TO mv_daily_revenue,
             mv_daily_revenue_old TO mv_daily_revenue_new;

TRUNCATE TABLE mv_daily_revenue_new;
```

---

## Example 10 : role-scoped DEFINER pattern

Expose a sensitive table to a reporting role without granting direct access to the base table.

```sql
-- 10.6+
-- Step 1 : service account that owns the view
CREATE USER 'reporting_owner'@'localhost' IDENTIFIED BY 'long-random';
GRANT SELECT ON salaries.* TO 'reporting_owner'@'localhost';

-- Step 2 : role for downstream consumers
CREATE ROLE reporting_reader;

-- Step 3 : view runs as reporting_owner, but only the bounded subset
CREATE OR REPLACE
  ALGORITHM = MERGE
  DEFINER   = 'reporting_owner'@'localhost'
  SQL SECURITY DEFINER
VIEW v_public_salaries AS
  SELECT department, ROUND(AVG(salary), 0) AS avg_salary, COUNT(*) AS headcount
    FROM salaries.employee_salary
   GROUP BY department;
-- Note : the GROUP BY makes this view non-updatable, which is intentional.

-- Step 4 : grant the role SELECT on the view, NOT on the base table
GRANT SELECT ON salaries.v_public_salaries TO reporting_reader;

-- Step 5 : assign role to actual users
GRANT reporting_reader TO 'analyst1'@'%';
SET DEFAULT ROLE reporting_reader FOR 'analyst1'@'%';
```

`analyst1` can `SELECT * FROM salaries.v_public_salaries` even though it has NO direct privilege on `salaries.employee_salary`. The view runs as `reporting_owner` which DOES have the base-table privilege.
