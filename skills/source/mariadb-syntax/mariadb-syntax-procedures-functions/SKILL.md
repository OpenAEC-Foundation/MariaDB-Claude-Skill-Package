---
name: mariadb-syntax-procedures-functions
description: >
  Use when writing stored procedures or stored functions, designing error handling with DECLARE HANDLER, using SIGNAL and RESIGNAL for custom errors, choosing DETERMINISTIC vs NOT DETERMINISTIC, or porting MySQL procedures to MariaDB.
  Prevents the common mistake of marking a function DETERMINISTIC when it reads NOW() or session variables, putting heavy stored-function logic in WHERE clauses (defeats index usage), missing EXIT or CONTINUE handlers leading to silent loop-abort, and forgetting to raise max_sp_recursion_depth before calling a recursive procedure.
  Covers CREATE PROCEDURE syntax with IN / OUT / INOUT parameters, CREATE FUNCTION with RETURNS, DELIMITER usage, local DECLARE variables, control flow (IF, CASE, LOOP, WHILE, REPEAT, ITERATE, LEAVE), cursor lifecycle (DECLARE / OPEN / FETCH / CLOSE with NOT FOUND handler), error handling via DECLARE HANDLER for CONTINUE / EXIT on SQLSTATE / SQLWARNING / NOT FOUND / SQLEXCEPTION / numeric error codes, SIGNAL with SQLSTATE 45000 and MESSAGE_TEXT, RESIGNAL inside a handler, DETERMINISTIC vs NOT DETERMINISTIC semantics for the optimizer, READS SQL DATA / MODIFIES SQL DATA / CONTAINS SQL / NO SQL data-access clauses, SQL SECURITY DEFINER vs INVOKER, max_sp_recursion_depth tuning, and the 11.8+ IN OUT and per-parameter DEFAULT Oracle-mode extensions.
  Keywords: stored procedure, stored function, CREATE PROCEDURE, CREATE FUNCTION, DELIMITER, IN OUT INOUT parameters, DECLARE variable, IF THEN ELSEIF, CASE WHEN, LOOP, WHILE, REPEAT UNTIL, ITERATE, LEAVE, cursor, DECLARE CURSOR, OPEN FETCH CLOSE, DECLARE HANDLER, CONTINUE HANDLER, EXIT HANDLER, SQLSTATE, SQLWARNING, NOT FOUND, SQLEXCEPTION, SIGNAL, RESIGNAL, MESSAGE_TEXT, MYSQL_ERRNO, 45000, DETERMINISTIC, NOT DETERMINISTIC, CONTAINS SQL, READS SQL DATA, MODIFIES SQL DATA, NO SQL, SQL SECURITY DEFINER, SQL SECURITY INVOKER, recursion limit, max_sp_recursion_depth, IN OUT 11.8, DEFAULT parameter 11.8, SYS_REFCURSOR, OUT parameter SET only, why is my function slow in WHERE, function blocks index, procedure cursor loop hangs, silent abort in handler, how do I raise a custom error, how do I catch duplicate key, function deterministic mistake, recursive procedure error, DELIMITER not reset
license: MIT
compatibility: "Designed for Claude Code. Requires MariaDB 10.6-LTS, 10.11-LTS, 11.x, 12.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# MariaDB Stored Procedures and Stored Functions

Deterministic guidance for writing stored procedures, stored functions, control flow, cursors, error handlers (DECLARE HANDLER, SIGNAL, RESIGNAL), and choosing DETERMINISTIC vs NOT DETERMINISTIC. This skill covers the routine body itself. For triggers, events, and views, see the sibling skill `mariadb-syntax-triggers-events-views`.

## Quick Reference

