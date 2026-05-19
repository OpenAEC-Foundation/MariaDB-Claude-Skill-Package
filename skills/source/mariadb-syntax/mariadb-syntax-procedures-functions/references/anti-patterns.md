# Anti-Patterns : MariaDB Stored Procedures and Stored Functions

Eight production-grade anti-patterns. Each entry shows the broken code, explains why it fails, then shows the corrected alternative. All claims verified against MariaDB KB on 2026-05-19.

---

## Anti-pattern 1 : DETERMINISTIC on a function that reads NOW(), RAND(), or session variables

### Broken

```sql
-- 10.6+ : looks innocent, optimizer can hoist or cache the call.
DELIMITER //
CREATE FUNCTION effective_tax(gross DECIMAL(10,2))
  RETURNS DECIMAL(10,2)
  DETERMINISTIC                           -- WRONG : reads NOW() and @session_tax_rate
  READS SQL DATA
BEGIN
  DECLARE rate DECIMAL(5,4);
  IF NOW() < '2026-07-01' THEN
    SET rate = @session_tax_rate;
  ELSE
    SET rate = 0.21;
  END IF;
  RETURN gross * (1 + rate);
END//
DELIMITER ;
```

### Why it fails

A function is "deterministic" ONLY when it produces the same output for the same inputs across all calls. This function depends on (a) the wall clock via `NOW()`, (b) a session variable that can change mid-query. Declaring it `DETERMINISTIC` tells the optimizer it may cache the result and replicate it under `STATEMENT` binlog. Consequence : queries return stale tax across the cutoff date, and replicas diverge from the source. The KB warns "declaring as DETERMINISTIC ... may yield incorrect results" (KB `create-function/`).

### Fix

Drop `DETERMINISTIC` and pass the rate as a parameter so the function depends only on inputs.

```sql
DELIMITER //
CREATE OR REPLACE FUNCTION effective_tax(
  gross DECIMAL(10,2),
  rate  DECIMAL(5,4)
)
  RETURNS DECIMAL(10,2)
  DETERMINISTIC                           -- correct : inputs fully determine output
  NO SQL
BEGIN
  RETURN gross * (1 + rate);
END//
DELIMITER ;
```

---

## Anti-pattern 2 : NOT DETERMINISTIC function in WHERE clause on a large table

### Broken

```sql
-- 10.6+ : function is honest about being non-deterministic, but used in WHERE on 10M rows.
DELIMITER //
CREATE FUNCTION is_recent_order(p_id BIGINT)
  RETURNS TINYINT
  NOT DETERMINISTIC
  READS SQL DATA
BEGIN
  RETURN (SELECT created_at > NOW() - INTERVAL 7 DAY FROM orders WHERE id = p_id);
END//
DELIMITER ;

-- Used in WHERE :
SELECT id FROM orders WHERE is_recent_order(id) = 1;
```

### Why it fails

The optimizer cannot fold a `NOT DETERMINISTIC` function to a constant, so `is_recent_order(id)` is evaluated PER ROW. Each call opens a sub-SELECT against `orders`, producing an O(N^2)-like blowup on a 10M-row table. EXPLAIN shows `type=ALL` and no index on `created_at` is used. The function call overhead alone (parse + invoke + close per row) can multiply runtime 50x.

### Fix

Rewrite the predicate inline so the optimizer can use the `created_at` index :

```sql
SELECT id FROM orders WHERE created_at > NOW() - INTERVAL 7 DAY;
```

If you cannot inline (the logic is too complex), materialize the result into a generated column with an index, or compute it once in the application and pass a constant.

---

## Anti-pattern 3 : Cursor loop without NOT FOUND handler

### Broken

```sql
-- 10.6+ : FETCH past the last row raises '02000' with no handler ; behaviour is undefined.
DELIMITER //
CREATE PROCEDURE process_all_orders()
  MODIFIES SQL DATA
BEGIN
  DECLARE v_id BIGINT;
  DECLARE cur CURSOR FOR SELECT id FROM orders;
  -- NO handler declared

  OPEN cur;
  LOOP
    FETCH cur INTO v_id;
    UPDATE orders SET processed = 1 WHERE id = v_id;
  END LOOP;
  CLOSE cur;
END//
DELIMITER ;
```

### Why it fails

When `FETCH` runs out of rows, MariaDB raises SQLSTATE `'02000'` (NOT FOUND). Without a handler, this propagates as an uncaught exception : the procedure aborts at the end-of-data point. Worse, if any code path has an OUTER `EXIT HANDLER FOR SQLEXCEPTION`, the cursor end-of-data is silently treated as a fatal error. Either way, the loop never terminates cleanly, and rows may be processed twice if you retry.

