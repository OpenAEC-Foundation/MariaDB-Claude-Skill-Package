---
name: mariadb-syntax-triggers-events-views
description: >
  Use when defining triggers for BEFORE / AFTER INSERT / UPDATE / DELETE, scheduling recurring tasks via events, creating views including the materialized-view workaround, or controlling DEFINER vs INVOKER security context.
  Prevents the common mistake of overusing triggers for business logic, forgetting event_scheduler is OFF by default, expecting MariaDB to support materialized views (it does not), or choosing TEMPTABLE algorithm on huge views.
  Covers CREATE TRIGGER with FOLLOWS / PRECEDES, multi-trigger per event (10.2.3+), CREATE EVENT scheduler, ON COMPLETION PRESERVE / NOT PRESERVE, CREATE VIEW with ALGORITHM (UNDEFINED / MERGE / TEMPTABLE), DEFINER vs INVOKER, WITH CHECK OPTION LOCAL / CASCADED, materialized-view workaround via CREATE TABLE AS SELECT + EVENT.
  Keywords: CREATE TRIGGER, BEFORE INSERT, AFTER UPDATE, BEFORE DELETE, AFTER DELETE, FOR EACH ROW, FOLLOWS, PRECEDES, NEW.col, OLD.col, NEW pseudo-row, OLD pseudo-row, CREATE EVENT, ON SCHEDULE EVERY, ON SCHEDULE AT, event_scheduler, SET GLOBAL event_scheduler, ON COMPLETION PRESERVE, ON COMPLETION NOT PRESERVE, DISABLE ON SLAVE, CREATE VIEW, CREATE OR REPLACE VIEW, ALGORITHM, MERGE, TEMPTABLE, UNDEFINED, DEFINER, SQL SECURITY DEFINER, SQL SECURITY INVOKER, WITH CHECK OPTION, LOCAL, CASCADED, materialized view, CREATE TABLE AS SELECT, INFORMATION_SCHEMA.TRIGGERS, INFORMATION_SCHEMA.EVENTS, SHOW TRIGGERS, SHOW CREATE VIEW, trigger limitations, why is my trigger not firing, why is my event not running, my view is slow, my scheduled job never runs, how to refresh a materialized view in MariaDB, MariaDB materialized view alternative, audit trigger, history trigger
license: MIT
compatibility: "Designed for Claude Code. Requires MariaDB 10.6-LTS, 10.11-LTS, 11.x, 12.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# MariaDB Triggers, Events, and Views

Deterministic guidance for row-level triggers, the event scheduler, and views in MariaDB. Use this skill when wiring side-effects to INSERT / UPDATE / DELETE, scheduling recurring SQL jobs, building updatable views with security boundaries, or simulating materialized views via the table-plus-event refresh pattern.

## Quick Reference