- **ALWAYS use `DELIMITER //` (or any non-semicolon) before `CREATE PROCEDURE` / `CREATE FUNCTION` in client tools** so the `;` inside the body is not interpreted as end-of-statement. Reset with `DELIMITER ;` afterwards. This is a client-side directive, NOT SQL. Drivers that send one statement per call (PDO, JDBC `executeUpdate`) do not need DELIMITER ; the multi-statement client (`mariadb` CLI, MySQL Workbench) does. Verified : KB `create-procedure/`.
- **ALWAYS write `DETERMINISTIC` only when the function truly is deterministic**. A function is deterministic only when "it can produce only one result for a given list of parameters". Functions that call `NOW()`, `CURRENT_TIMESTAMP()`, `RAND()`, `UUID()`, `CONNECTION_ID()`, `LAST_INSERT_ID()`, or that read session variables (`@var`) or non-immutable table data are NOT deterministic. Mis-declaring as DETERMINISTIC lets the optimizer cache or hoist the call, producing wrong results. Verified : KB `create-function/`.
- **NEVER call a NOT DETERMINISTIC function in a WHERE clause on a large table**. The optimizer cannot fold the call to a constant, so it is re-evaluated per row, no index on the function-result can be used, and the plan degrades to a full scan with a per-row procedure call. ALWAYS push the predicate to a constant-folded expression in the application, or store the result in a column.
- **ALWAYS declare local variables and cursors and handlers in the correct order inside `BEGIN ... END`** : (1) local `DECLARE var TYPE`, then (2) `DECLARE condition_name CONDITION`, then (3) `DECLARE cursor_name CURSOR`, then (4) `DECLARE ... HANDLER`. Any other order is a syntax error. Verified : KB `declare-cursor/`.
- **ALWAYS attach `DECLARE CONTINUE HANDLER FOR NOT FOUND` to detect end-of-cursor**. The handler sets a flag the LOOP body checks ; without it, FETCH past the last row raises SQLSTATE `02000` and the procedure either silently exits (no handler) or hangs forever (handler badly written). Pattern : `DECLARE done INT DEFAULT 0 ; DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = 1 ;`.
- **ALWAYS use `SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = '...'` for user-defined errors**. `45000` is the documented class for application-raised conditions ; it surfaces as an error (not a warning) when unhandled. Optionally set `MYSQL_ERRNO = <integer>` for a custom numeric code. Verified : KB `signal/`.
- **ALWAYS use `RESIGNAL` (not `SIGNAL`) inside a handler to re-raise the caught error**. `RESIGNAL` with no arguments re-throws the original condition unchanged ; `RESIGNAL SET MESSAGE_TEXT = '...'` re-throws with an overridden message but preserves the original SQLSTATE and MYSQL_ERRNO. Using `SIGNAL` instead loses the original error context. Verified : KB `resignal/`.
- **ALWAYS raise `max_sp_recursion_depth` before calling a recursive procedure**. Default is `0`, which forbids recursion ; max is `255`. Set per-session : `SET @@SESSION.max_sp_recursion_depth = 32 ;`. Without this, the second call into the same procedure errors with `Recursive limit 0 (as set by the max_sp_recursion_depth variable) was exceeded`. Verified : KB `create-procedure/`.
- **ALWAYS pick the right characteristic clause**. Functions default to `CONTAINS SQL` and `NOT DETERMINISTIC`. Set `READS SQL DATA` when the body has `SELECT`, `MODIFIES SQL DATA` when it has `INSERT` / `UPDATE` / `DELETE` / DDL. These clauses are required to be accurate when `log_bin_trust_function_creators = OFF` (the secure default) ; otherwise `CREATE FUNCTION` is refused for binlog-safety. Verified : KB `create-function/`.
- **ALWAYS prefer `SQL SECURITY INVOKER` for routines that touch user data**. The `DEFINER` default runs the body with the creator's privileges, which is convenient for exposing a subset of sensitive tables but is a privilege-escalation vector if the routine is misused. `INVOKER` defers privilege checks to the caller. Audit `SHOW CREATE PROCEDURE` regularly for unexpected `DEFINER` accounts.
- **NEVER use `INVISIBLE`, `BEFORE` triggers, or trigger-style transaction control inside a function**. Triggers and views are not in this skill (see sibling `mariadb-syntax-triggers-events-views`) but the rule that "functions cannot contain transaction control" (`COMMIT`, `ROLLBACK`, `SAVEPOINT`, `START TRANSACTION`) is enforced at runtime with `Explicit or implicit commit is not allowed in stored function or trigger`. Procedures CAN contain transaction control.
- **`OUT` / `INOUT` / `IN OUT` on a function (10.8+)** can ONLY be called via `SET` and `CALL`, NEVER inside `SELECT`. Attempting `SELECT myfunc(@x)` on a function with OUT/INOUT parameters raises error 4186. For most cases : if you need multi-value return, write a PROCEDURE with OUT parameters, not a FUNCTION. Verified : KB `create-function/`.
- **MariaDB 11.8+ adds Oracle-mode `IN OUT` and per-parameter `DEFAULT`**. On 11.8 and later, `CREATE PROCEDURE p(IN OUT a INT, IN b INT DEFAULT 0)` is legal. On 10.6 through 11.7, only `IN | OUT | INOUT` modes exist and DEFAULT is a syntax error. Verified : KB `create-procedure/`.

