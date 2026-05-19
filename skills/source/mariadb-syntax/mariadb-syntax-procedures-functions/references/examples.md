# Examples : MariaDB Stored Procedures and Stored Functions

Twelve runnable examples covering parameter modes, control flow, cursors, error handling, custom errors, recursion, and nested handlers. Every example carries a version annotation. All examples assume `DELIMITER //` is set before each `CREATE` and reset after.

---

## Example 1 : Simple procedure with IN and OUT parameters

```sql
-- 10.6+ : compute order total in OUT parameter.
DELIMITER //
CREATE OR REPLACE PROCEDURE get_order_total(
  IN  p_order_id BIGINT,
  OUT p_total    DECIMAL(15,2)
)
  READS SQL DATA
  SQL SECURITY INVOKER
BEGIN
  SELECT COALESCE(SUM(qty * unit_price), 0)
  INTO   p_total
  FROM   order_items
  WHERE  order_id = p_order_id;
END//
DELIMITER ;

CALL get_order_total(42, @t);
SELECT @t;
```

---

## Example 2 : INOUT parameter accumulator

```sql
-- 10.6+ : INOUT parameter modified across multiple calls.
DELIMITER //
CREATE OR REPLACE PROCEDURE accumulate(
  IN    p_x INT,
  INOUT p_acc INT
)
  CONTAINS SQL
BEGIN
  SET p_acc = COALESCE(p_acc, 0) + p_x;
END//
DELIMITER ;

SET @sum = 0;
CALL accumulate(10, @sum);
CALL accumulate(20, @sum);
CALL accumulate(30, @sum);
SELECT @sum;        -- 60
```

---

## Example 3 : Deterministic function for use in WHERE

```sql
-- 10.6+ : pure arithmetic, deterministic, safe in WHERE on indexed virtual column.
DELIMITER //
CREATE OR REPLACE FUNCTION discounted_price(
  base DECIMAL(12,2),
  pct  DECIMAL(5,4)
)
  RETURNS DECIMAL(12,2)
  DETERMINISTIC
  CONTAINS SQL
BEGIN
  RETURN base * (1 - pct);
END//
DELIMITER ;

-- Used safely in indexed VIRTUAL column :
ALTER TABLE products
  ADD COLUMN sale_price AS (discounted_price(price, discount)) PERSISTENT,
  ADD INDEX ix_sale_price (sale_price);

-- Query uses ix_sale_price :
SELECT id FROM products WHERE sale_price < 50;
```

---

## Example 4 : Cursor loop over aggregated result with NOT FOUND handler

```sql
-- 10.6+ : iterate customers, flag those above threshold.
DELIMITER //
CREATE OR REPLACE PROCEDURE tag_loyal_customers(IN threshold DECIMAL(15,2))
  MODIFIES SQL DATA
BEGIN
  DECLARE v_id     BIGINT;
  DECLARE v_total  DECIMAL(15,2);
  DECLARE done     INT DEFAULT 0;

  DECLARE cur CURSOR FOR
    SELECT customer_id, SUM(total)
    FROM   orders
    GROUP BY customer_id
    HAVING SUM(total) >= threshold;

  DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = 1;

  OPEN cur;
  row_loop: LOOP
    FETCH cur INTO v_id, v_total;
    IF done = 1 THEN LEAVE row_loop; END IF;
    UPDATE customers
       SET tags = JSON_ARRAY_APPEND(COALESCE(tags, JSON_ARRAY()), '$', 'loyal')
     WHERE id = v_id;
  END LOOP;
  CLOSE cur;
END//
DELIMITER ;
```

---

## Example 5 : SIGNAL for custom application error