- **ALWAYS turn on the event scheduler before relying on events**. The global `event_scheduler` variable defaults to `OFF` on most distros. A `CREATE EVENT` statement succeeds, but the event never fires until `SET GLOBAL event_scheduler = ON` is executed AND persisted in `[mysqld]` of `my.cnf` so it survives restart. Verify with `SHOW VARIABLES LIKE 'event_scheduler'`. Verified : KB `events/`.
- **ALWAYS use FOR EACH ROW**. MariaDB has NO statement-level triggers. Every trigger fires once per affected row, so a 10 000-row bulk DML fires the trigger 10 000 times. Keep trigger bodies short ; move heavy logic to stored procedures invoked once by the application. Verified : KB `create-trigger/`.
- **NEVER expect materialized views in MariaDB**. The keyword `MATERIALIZED VIEW` does not exist. The supported alternative is `CREATE TABLE snapshot AS SELECT ... ;` plus an event-scheduled refresh that truncates and reloads the snapshot. The optimizer column `select_type = MATERIALIZED` refers to derived-table materialization, NOT to materialized-view objects. Verified : KB `create-view/`.
- **NEVER use COMMIT, ROLLBACK, SAVEPOINT, or START TRANSACTION inside a trigger**. The statement parses at CREATE time but raises `Explicit or implicit commit is not allowed in stored function or trigger` at runtime. Triggers run inside the calling statement's transaction and cannot manage it. Verified : KB `trigger-limitations/`.
- **ALWAYS choose the right view ALGORITHM**. `UNDEFINED` (default) lets the optimizer pick `MERGE` when possible. `MERGE` inlines the view body into the outer query, preserves index pushdown, and keeps the view updatable. `TEMPTABLE` materializes the entire view result into a per-query temp table, breaks index pushdown, and forces a full scan on every reference. NEVER use `TEMPTABLE` on a multi-million-row base table unless the view body contains a non-mergeable construct (DISTINCT, GROUP BY, aggregate, UNION) that the optimizer cannot push down anyway. Verified : KB `create-view/`.
- **ALWAYS set DEFINER explicitly on security-sensitive objects**. Views, triggers, events, and stored routines all carry a `DEFINER` clause. If the DEFINER account is dropped, the object becomes invalid and access fails with `The user specified as a definer ... does not exist`. For shared / multi-DBA environments, set `DEFINER = 'role_owner'@'localhost'` to a stable service account. Verified : KB `create-view/`, KB `create-trigger/`.
- **NEVER use DEFINER + SQL SECURITY DEFINER for code that must respect caller permissions**. `SQL SECURITY DEFINER` (default) means the view / trigger / routine runs with the definer's privileges, bypassing the caller's GRANTs. Use `SQL SECURITY INVOKER` when the caller's GRANT set must be enforced. Verified : KB `create-view/`.
- **Multi-trigger per event requires explicit ordering since 10.2.3+**. Use `FOLLOWS other_trigger` or `PRECEDES other_trigger` in the `CREATE TRIGGER` clause. Order is observable in `INFORMATION_SCHEMA.TRIGGERS.ACTION_ORDER`. Without explicit ordering, the engine assigns the next free `ACTION_ORDER` value, which is fragile across migrations. Verified : KB `create-trigger/`.
- **DISABLE ON SLAVE prevents double-execution of events under replication**. Without it, the same event fires on both primary and replica with statement-based or mixed binlog. `DISABLE ON SLAVE` is the standard safety for any event that already replicates its effects through binlog. Verified : KB `events/`.
- **WITH CHECK OPTION rejects INSERT / UPDATE that produces a row outside the view's WHERE predicate**. `CASCADED` (default) walks every view in the chain ; `LOCAL` checks only the immediate view. Without `WITH CHECK OPTION`, a row can be inserted through the view that the view itself cannot see. Verified : KB `create-view/`.
- **Pseudo-rows are scoped per timing**. `NEW.col` is readable in BEFORE / AFTER INSERT and UPDATE triggers ; it is writable only in BEFORE triggers via `SET NEW.col = expr`. `OLD.col` is read-only in UPDATE and DELETE triggers. NEW does not exist in DELETE triggers ; OLD does not exist in INSERT triggers. Verified : KB `trigger-overview/`.

## Decision Trees

### Tree 1 : "Should this be a trigger, a CHECK constraint, an event, or application code?"

```
What is the requirement?
  Validate a single-row invariant (range, regex, conditional non-null) on write ?
    -> CHECK constraint (since 10.2, see mariadb-syntax-check-constraints).
       Triggers are wrong here : CHECK is faster, declarative, and engine-enforced.
  Maintain a derived column from other columns in the same row ?
    -> Generated column (VIRTUAL or PERSISTENT).
       Triggers are wrong here : generated columns are declarative and indexable.
  Write an audit / history row to a DIFFERENT table on every change ?
    -> AFTER INSERT / UPDATE / DELETE trigger.
       This is the canonical valid use case for a trigger.
  Block a write that depends on a multi-row condition ?
    -> BEFORE INSERT / UPDATE trigger that raises SIGNAL SQLSTATE '45000'.
       Note : multi-row conditions create lock contention. Prefer application-level
       validation with proper isolation.
  Run a job on a schedule ?
    -> CREATE EVENT (one-shot AT timestamp, or recurring EVERY interval).
       Application cron is acceptable too ; choose EVENT when the job is pure SQL
       and you want it co-located with the schema.
  Snapshot a slow aggregate query into a fast lookup table ?
    -> Materialized-view workaround : CREATE TABLE snapshot AS SELECT ...
       plus CREATE EVENT to refresh on schedule. See examples.md recipe.
```

### Tree 2 : "Why is my event not running?"

