# Cluster 2 Research : Advanced SQL and Routines

Scope : window functions, CTEs, system-versioned tables (**MariaDB-only**), application-time periods (**MariaDB-only**), stored procedures, stored functions, triggers, events, views, EXPLAIN / ANALYZE / optimizer flags. All claims verified via WebFetch against [mariadb.com/kb](https://mariadb.com/kb/) on 2026-05-19.

Versions in scope : MariaDB 10.6 LTS, 10.11 LTS, 11.x current, 12.x next. Divergence from MySQL is flagged inline.

---

## 1. Window functions

Window functions ([KB](https://mariadb.com/kb/en/window-functions-overview/)) were introduced in MariaDB 10.2. They compute a value over a window of rows defined by `OVER()` without collapsing rows the way `GROUP BY` does.

```sql
-- 10.2+
SELECT
  emp_id,
  dept_id,
  salary,
  RANK()       OVER w AS dept_rank,
  AVG(salary)  OVER w AS dept_avg,
  LAG(salary, 1, 0) OVER (PARTITION BY dept_id ORDER BY hire_date) AS prev_salary
FROM employees
WINDOW w AS (PARTITION BY dept_id ORDER BY salary DESC);
```

`OVER()` accepts three optional sub-clauses : `PARTITION BY expr [, ...]`, `ORDER BY expr [ASC|DESC] [, ...]`, and a frame `{ROWS|RANGE} frame_clause` ([KB](https://mariadb.com/kb/en/window-frames/)). Frame borders : `UNBOUNDED PRECEDING`, `n PRECEDING`, `CURRENT ROW`, `n FOLLOWING`, `UNBOUNDED FOLLOWING`, optionally `BETWEEN border AND border`.

**Critical MariaDB constraint** : only `ROWS` and `RANGE` frame units are supported. **`GROUPS` is NOT implemented in MariaDB** at any tested version up to 12.x ([KB](https://mariadb.com/kb/en/window-frames/)) ; do not assume parity with PostgreSQL or MySQL 8.0+ here. Frame exclusion (`EXCLUDE CURRENT ROW`, etc.), explicit `NULLS FIRST/LAST`, and `DISTINCT` inside aggregate-as-window are also unsupported.

Default frame, when `ORDER BY` is present but no frame is specified : `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` ([KB](https://mariadb.com/kb/en/window-frames/)). Without `ORDER BY`, the entire partition is the window. This default matters : `SUM(x) OVER (ORDER BY t)` produces a running total, not a partition-wide sum.

Ranking functions ([KB](https://mariadb.com/kb/en/window-functions-overview/)) : `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()`, `PERCENT_RANK()`, `CUME_DIST()`, `NTILE(n)`. Ranking functions ignore any frame clause. `RANK` leaves gaps on ties (1, 1, 3) ; `DENSE_RANK` does not (1, 1, 2).

Value functions ([KB](https://mariadb.com/kb/en/lag/)) : `LAG(expr [, offset [, default]])`, `LEAD(expr [, offset [, default]])`, `FIRST_VALUE(expr)`, `LAST_VALUE(expr)`, `NTH_VALUE(expr, n)`. All require `ORDER BY` in `OVER()`. `LAST_VALUE` interacts with the default running frame : without an explicit `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`, it returns the current row, not the partition tail.

Aggregates that work as window functions : `SUM`, `AVG`, `COUNT`, `MIN`, `MAX`, `BIT_AND/OR/XOR`, `STDDEV`, `VARIANCE`, `JSON_ARRAYAGG`, `JSON_OBJECTAGG`. Named windows (`WINDOW w AS (...)`) eliminate duplication across multiple window expressions in the same `SELECT`.

---

## 2. Common Table Expressions and recursive queries

CTEs ([KB](https://mariadb.com/kb/en/with/)) entered in MariaDB 10.2.1 (non-recursive) and 10.2.2 (recursive).

```sql
-- 10.2.2+
WITH RECURSIVE org_tree (emp_id, manager_id, depth, path) AS (
  SELECT emp_id, manager_id, 0, CAST(emp_id AS CHAR(1000))
    FROM employees WHERE manager_id IS NULL
  UNION ALL
  SELECT e.emp_id, e.manager_id, t.depth + 1,
         CONCAT(t.path, ',', e.emp_id)
    FROM employees e JOIN org_tree t ON e.manager_id = t.emp_id
   WHERE t.depth < 20
)
SELECT * FROM org_tree;
```

Cycle detection : MariaDB 10.5.2+ adds `CYCLE col_list RESTRICT` after the CTE body ([KB](https://mariadb.com/kb/en/with/)) ; before 10.5.2, you embed a guard column (the `path`-string pattern above) or use `max_recursive_iterations` to cap depth. Termination of the recursive arm is the developer's responsibility ; otherwise `max_recursive_iterations` (default 1000) aborts with an error.

Materialization : MariaDB may merge a non-recursive CTE into the outer query or materialize it into a temp table ; the optimizer chooses based on `optimizer_switch` flags `derived_merge=on` and `condition_pushdown_for_derived=on` ([KB](https://mariadb.com/kb/en/optimizer-switch/)). Use `EXPLAIN` to confirm — a `DERIVED` row in `select_type` means materialized, an inline expansion means merged. Recursive CTEs are always materialized.

Use cases the KB documents : tree-walks, gap-and-island reduction (combine with `ROW_NUMBER`), top-N per group (CTE + window-function filter), and graph traversal with cycle-safe enumeration.

---

## 3. System-versioned tables (**MariaDB-only**) and application-time periods (**MariaDB-only**)

System-versioned tables ([KB](https://mariadb.com/kb/en/system-versioned-tables/)) arrived in MariaDB 10.3 and are an SQL:2011 feature. MySQL has no equivalent ; this is a major divergence.

```sql
-- 10.3+
CREATE TABLE accounts (
  id INT PRIMARY KEY,
  balance DECIMAL(15,2),
  row_start TIMESTAMP(6) GENERATED ALWAYS AS ROW START,
  row_end   TIMESTAMP(6) GENERATED ALWAYS AS ROW END,
  PERIOD FOR SYSTEM_TIME (row_start, row_end)
) WITH SYSTEM VERSIONING;

-- query history
SELECT * FROM accounts FOR SYSTEM_TIME AS OF TIMESTAMP '2026-01-01 00:00:00';
SELECT * FROM accounts FOR SYSTEM_TIME BETWEEN '2026-01-01' AND '2026-04-30';
SELECT * FROM accounts FOR SYSTEM_TIME FROM '2026-01-01' TO '2026-05-01'; -- end-exclusive
SELECT * FROM accounts FOR SYSTEM_TIME ALL;
```

Implicit form `WITH SYSTEM VERSIONING` auto-generates the period columns invisibly. The explicit form lets you name them and is required for partition-by-system-time ([KB](https://mariadb.com/kb/en/system-versioned-tables/)).

Transaction-precision (InnoDB only) uses `BIGINT UNSIGNED` columns of transaction IDs instead of timestamps : `SELECT ... FOR SYSTEM_TIME AS OF TRANSACTION 12345`. Trade-off : transaction-precision tables cannot use `PARTITION BY SYSTEM_TIME`.

History pruning : `DELETE HISTORY FROM t [BEFORE SYSTEM_TIME 'ts']` requires the `DELETE HISTORY` privilege. Partition-by-system-time keeps history separable for fast purge :

```sql
-- 10.9.1+
CREATE TABLE audit_log (event_id BIGINT, payload TEXT) WITH SYSTEM VERSIONING
  PARTITION BY SYSTEM_TIME INTERVAL 1 HOUR AUTO; -- auto-rotate
```

Application-time periods ([KB](https://mariadb.com/kb/en/application-time-periods/), 10.4+) are user-defined, business-meaningful date ranges :

```sql
-- 10.4+
CREATE TABLE prices (
  product_id INT, price DECIMAL(10,2),
  valid_from DATE, valid_to DATE,
  PERIOD FOR validity (valid_from, valid_to)
);
DELETE FROM prices FOR PORTION OF validity FROM '2026-03-01' TO '2026-04-01';
UPDATE prices FOR PORTION OF validity FROM '2026-03-01' TO '2026-04-01'
  SET price = 9.99;
```

Bitemporal (10.5+) combines both : `WITH SYSTEM VERSIONING` plus a user `PERIOD FOR` clause on different column-pairs ([KB](https://mariadb.com/kb/en/application-time-periods/)).

---

## 4. Stored procedures

`CREATE PROCEDURE` ([KB](https://mariadb.com/kb/en/create-procedure/)) accepts parameters with three modes : `IN` (default, copy-in), `OUT` (copy-out at return), `INOUT` (both). MariaDB 11.8 adds Oracle-mode `IN OUT` and per-parameter `DEFAULT`. Characteristics : `LANGUAGE SQL`, `[NOT] DETERMINISTIC` (informative-only for procedures), `CONTAINS SQL | NO SQL | READS SQL DATA | MODIFIES SQL DATA`, `SQL SECURITY {DEFINER|INVOKER}`, `COMMENT 'string'`.

```sql
DELIMITER //
CREATE OR REPLACE PROCEDURE transfer(IN src INT, IN dst INT, IN amt DECIMAL(15,2))
  MODIFIES SQL DATA
  SQL SECURITY INVOKER
BEGIN
  DECLARE bal DECIMAL(15,2);
  DECLARE EXIT HANDLER FOR SQLEXCEPTION
  BEGIN
    ROLLBACK;
    RESIGNAL;
  END;
  START TRANSACTION;
  SELECT balance INTO bal FROM accounts WHERE id = src FOR UPDATE;
  IF bal < amt THEN
    SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'insufficient funds';
  END IF;
  UPDATE accounts SET balance = balance - amt WHERE id = src;
  UPDATE accounts SET balance = balance + amt WHERE id = dst;
  COMMIT;
END//
DELIMITER ;
```

Local variables : `DECLARE var_name type [DEFAULT expr]` (must come before any other statement in the block). Control flow : `IF ... THEN ... ELSEIF ... ELSE ... END IF`, `CASE`, `LOOP label`, `WHILE`, `REPEAT ... UNTIL`, `ITERATE label`, `LEAVE label`. Cursors : `DECLARE cur CURSOR FOR SELECT ...`, then `OPEN cur`, `FETCH cur INTO ...`, `CLOSE cur`.

Error handling ([KB](https://mariadb.com/kb/en/declare-handler/)) : `DECLARE {CONTINUE|EXIT} HANDLER FOR condition statement`. Conditions : a five-char `SQLSTATE 'xxxxx'` (e.g. `'23000'` = duplicate key), a numeric MariaDB error code, or the shorthands `SQLWARNING` (class `01`), `NOT FOUND` (class `02`), `SQLEXCEPTION` (everything else). Custom errors are raised with `SIGNAL SQLSTATE 'xxxxx' SET MESSAGE_TEXT='msg', MYSQL_ERRNO=n` ([KB](https://mariadb.com/kb/en/signal/)) ; inside a handler, `RESIGNAL` re-throws the caught error preserving context. Recursion is capped by `max_sp_recursion_depth` (default 0, must be raised for recursive procedures up to 255).

---

## 5. Stored functions

`CREATE FUNCTION` ([KB](https://mariadb.com/kb/en/create-function/)) differs from procedures : it has a mandatory `RETURNS type` clause, all parameters are `IN` by default (10.8+ adds `OUT`/`INOUT` but only for `SET`-style calls), and the body must execute `RETURN expr` to produce a value.

```sql
DELIMITER //
CREATE OR REPLACE FUNCTION net_price(gross DECIMAL(10,2), vat_rate DECIMAL(5,4))
  RETURNS DECIMAL(10,2)
  DETERMINISTIC
  CONTAINS SQL
BEGIN
  RETURN gross / (1 + vat_rate);
END//
DELIMITER ;
```

Characteristics : `[NOT] DETERMINISTIC` is **load-bearing** for functions, unlike procedures. The optimizer cannot inline or cache a `NOT DETERMINISTIC` call, and any non-deterministic function in a `WHERE` clause forces a per-row evaluation that defeats index usage on its result ([KB](https://mariadb.com/kb/en/create-function/)). Functions using `NOW()`, `CURRENT_TIMESTAMP()`, `RAND()`, or session variables are inherently non-deterministic ; declaring them `DETERMINISTIC` "may yield incorrect results" per the KB. Data-access clauses : `CONTAINS SQL` (default), `READS SQL DATA`, `MODIFIES SQL DATA`, `NO SQL` (currently a no-op in MariaDB). `SQL SECURITY {DEFINER|INVOKER}` controls privilege escalation.

---

## 6. Triggers

`CREATE TRIGGER` ([KB](https://mariadb.com/kb/en/create-trigger/)) fires `BEFORE` or `AFTER` an `INSERT`, `UPDATE`, or `DELETE`, always `FOR EACH ROW` (MariaDB does not implement statement-level triggers). Pseudo-records : `NEW.col` is readable in `INSERT` and `UPDATE` triggers (writable only in `BEFORE` triggers with `UPDATE` privilege via `SET NEW.col = expr`) ; `OLD.col` is read-only in `UPDATE` and `DELETE` triggers.

```sql
-- 10.2.3+
CREATE TRIGGER trg_audit_balance AFTER UPDATE ON accounts
  FOR EACH ROW
  FOLLOWS trg_audit_login
INSERT INTO balance_audit (account_id, old_bal, new_bal, changed_by, changed_at)
VALUES (NEW.id, OLD.balance, NEW.balance, CURRENT_USER(), NOW(6));
```

Multiple triggers on the same event are supported from 10.2.3+ with explicit ordering via `FOLLOWS trigger_name` or `PRECEDES trigger_name` ([KB](https://mariadb.com/kb/en/create-trigger/)). Order is observable in `INFORMATION_SCHEMA.TRIGGERS.ACTION_ORDER`. Limitations ([KB](https://mariadb.com/kb/en/trigger-limitations/)) : a trigger cannot return a result set (`SELECT` with output is forbidden, use `SELECT ... INTO var` or `INSERT ... SELECT`), the `RETURN` statement is illegal (use `LEAVE` to exit early), transaction-control statements (`COMMIT`, `ROLLBACK`, `SAVEPOINT`, `START TRANSACTION`) are forbidden, and triggers may not be defined on tables in `mysql`, `information_schema`, or `performance_schema`. Replication interaction : with row-based binlog (`binlog_format=ROW`), triggers execute on the source and their row-effects are replicated, so the replica does not re-fire them. With `STATEMENT` or `MIXED`, the trigger re-runs on replicas, which can desync if the trigger is non-deterministic.

---

## 7. Events (scheduler)

`CREATE EVENT` ([KB](https://mariadb.com/kb/en/create-event/)) defines a scheduled task executed by the event scheduler thread.

```sql
CREATE OR REPLACE EVENT ev_purge_old_audits
  ON SCHEDULE EVERY 1 DAY
    STARTS CURRENT_TIMESTAMP + INTERVAL 1 HOUR
  ON COMPLETION PRESERVE
  ENABLE
  COMMENT 'nightly audit retention'
DO
  DELETE FROM balance_audit WHERE changed_at < NOW() - INTERVAL 90 DAY;
```

Scheduling : `ON SCHEDULE AT timestamp` for one-shot, `ON SCHEDULE EVERY n unit [STARTS ts] [ENDS ts]` for recurring. `ON COMPLETION PRESERVE` retains a one-shot event after it finishes ; `NOT PRESERVE` (default) drops it. State : `ENABLE` (default), `DISABLE`, `DISABLE ON SLAVE` (avoid running the same event twice in replication topologies). The scheduler thread itself is gated by the global `event_scheduler` variable (`ON`, `OFF`, `DISABLED`) ; `OFF` at default in many distributions, so events created on a fresh server silently never fire until it is enabled. Monitor with `SELECT * FROM INFORMATION_SCHEMA.EVENTS` for status, `LAST_EXECUTED`, and `STATUS`. Error handling inside the `DO` body uses the same `DECLARE HANDLER` mechanism as procedures ; an uncaught error aborts that one execution but does not disable the event.

---

## 8. Views

`CREATE VIEW` ([KB](https://mariadb.com/kb/en/create-view/)) wraps a `SELECT` as a virtual table. The `ALGORITHM` clause selects the resolution strategy : `UNDEFINED` (default, optimizer chooses), `MERGE` (rewrite : view-body inlined into the referencing query, preserves index usage and updatability when possible), or `TEMPTABLE` (materialize the view into a temp table per query, breaks index pushdown and forces a full scan on each reference).

```sql
CREATE OR REPLACE
  ALGORITHM = MERGE
  DEFINER = 'reporting_owner'@'localhost'
  SQL SECURITY DEFINER
VIEW v_active_accounts AS
  SELECT id, balance FROM accounts WHERE status = 'active'
  WITH CHECK OPTION;
```

`SQL SECURITY DEFINER` (default) runs with the view-creator's privileges, the typical pattern for exposing a subset of a sensitive table ; `INVOKER` defers privilege checks to the caller. Updatable views require a one-to-one row mapping between view rows and underlying-table rows ; `DISTINCT`, `GROUP BY`, aggregates, `UNION`, and most subqueries break updatability ([KB](https://mariadb.com/kb/en/create-view/)). `WITH CHECK OPTION` rejects `INSERT`/`UPDATE` that would produce a row outside the view's predicate ; `CASCADED` (default) propagates the check through chained views, `LOCAL` checks only the current view. **Materialized views are NOT supported in MariaDB** — workarounds : `CREATE TABLE AS SELECT` plus a refresh job via the event scheduler, or external tools such as Flexviews.

---

## 9. EXPLAIN and the optimizer

`EXPLAIN` ([KB](https://mariadb.com/kb/en/explain/)) returns the query plan without executing. Columns : `id` (join sequence), `select_type`, `table`, `type`, `possible_keys`, `key`, `key_len`, `ref`, `rows`, `filtered`, `Extra`.

`select_type` values : `SIMPLE`, `PRIMARY`, `UNION`, `DEPENDENT UNION`, `UNION RESULT`, `SUBQUERY`, `DEPENDENT SUBQUERY`, `DERIVED`, `MATERIALIZED`, `UNCACHEABLE SUBQUERY`, `UNCACHEABLE UNION`.

`type` ranking (best to worst, ([KB](https://mariadb.com/kb/en/explain/))) : `system`, `const`, `eq_ref`, `ref`, `fulltext`, `ref_or_null`, `index_merge`, `unique_subquery`, `index_subquery`, `range`, `index`, `ALL`. The boundary between "fine" and "fix it now" is at `range` ; anything below should be examined.

Common `Extra` flags : `Using where`, `Using index` (covering-index, no row read), `Using filesort` (sort step), `Using temporary` (intermediate temp table for `GROUP BY` or `DISTINCT`), `Using join buffer (Block Nested Loop)`, `Using index condition` (index-condition pushdown to the storage engine), `Range checked for each record` (no usable index decided up-front, re-evaluated per outer row), `Not exists` (LEFT-JOIN short-circuit), `Select tables optimized away`.

`ANALYZE [FORMAT=JSON] SELECT ...` ([KB](https://mariadb.com/kb/en/analyze-statement/), 10.1+) actually runs the query and returns the plan **plus** observed values `r_rows`, `r_filtered`, `r_total_time_ms`. Use it to validate optimizer estimates. Warning : `ANALYZE UPDATE` and `ANALYZE DELETE` do execute the mutation ; only `ANALYZE SELECT` discards results.

Index hints ([KB](https://mariadb.com/kb/en/index-hints-how-to-force-query-plans/)) : `USE INDEX (idx_a, idx_b)` restricts the candidate set, `FORCE INDEX (...)` additionally treats table-scan as prohibitively expensive, `IGNORE INDEX (...)` excludes specific indexes while leaving the rest considered. All three accept `FOR JOIN`, `FOR ORDER BY`, `FOR GROUP BY` modifiers. **Risk** : `FORCE INDEX` on stale statistics locks you to a bad plan ; run `ANALYZE TABLE` first, then prefer `IGNORE INDEX` for the smaller blast-radius.

`optimizer_switch` ([KB](https://mariadb.com/kb/en/optimizer-switch/)) toggles individual optimizations. Defaults on : `index_merge`, `index_merge_union`, `index_merge_sort_union`, `index_merge_intersection`, `index_condition_pushdown`, `semijoin`, `materialization`, `loosescan`, `firstmatch`, `subquery_cache`, `derived_merge`, `derived_with_keys`, `condition_pushdown_for_derived`, `condition_pushdown_from_having`, `rowid_filter` (10.4.3+). Defaults off : `mrr`, `mrr_cost_based`, `index_merge_sort_intersection`. For deep-dive diagnostics enable `optimizer_trace=enabled=on` and read `INFORMATION_SCHEMA.OPTIMIZER_TRACE`.

---

## Anti-patterns

1. **`LAST_VALUE` without an explicit frame.** `LAST_VALUE(x) OVER (PARTITION BY g ORDER BY t)` returns the current row, not the partition tail, because the default frame is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` ([KB](https://mariadb.com/kb/en/window-frames/)). Fix : add `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`.

2. **Recursive CTE without a termination predicate.** A self-joining anchor on cyclic graph data hits `max_recursive_iterations` (default 1000) and errors out, sometimes after burning minutes. Fix : add a depth guard (`WHERE depth < N`) or use `CYCLE col RESTRICT` (10.5.2+).

3. **System-versioned table without history partitioning.** Writes append history rows to the same physical structure as live rows ; over years the table degrades to scan-bound. Fix : `PARTITION BY SYSTEM_TIME` with explicit `HISTORY` and `CURRENT` partitions, or `INTERVAL ... AUTO` (10.9.1+), plus periodic `DELETE HISTORY BEFORE SYSTEM_TIME ...` ([KB](https://mariadb.com/kb/en/system-versioned-tables/)).

4. **Trigger doing N+1 lookups.** `AFTER INSERT FOR EACH ROW` calling a `SELECT` against another large table multiplies a 10k-row bulk insert by 10k point-queries. Fix : batch-aggregate in application code, or move logic to a stored procedure invoked once.

5. **`FUNCTION` marked `DETERMINISTIC` but reading session variables.** `RETURN @session_tax_rate * gross` declared deterministic lets the optimizer cache or hoist the call, producing stale results when the variable changes mid-query. Fix : drop `DETERMINISTIC`, accept the per-row evaluation, or pass the rate as a parameter ([KB](https://mariadb.com/kb/en/create-function/)).

6. **View with `ALGORITHM=TEMPTABLE` on a huge base table.** Each reference materializes the entire result into a temp table, with no index pushdown. Fix : default to `UNDEFINED` and trust the optimizer ; use `TEMPTABLE` only when the view body has non-mergeable constructs that you specifically want frozen.

7. **Ignoring `type=ALL` in `EXPLAIN`.** Production code shipped because the query "works on dev". Fix : block any new query whose `EXPLAIN type` is `ALL` against a table over a documented row-threshold ; add an index or rewrite.

8. **`FORCE INDEX` with stale statistics.** After bulk-loading 10 million rows without `ANALYZE TABLE`, the cardinality estimates are wrong ; a previously good `FORCE INDEX` now ignores a better composite. Fix : run `ANALYZE TABLE` after large data shifts, prefer `IGNORE INDEX` when you only want to exclude a specific path.

9. **Event scheduler off.** Created a nightly purge event, but `event_scheduler=OFF` (the default on many distros) means nothing runs. Fix : `SET GLOBAL event_scheduler = ON` and persist in `[mysqld]` config ; verify with `SHOW VARIABLES LIKE 'event_scheduler'`.

10. **Trigger with `COMMIT` or `ROLLBACK`.** Compiles on creation but raises `Explicit or implicit commit is not allowed in stored function or trigger` at runtime ([KB](https://mariadb.com/kb/en/trigger-limitations/)). Fix : move transactional logic to a stored procedure called by the application.

---

## Newly Discovered Sub-Topics

1. **`SYS_REFCURSOR` and Oracle-mode `IN OUT` parameters (11.8+)** — newer MariaDB function-return capability worth a dedicated section in `mariadb-syntax-stored-routines`.
2. **`system_versioning_insert_history` (10.11+)** — allows direct insertion of historical rows, useful for data import / migration into versioned tables.
3. **`max_recursive_iterations` cap on recursive CTEs** — anti-pattern coverage should reference this exact variable, not generic "depth limit".
4. **`ANALYZE UPDATE` / `ANALYZE DELETE` actually mutate** — must be flagged as a footgun in the `mariadb-impl-query-optimization` skill, distinct from `ANALYZE SELECT`.
5. **`PARTITION BY SYSTEM_TIME INTERVAL ... AUTO` (10.9.1+)** — auto-partition creation : new sub-topic for system-versioning skill.
6. **`optimizer_trace` system variable + `INFORMATION_SCHEMA.OPTIMIZER_TRACE`** — deep-dive diagnostic tool beyond `EXPLAIN FORMAT=JSON`, deserves its own section.
7. **MariaDB has no `GROUPS` frame** — explicit divergence from PostgreSQL and MySQL 8.0+ that must be called out in the window-functions skill to prevent hallucinated syntax.
8. **Multi-source replication interaction with triggers** — triggers fire per source, not per replicated event, when row-based logging is on ; intersects with the replication-cluster research.