### Fix

Declare a `CONTINUE HANDLER FOR NOT FOUND` that sets a flag, and check the flag in the loop body :

```sql
DELIMITER //
CREATE OR REPLACE PROCEDURE process_all_orders()
  MODIFIES SQL DATA
BEGIN
  DECLARE v_id  BIGINT;
  DECLARE done  INT DEFAULT 0;
  DECLARE cur CURSOR FOR SELECT id FROM orders;
  DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = 1;

  OPEN cur;
  read_loop: LOOP
    FETCH cur INTO v_id;
    IF done = 1 THEN LEAVE read_loop; END IF;
    UPDATE orders SET processed = 1 WHERE id = v_id;
  END LOOP;
  CLOSE cur;
END//
DELIMITER ;
```

---

## Anti-pattern 4 : Recursive procedure without setting max_sp_recursion_depth

### Broken

```sql
-- 10.6+ : recursive procedure created, but never callable.
DELIMITER //
CREATE PROCEDURE walk_tree(IN p_node INT)
  READS SQL DATA
BEGIN
  DECLARE v_child INT;
  DECLARE done INT DEFAULT 0;
  DECLARE cur CURSOR FOR SELECT id FROM tree WHERE parent_id = p_node;
  DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = 1;

  OPEN cur;
  l: LOOP
    FETCH cur INTO v_child;
    IF done THEN LEAVE l; END IF;
    CALL walk_tree(v_child);              -- recurses
  END LOOP;
  CLOSE cur;
END//
DELIMITER ;

CALL walk_tree(1);
-- ERROR 1456 (HY000): Recursive limit 0 (as set by the max_sp_recursion_depth variable)
-- was exceeded for routine walk_tree
```

### Why it fails

`max_sp_recursion_depth` defaults to `0`, which forbids ALL recursion. The first recursive `CALL` raises error 1456 immediately. The procedure was syntactically valid, so the bug is invisible until the first real call. Setting the variable globally and forgetting on a fresh server is a classic deployment trap.

### Fix

Raise `max_sp_recursion_depth` per-session before the recursive call, or set it globally in `[mysqld]` with a sane upper bound (max is 255) :

```sql
SET @@SESSION.max_sp_recursion_depth = 64;
CALL walk_tree(1);
```

Document the requirement at the top of the procedure (`COMMENT 'requires max_sp_recursion_depth >= 64'`).

---

## Anti-pattern 5 : DELIMITER not reset, breaking subsequent statements

### Broken

```sql
-- Inside the mariadb CLI :
DELIMITER //
CREATE PROCEDURE p() BEGIN SELECT 1 ; END//
-- (forgot to reset)

SELECT NOW();          -- silently buffered, never executes
SHOW TABLES;           -- silently buffered, never executes
```

### Why it fails

`DELIMITER` is a CLIENT-SIDE directive (not SQL). After setting it to `//`, the CLI buffers everything up to the next `//` before sending. The user types subsequent statements ending in `;`, which the client just appends to the buffer. Hours of debugging follow when nothing appears to execute. This pattern is reported across many GitHub issues and Stack Overflow questions.

### Fix

Always reset `DELIMITER ;` immediately after the routine creation :

```sql
DELIMITER //
CREATE PROCEDURE p() BEGIN SELECT 1 ; END//
DELIMITER ;

SELECT NOW();          -- works
```

For automated deployment scripts that send one statement per call (PDO, JDBC, mariadb-connector-c with `prepare`), the DELIMITER directive is NOT required because the client never has to find a statement boundary inside the procedure body.

---

## Anti-pattern 6 : SQL SECURITY DEFINER without considering privilege escalation

### Broken

```sql
-- 10.6+ : created by root, callable by any low-privileged user.
DELIMITER //
CREATE DEFINER='root'@'localhost' PROCEDURE purge_old_audits(IN p_days INT)
  MODIFIES SQL DATA
  SQL SECURITY DEFINER          -- runs as root regardless of caller
BEGIN
  DELETE FROM audit_log WHERE created_at < NOW() - INTERVAL p_days DAY;
END//
DELIMITER ;

GRANT EXECUTE ON PROCEDURE purge_old_audits TO 'reporting'@'%';
```

### Why it fails

`SQL SECURITY DEFINER` (the silent default) runs the body with the DEFINER's privileges. The `reporting` user has only `SELECT` privileges normally, but via `CALL purge_old_audits(0)` they can wipe the entire audit table because the procedure runs as root. Any procedure that accepts user-controlled parameters and does DML against sensitive tables is a privilege-escalation vector unless explicitly designed to be one.