```
Created the event, see nothing in the target table ?
  Step 1 : SHOW VARIABLES LIKE 'event_scheduler' ;
    Value = OFF -> the scheduler thread is not running.
                   Fix : SET GLOBAL event_scheduler = ON ;
                         Persist in [mysqld] section : event_scheduler = ON
    Value = DISABLED -> server was started with --event-scheduler=DISABLED.
                         Fix : restart with --event-scheduler=ON, edit [mysqld].
                         SET GLOBAL alone does NOT work when DISABLED.
    Value = ON -> scheduler is running, continue to Step 2.
  Step 2 : SELECT * FROM INFORMATION_SCHEMA.EVENTS WHERE EVENT_NAME = 'name' ;
    STATUS = DISABLED -> event is disabled.
                          Fix : ALTER EVENT name ENABLE ;
    STATUS = SLAVESIDE_DISABLED -> DISABLE ON SLAVE was set and this server is a replica.
                          Fix : nothing if intentional ; check the primary executes it.
    LAST_EXECUTED IS NULL but ENABLED -> first run not yet due.
                          Fix : check ON SCHEDULE clause and STARTS timestamp.
    LAST_EXECUTED is recent but no rows in target -> the DO body raised an error.
                          Fix : check error log ; wrap body in DECLARE HANDLER if expected.
  Step 3 : ON COMPLETION NOT PRESERVE was set on a one-shot AT event.
            Result : the event ran, completed, and was auto-dropped.
                     Fix : use ON COMPLETION PRESERVE if you need it to remain visible.
```

### Tree 3 : "Why is my view slow?"

```
EXPLAIN SELECT ... FROM view_name :
  type = ALL on a table the view selects from ?
    Possible cause 1 : ALGORITHM = TEMPTABLE forces full materialization.
       Fix : ALTER VIEW ... ALGORITHM = UNDEFINED ; or MERGE.
    Possible cause 2 : view body uses DISTINCT, GROUP BY, aggregate, or UNION.
       Result : optimizer is forced to TEMPTABLE because the view is non-mergeable.
       Fix : if the aggregate is the requirement, accept the cost ; or use the
             materialized-view workaround (snapshot table + event refresh).
    Possible cause 3 : missing index on the base table.
       Fix : see mariadb-syntax-indexing.
  select_type = DERIVED for the view in EXPLAIN ?
    -> materialization happened. Same fixes as TEMPTABLE above.
  select_type does NOT mention the view ?
    -> view was merged into the outer query, the slowness is in the base SQL.
       Fix : optimize the base SELECT, see mariadb-impl-query-optimization.
```

## Patterns

### Pattern 1 : audit-log trigger (the canonical valid use case)

```sql
-- 10.6+
CREATE OR REPLACE TABLE accounts (
  id        BIGINT      PRIMARY KEY,
  balance   DECIMAL(20,2) NOT NULL,
  status    VARCHAR(20) NOT NULL
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

### Pattern 2 : validation trigger that raises SIGNAL

```sql
-- 10.6+
CREATE OR REPLACE TRIGGER trg_check_positive_balance
  BEFORE INSERT ON accounts
  FOR EACH ROW
BEGIN
  IF NEW.balance < 0 THEN
    SIGNAL SQLSTATE '45000'
      SET MESSAGE_TEXT = 'balance must be non-negative';
  END IF;
END;
```

Prefer a `CHECK (balance >= 0)` constraint when the rule is single-row and stateless. The trigger pattern is appropriate when the validation needs to consult another table or session context.

### Pattern 3 : daily-cleanup event

```sql
-- 10.6+
-- Step 1 : enable scheduler globally (and persist in [mysqld])
SET GLOBAL event_scheduler = ON;

-- Step 2 : create the event
CREATE OR REPLACE EVENT ev_purge_old_audits
  ON SCHEDULE EVERY 1 DAY
    STARTS CURRENT_TIMESTAMP + INTERVAL 1 HOUR
  ON COMPLETION PRESERVE
  ENABLE
  COMMENT 'nightly audit retention'
DO
  DELETE FROM balance_audit
   WHERE changed_at < NOW() - INTERVAL 90 DAY;
```

`ON COMPLETION PRESERVE` keeps the event visible after its ENDS timestamp (and is harmless on recurring events). `NOT PRESERVE` on a one-shot `AT` event auto-drops the event after it fires.

### Pattern 4 : view with MERGE algorithm (the fast default)

```sql
-- 10.6+
CREATE OR REPLACE
  ALGORITHM = MERGE
  DEFINER   = 'reporting_owner'@'localhost'
  SQL SECURITY DEFINER
VIEW v_active_accounts AS
  SELECT id, balance
    FROM accounts
   WHERE status = 'active'
  WITH CHECK OPTION;