## Decision Trees

### Tree 1 : Procedure or function?

```
What does the routine do?
  Returns ONE scalar value, called inside expressions (SELECT, WHERE, SET) ?
    -> FUNCTION with RETURNS type.
    Is the result reproducible from inputs alone (no clock, no random, no session vars) ?
      Yes -> DETERMINISTIC.
      No  -> NOT DETERMINISTIC (default). Avoid using in WHERE on large tables.

  Returns multiple values, or no value, or mutates many rows ?
    -> PROCEDURE.
    Call site uses CALL p(...) ;
    Needs to return values ?     -> OUT or INOUT parameters.
    Needs to commit / rollback ? -> ALLOWED in a procedure (NOT in a function).
    Needs to be called from SELECT row-by-row ? -> NO. Functions can be ;
                                                   procedures CANNOT.
```

### Tree 2 : Which handler do I need?

```
What condition do you want to catch?
  Any error (catch-all) ?
    -> DECLARE EXIT HANDLER FOR SQLEXCEPTION ...   -- aborts the BEGIN block
    -> DECLARE CONTINUE HANDLER FOR SQLEXCEPTION ... -- swallows, continues
  Specifically duplicate-key (ER_DUP_ENTRY) ?
    -> SQLSTATE '23000' or numeric 1062
  End of cursor data ?
    -> DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = 1 ;
  Any warning ?
    -> DECLARE CONTINUE HANDLER FOR SQLWARNING ...

Handler scope = the BEGIN ... END block where it is DECLAREd, including inner blocks
unless overridden by an inner handler. EXIT terminates the block, CONTINUE returns
to the statement after the failing one.
```

### Tree 3 : DETERMINISTIC vs NOT DETERMINISTIC

```
Does the function body call any of :
  NOW(), CURRENT_TIMESTAMP(), SYSDATE(), CURRENT_DATE(), CURRENT_TIME() ?
  RAND(), UUID(), CONNECTION_ID(), LAST_INSERT_ID() ?
  Read a session variable (@var) or @@session.<var> ?
  SELECT from any table not guaranteed immutable ?
  Call another NOT DETERMINISTIC function ?

  Any "yes"  -> NOT DETERMINISTIC (default). Document why.
  All "no"   -> DETERMINISTIC. The optimizer can cache the call and replicate
                safely under STATEMENT binlog.
```

### Tree 4 : SIGNAL vs RESIGNAL

```
Am I currently inside an active handler?
  No  -> SIGNAL SQLSTATE 'xxxxx' SET MESSAGE_TEXT = '...' ;
         (Used to raise NEW application-level errors. Use SQLSTATE '45000'.)

  Yes -> RESIGNAL ;                                    -- re-throw unchanged
         RESIGNAL SET MESSAGE_TEXT = 'wrap' ;          -- re-throw with new text,
                                                       -- original SQLSTATE preserved
         (Used to add context to a CAUGHT error without losing the diagnostic area.)
         Using SIGNAL inside a handler is legal but LOSES the original error's
         SQLSTATE and code unless you copy them manually.
```

## Patterns

### Pattern 1 : Procedure with IN and OUT parameters

```sql
-- 10.6+ : returns counts via OUT parameters, modifies data, INVOKER security.
DELIMITER //
CREATE OR REPLACE PROCEDURE archive_orders_before(
  IN  cutoff   DATETIME,
  OUT moved    INT,
  OUT skipped  INT
)
  MODIFIES SQL DATA
  SQL SECURITY INVOKER
  COMMENT 'Moves orders.created_at < cutoff to orders_archive'
BEGIN
  DECLARE EXIT HANDLER FOR SQLEXCEPTION
  BEGIN
    ROLLBACK;
    RESIGNAL;
  END;

  START TRANSACTION;
  INSERT INTO orders_archive SELECT * FROM orders WHERE created_at < cutoff;
  SET moved = ROW_COUNT();
  DELETE FROM orders WHERE created_at < cutoff;
  SET skipped = (SELECT COUNT(*) FROM orders WHERE created_at < cutoff);
  COMMIT;
END//
DELIMITER ;

-- Call site
CALL archive_orders_before('2025-01-01', @m, @s);
SELECT @m AS moved, @s AS skipped;
```

### Pattern 2 : Deterministic function safe in WHERE