```sql
-- 10.6+ : raise SQLSTATE '45000' with MYSQL_ERRNO 31002.
DELIMITER //
CREATE OR REPLACE PROCEDURE create_user(IN p_email VARCHAR(255), IN p_age INT)
  MODIFIES SQL DATA
BEGIN
  IF p_age < 18 THEN
    SIGNAL SQLSTATE '45000'
      SET MESSAGE_TEXT = 'user must be at least 18',
          MYSQL_ERRNO  = 31002;
  END IF;
  IF p_email NOT LIKE '%@%.%' THEN
    SIGNAL SQLSTATE '45000'
      SET MESSAGE_TEXT = 'invalid email format',
          MYSQL_ERRNO  = 31003;
  END IF;
  INSERT INTO users (email, age, created_at)
  VALUES (p_email, p_age, NOW(6));
END//
DELIMITER ;

-- Caller sees :
-- ERROR 31002 (45000): user must be at least 18
CALL create_user('alice@example.com', 15);
```

---

## Example 6 : EXIT handler with RESIGNAL preserving original error

```sql
-- 10.6+ : rollback + re-throw on any SQLEXCEPTION.
DELIMITER //
CREATE OR REPLACE PROCEDURE transfer(
  IN p_src INT, IN p_dst INT, IN p_amt DECIMAL(15,2)
)
  MODIFIES SQL DATA
BEGIN
  DECLARE EXIT HANDLER FOR SQLEXCEPTION
  BEGIN
    ROLLBACK;
    RESIGNAL;          -- re-throw original error untouched
  END;

  START TRANSACTION;
    UPDATE accounts SET balance = balance - p_amt WHERE id = p_src;
    UPDATE accounts SET balance = balance + p_amt WHERE id = p_dst;
  COMMIT;
END//
DELIMITER ;
```

---

## Example 7 : CONTINUE handler that logs warnings and proceeds

```sql
-- 10.6+ : count duplicate-key warnings during bulk-import.
DELIMITER //
CREATE OR REPLACE PROCEDURE import_emails(IN p_count INT)
  MODIFIES SQL DATA
BEGIN
  DECLARE i INT DEFAULT 0;
  DECLARE dup_count INT DEFAULT 0;

  -- Catch SQLSTATE '23000' (integrity violation, e.g. duplicate key) and CONTINUE.
  DECLARE CONTINUE HANDLER FOR SQLSTATE '23000'
    SET dup_count = dup_count + 1;

  WHILE i < p_count DO
    INSERT IGNORE INTO email_log (id, email)
    VALUES (i, CONCAT('user', i, '@example.com'));
    SET i = i + 1;
  END WHILE;

  INSERT INTO import_runs (run_at, duplicates) VALUES (NOW(6), dup_count);
END//
DELIMITER ;
```

---

## Example 8 : Recursive procedure with max_sp_recursion_depth

```sql
-- 10.6+ : recursive factorial. Default max_sp_recursion_depth is 0 ; must raise.
SET @@SESSION.max_sp_recursion_depth = 64;

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

---

## Example 9 : Nested handler (inner block overrides outer)

```sql
-- 10.6+ : outer handler catches everything ; inner block has a specific catch.
DELIMITER //
CREATE OR REPLACE PROCEDURE robust_insert(IN p_id INT, IN p_name VARCHAR(100))
  MODIFIES SQL DATA
BEGIN
  -- Outer catch-all
  DECLARE EXIT HANDLER FOR SQLEXCEPTION
  BEGIN
    INSERT INTO error_log (occurred_at, context)
    VALUES (NOW(6), CONCAT('robust_insert id=', p_id));
    RESIGNAL;
  END;

  BEGIN
    -- Inner specific handler : duplicate key is non-fatal.
    DECLARE CONTINUE HANDLER FOR SQLSTATE '23000' BEGIN END;
    INSERT INTO names (id, name) VALUES (p_id, p_name);
  END;

  -- Any OTHER error here is caught by the outer EXIT handler.
  UPDATE names SET name = UPPER(name) WHERE id = p_id;