```

`WITH CHECK OPTION` blocks an UPDATE through the view that would set `status` to anything other than `'active'`, because the resulting row would be invisible to the view.

### Pattern 5 : materialized-view workaround (snapshot + scheduled refresh)

```sql
-- 10.6+ : MariaDB has NO native materialized views.
-- Approach : a real table plus an event that refreshes it.

-- Step 1 : snapshot table seeded with the expensive query
CREATE OR REPLACE TABLE mv_daily_revenue AS
  SELECT DATE(order_at)             AS d,
         SUM(total)                 AS revenue,
         COUNT(*)                   AS orders
    FROM orders
   GROUP BY DATE(order_at);

-- Step 2 : index it like any other table
CREATE INDEX ix_mv_daily_revenue_d ON mv_daily_revenue (d);

-- Step 3 : event-scheduled refresh
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

`DISABLE ON SLAVE` prevents the replica from refreshing in parallel ; the binlog of the primary replays the TRUNCATE and INSERT on the replica automatically. Use this pattern when the query is too expensive to run on every read but eventual-consistency at the chosen interval is acceptable.

### Pattern 6 : multi-trigger ordering with FOLLOWS

```sql
-- 10.2.3+
CREATE OR REPLACE TRIGGER trg_audit_login
  AFTER UPDATE ON accounts
  FOR EACH ROW
INSERT INTO login_log (account_id, at) VALUES (NEW.id, NOW(6));

CREATE OR REPLACE TRIGGER trg_audit_balance
  AFTER UPDATE ON accounts
  FOR EACH ROW
  FOLLOWS trg_audit_login
INSERT INTO balance_audit (account_id, old_bal, new_bal)
VALUES (NEW.id, OLD.balance, NEW.balance);

-- Verify : ACTION_ORDER column shows the firing sequence
SELECT TRIGGER_NAME, ACTION_ORDER
  FROM INFORMATION_SCHEMA.TRIGGERS
 WHERE EVENT_OBJECT_TABLE = 'accounts'
 ORDER BY ACTION_ORDER;
```

## Cross-References

- `mariadb-syntax-check-constraints` : prefer CHECK over BEFORE triggers for single-row validation.
- `mariadb-syntax-procedures-functions` : split was made per D-01 ; triggers / events / views split off from stored routines for separation of concerns.
- `mariadb-syntax-indexing` : view performance depends on base-table indexes ; materialized-view snapshots must be indexed explicitly.
- `mariadb-impl-query-optimization` : EXPLAIN-driven view performance diagnosis.
- `mariadb-syntax-sql-dml` : DML on views and the WITH CHECK OPTION semantics.

## Reference Links

- `references/methods.md` : complete CREATE / ALTER / DROP grammar for TRIGGER, EVENT, VIEW ; INFORMATION_SCHEMA schemata ; system variables ; SHOW commands.
- `references/examples.md` : ten working recipes : audit trigger, SIGNAL validation, multi-trigger FOLLOWS, daily-cleanup event, hourly refresh event, view with MERGE, view with TEMPTABLE (and why to avoid it), view with WITH CHECK OPTION CASCADED, materialized-view workaround, role-scoped DEFINER pattern.
- `references/anti-patterns.md` : eight real anti-patterns with root cause and fix : N+1 trigger lookups, event_scheduler OFF, TEMPTABLE on huge view, materialized-view expectation, dropped DEFINER, replication-format trigger desync, IGNORE-hidden CHECK OPTION violation, transaction control inside trigger.

## Sources

All claims verified 2026-05-19 via WebFetch against the MariaDB Knowledge Base.

- KB `create-trigger/` : trigger grammar, FOLLOWS / PRECEDES, NEW / OLD scoping.
- KB `trigger-overview/` : BEFORE / AFTER timing for INSERT / UPDATE / DELETE.
- KB `trigger-limitations/` : forbidden statements (COMMIT / ROLLBACK / RETURN / SELECT result-set).
- KB `create-event/` : event grammar, ON SCHEDULE, ENABLE / DISABLE / DISABLE ON SLAVE.
- KB `events/` : event_scheduler global, INFORMATION_SCHEMA.EVENTS columns.
- KB `create-view/` : ALGORITHM, SQL SECURITY, DEFINER, WITH CHECK OPTION, LOCAL vs CASCADED.
- KB `create-or-replace/` : CREATE OR REPLACE semantics on triggers, events, views.