### Fix

Use `SQL SECURITY INVOKER` unless you have a deliberate reason to elevate privileges. Validate parameters inside the routine :

```sql
DELIMITER //
CREATE OR REPLACE DEFINER='root'@'localhost' PROCEDURE purge_old_audits(IN p_days INT)
  MODIFIES SQL DATA
  SQL SECURITY INVOKER          -- caller must have DELETE on audit_log
BEGIN
  IF p_days IS NULL OR p_days < 30 THEN
    SIGNAL SQLSTATE '45000'
      SET MESSAGE_TEXT = 'p_days must be at least 30';
  END IF;
  DELETE FROM audit_log WHERE created_at < NOW() - INTERVAL p_days DAY;
END//
DELIMITER ;
```

If the procedure MUST run as DEFINER, audit `INFORMATION_SCHEMA.ROUTINES.DEFINER` for unexpected accounts after each deployment.

---

## Anti-pattern 7 : SIGNAL without SET MESSAGE_TEXT

### Broken

```sql
-- 10.6+ : raises a custom error with no message.
DELIMITER //
CREATE PROCEDURE check_age(IN p_age INT)
  CONTAINS SQL
BEGIN
  IF p_age < 18 THEN
    SIGNAL SQLSTATE '45000';        -- no MESSAGE_TEXT, no MYSQL_ERRNO
  END IF;
END//
DELIMITER ;

CALL check_age(15);
-- ERROR 1644 (45000): Unhandled user-defined exception condition
```

### Why it fails

Without `SET MESSAGE_TEXT`, MariaDB substitutes the generic placeholder "Unhandled user-defined exception condition". The caller receives no diagnostic information. In a multi-procedure stack, the support team cannot tell which check failed without grepping every `SIGNAL` site. Logs are useless. The error becomes "something rejected the request" rather than "user must be at least 18".

### Fix

ALWAYS set `MESSAGE_TEXT`. Also set `MYSQL_ERRNO` to a stable numeric code in a documented range (e.g. 31000 through 31999 for application errors) so application code can dispatch on the code, not the message string.

```sql
DELIMITER //
CREATE OR REPLACE PROCEDURE check_age(IN p_age INT)
  CONTAINS SQL
BEGIN
  IF p_age < 18 THEN
    SIGNAL SQLSTATE '45000'
      SET MESSAGE_TEXT = 'user must be at least 18',
          MYSQL_ERRNO  = 31002;
  END IF;
END//
DELIMITER ;
```

---

## Anti-pattern 8 : Heavy SQL inside a function called per row (N+1)

### Broken

```sql
-- 10.6+ : function does a sub-SELECT per row.
DELIMITER //
CREATE FUNCTION customer_order_count(p_customer BIGINT)
  RETURNS INT
  NOT DETERMINISTIC
  READS SQL DATA
BEGIN
  RETURN (SELECT COUNT(*) FROM orders WHERE customer_id = p_customer);
END//
DELIMITER ;

-- Used in SELECT projection :
SELECT id, name, customer_order_count(id) AS orders
FROM customers;                  -- 100k customers -> 100k sub-SELECTs
```

### Why it fails

Each row invokes the function, each invocation runs a fresh `SELECT COUNT(*)` against `orders`. With 100k customers and `orders` containing 10M rows, you get 100k aggregate scans regardless of any index. The classic ORM "N+1" problem moved into SQL. Query that should be a single JOIN-GROUP BY takes minutes instead of milliseconds.

### Fix

Use a single set-based query with `JOIN` and `GROUP BY` :

```sql
SELECT c.id, c.name, COALESCE(o.cnt, 0) AS orders
FROM   customers c
LEFT JOIN (
  SELECT customer_id, COUNT(*) AS cnt
  FROM   orders
  GROUP BY customer_id
) o ON o.customer_id = c.id;
```

If the function must exist for ad-hoc tooling, mark it clearly with `COMMENT 'do not use in WHERE or projection over many rows'` and prefer materialized aggregates in a separate table maintained by triggers or scheduled events.

---

## Cross-reference

- For trigger-specific limitations (no transaction control, no result-sets) see sibling skill `mariadb-syntax-triggers-events-views`.
- For end-of-cursor handler patterns also see the cursor section in `mariadb-impl-batch-processing`.
- For the optimizer behaviour around non-deterministic functions in WHERE see `mariadb-impl-query-optimization`.