END//
DELIMITER ;
```

---

## Example 10 : Multi-statement WHILE loop with LEAVE on a condition

```sql
-- 10.6+ : retry a row-update up to N times on deadlock (SQLSTATE '40001').
DELIMITER //
CREATE OR REPLACE PROCEDURE update_with_retry(
  IN p_id INT, IN p_new_status VARCHAR(20)
)
  MODIFIES SQL DATA
BEGIN
  DECLARE attempt INT DEFAULT 0;
  DECLARE got_deadlock INT DEFAULT 0;
  DECLARE done INT DEFAULT 0;

  retry_loop: WHILE attempt < 5 AND done = 0 DO
    SET got_deadlock = 0;

    BEGIN
      DECLARE CONTINUE HANDLER FOR SQLSTATE '40001'
        SET got_deadlock = 1;

      START TRANSACTION;
        UPDATE orders SET status = p_new_status WHERE id = p_id;
      COMMIT;

      IF got_deadlock = 0 THEN
        SET done = 1;
      ELSE
        ROLLBACK;
        SET attempt = attempt + 1;
        DO SLEEP(0.05 * attempt);
      END IF;
    END;
  END WHILE retry_loop;

  IF done = 0 THEN
    SIGNAL SQLSTATE '45000'
      SET MESSAGE_TEXT = 'update_with_retry exhausted 5 attempts',
          MYSQL_ERRNO  = 31010;
  END IF;
END//
DELIMITER ;
```

---

## Example 11 : Searched CASE with mixed conditions

```sql
-- 10.6+ : classify orders into tiers.
DELIMITER //
CREATE OR REPLACE FUNCTION order_tier(p_total DECIMAL(12,2))
  RETURNS VARCHAR(10)
  DETERMINISTIC
  NO SQL
BEGIN
  DECLARE v_tier VARCHAR(10);
  CASE
    WHEN p_total IS NULL OR p_total < 0 THEN
      SET v_tier = 'invalid';
    WHEN p_total < 50 THEN
      SET v_tier = 'small';
    WHEN p_total < 500 THEN
      SET v_tier = 'medium';
    WHEN p_total < 5000 THEN
      SET v_tier = 'large';
    ELSE
      SET v_tier = 'enterprise';
  END CASE;
  RETURN v_tier;
END//
DELIMITER ;

SELECT order_tier(125.50);   -- 'medium'
```

---

## Example 12 : RESIGNAL with overridden MESSAGE_TEXT, preserving SQLSTATE

```sql
-- 10.6+ : wrap a low-level error with a domain-specific message.
DELIMITER //
CREATE OR REPLACE PROCEDURE register_customer(
  IN p_email VARCHAR(255), IN p_name VARCHAR(100)
)
  MODIFIES SQL DATA
BEGIN
  DECLARE EXIT HANDLER FOR SQLSTATE '23000'
    RESIGNAL SET MESSAGE_TEXT =
      'a customer with this email already exists',
      MYSQL_ERRNO = 31020;
      -- SQLSTATE '23000' is preserved ; caller still sees '23000'.

  INSERT INTO customers (email, name, created_at)
  VALUES (p_email, p_name, NOW(6));
END//
DELIMITER ;

-- Caller sees :
-- ERROR 31020 (23000): a customer with this email already exists
CALL register_customer('alice@example.com', 'Alice');
CALL register_customer('alice@example.com', 'Alice');   -- second call fails
```

---

## Notes on testing these examples

- ALWAYS run `DELIMITER //` before and `DELIMITER ;` after every `CREATE` block in the `mariadb` CLI. Drivers that send one statement per call do not need this.
- ALWAYS prefer `CREATE OR REPLACE` over `DROP + CREATE` so that existing grants are preserved.
- Verify recursion examples actually recurse : `SHOW VARIABLES LIKE 'max_sp_recursion_depth' ;` must show a value greater than 0 in the SESSION scope.
- Inspect routine definition with `SHOW CREATE PROCEDURE sp_name \G` (vertical layout, easier to read).