```sql
-- 10.6+ : pure arithmetic, deterministic, indexable on the column it computes on.
DELIMITER //
CREATE OR REPLACE FUNCTION net_price(gross DECIMAL(12,2), vat_rate DECIMAL(5,4))
  RETURNS DECIMAL(12,2)
  DETERMINISTIC
  CONTAINS SQL
  SQL SECURITY INVOKER
BEGIN
  RETURN gross / (1 + vat_rate);
END//
DELIMITER ;

-- Safe in WHERE on a generated column :
ALTER TABLE invoices
  ADD COLUMN net AS (net_price(gross, vat_rate)) PERSISTENT,
  ADD INDEX ix_net (net);
SELECT id FROM invoices WHERE net > 100;
```

### Pattern 3 : Cursor loop with NOT FOUND handler

```sql
-- 10.6+ : iterate over a result set inside a procedure.
DELIMITER //
CREATE OR REPLACE PROCEDURE flag_high_value_customers()
  MODIFIES SQL DATA
BEGIN
  DECLARE v_id      BIGINT;
  DECLARE v_total   DECIMAL(15,2);
  DECLARE done      INT DEFAULT 0;
  DECLARE cur CURSOR FOR
    SELECT customer_id, SUM(total)
    FROM orders
    GROUP BY customer_id
    HAVING SUM(total) > 10000;
  DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = 1;

  OPEN cur;
  read_loop: LOOP
    FETCH cur INTO v_id, v_total;
    IF done = 1 THEN
      LEAVE read_loop;
    END IF;
    UPDATE customers SET tier = 'high' WHERE id = v_id;
  END LOOP;
  CLOSE cur;
END//
DELIMITER ;
```

### Pattern 4 : Custom error with SIGNAL plus catching with handler

```sql
-- 10.6+ : raise a domain error, caller catches via handler.
DELIMITER //
CREATE OR REPLACE PROCEDURE withdraw(IN acct INT, IN amt DECIMAL(15,2))
  MODIFIES SQL DATA
BEGIN
  DECLARE bal DECIMAL(15,2);
  START TRANSACTION;
  SELECT balance INTO bal FROM accounts WHERE id = acct FOR UPDATE;
  IF bal IS NULL THEN
    ROLLBACK;
    SIGNAL SQLSTATE '45000'
      SET MESSAGE_TEXT = 'account not found', MYSQL_ERRNO = 31001;
  END IF;
  IF bal < amt THEN
    ROLLBACK;
    SIGNAL SQLSTATE '45000'
      SET MESSAGE_TEXT = 'insufficient funds', MYSQL_ERRNO = 31002;
  END IF;
  UPDATE accounts SET balance = balance - amt WHERE id = acct;
  COMMIT;
END//
DELIMITER ;
```

### Pattern 5 : Recursive procedure with max_sp_recursion_depth

```sql
-- 10.6+ : factorial via recursion. Without the SET, second call errors out.
SET @@SESSION.max_sp_recursion_depth = 32;

DELIMITER //
CREATE OR REPLACE PROCEDURE factorial(IN n INT, INOUT result BIGINT)
  CONTAINS SQL
BEGIN
  IF n <= 1 THEN
    SET result = 1;
  ELSE
    CALL factorial(n - 1, result);
    SET result = result * n;
  END IF;
END//
DELIMITER ;

CALL factorial(10, @r); SELECT @r;   -- 3628800
```

## References

- `references/methods.md` : full grammar, all relevant SQLSTATE codes, characteristic matrix, system variables.
- `references/examples.md` : 10+ working procedure / function examples covering IN/OUT, cursors, handlers, SIGNAL, RESIGNAL, recursion, nested handlers.
- `references/anti-patterns.md` : 8 production-grade anti-patterns with why-it-fails and corrected alternative.

### Official source URLs (last verified 2026-05-19)

- `https://mariadb.com/kb/en/create-procedure/`
- `https://mariadb.com/kb/en/create-function/`
- `https://mariadb.com/kb/en/declare-handler/`
- `https://mariadb.com/kb/en/signal/`
- `https://mariadb.com/kb/en/resignal/`
- `https://mariadb.com/kb/en/declare-cursor/`
- `https://mariadb.com/kb/en/cursors/`

### Related skills

- `mariadb-syntax-triggers-events-views` : triggers, scheduled events, views ; shares DECLARE HANDLER and SIGNAL semantics.
- `mariadb-impl-transactions-and-locking` : transactional patterns invoked from procedures.
- `mariadb-errors-procedures-and-handlers` : diagnostic recipes when a routine misbehaves.
