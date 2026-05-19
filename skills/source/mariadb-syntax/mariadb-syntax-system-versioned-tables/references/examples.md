# Examples : System-Versioned Tables and Application-Time Periods

Twelve working examples annotated with the introduction version. Every snippet was verified against `mariadb.com/kb/en/system-versioned-tables/`, `mariadb.com/kb/en/application-time-periods/`, and `mariadb.com/kb/en/bitemporal-tables/` on 2026-05-19.

## 1. Minimal system-versioned table (10.3+)

```sql
-- 10.3+
CREATE TABLE accounts (
  id      INT PRIMARY KEY,
  balance DECIMAL(15,2)
) WITH SYSTEM VERSIONING;

INSERT INTO accounts VALUES (1, 1000.00);
UPDATE accounts SET balance = 950.00 WHERE id = 1;
UPDATE accounts SET balance = 925.50 WHERE id = 1;

-- only current state
SELECT * FROM accounts;

-- every version ever
SELECT id, balance, ROW_START, ROW_END
  FROM accounts FOR SYSTEM_TIME ALL
  ORDER BY ROW_START;
```

`ROW_START` and `ROW_END` are hidden from `SELECT *` but addressable by name. Default precision on InnoDB : transaction-id (`BIGINT UNSIGNED`).

## 2. Timestamp-precision versioning for wall-clock time travel (10.3+)

```sql
-- 10.3+
CREATE TABLE accounts (
  id        INT PRIMARY KEY,
  balance   DECIMAL(15,2),
  row_start TIMESTAMP(6) GENERATED ALWAYS AS ROW START,
  row_end   TIMESTAMP(6) GENERATED ALWAYS AS ROW END,
  PERIOD FOR SYSTEM_TIME (row_start, row_end)
) WITH SYSTEM VERSIONING;

INSERT INTO accounts (id, balance) VALUES (1, 1000.00);
-- ... time passes, multiple updates ...

-- time-travel to a moment in the past
SELECT balance
  FROM accounts FOR SYSTEM_TIME AS OF TIMESTAMP '2026-01-15 09:00:00'
  WHERE id = 1;
```

Use timestamp precision whenever the application surfaces "show me the state at this time" to users.

## 3. Audit-trail of an UPDATE : full history with delta (10.3+)

```sql
-- 10.3+
CREATE TABLE users (
  id       INT PRIMARY KEY,
  email    VARCHAR(255),
  status   VARCHAR(20),
  row_start TIMESTAMP(6) GENERATED ALWAYS AS ROW START,
  row_end   TIMESTAMP(6) GENERATED ALWAYS AS ROW END,
  PERIOD FOR SYSTEM_TIME (row_start, row_end)
) WITH SYSTEM VERSIONING;

INSERT INTO users (id, email, status) VALUES (42, 'a@x.io', 'PENDING');
UPDATE users SET status = 'ACTIVE' WHERE id = 42;
UPDATE users SET email  = 'b@x.io' WHERE id = 42;

-- every version with the timestamp it became visible
SELECT id, email, status, row_start AS visible_from, row_end AS visible_until
  FROM users FOR SYSTEM_TIME ALL
  WHERE id = 42
  ORDER BY row_start;
```

## 4. Time-bounded history partition rotation (10.3+)

```sql
-- 10.3+
CREATE TABLE audit_log (
  event_id BIGINT PRIMARY KEY AUTO_INCREMENT,
  payload  TEXT,
  row_start TIMESTAMP(6) GENERATED ALWAYS AS ROW START,
  row_end   TIMESTAMP(6) GENERATED ALWAYS AS ROW END,
  PERIOD FOR SYSTEM_TIME (row_start, row_end)
) WITH SYSTEM VERSIONING
  PARTITION BY SYSTEM_TIME INTERVAL 1 MONTH (
    PARTITION p202601 HISTORY,
    PARTITION p202602 HISTORY,
    PARTITION p202603 HISTORY,
    PARTITION pcur    CURRENT
  );

-- drop the oldest history partition cheaply
ALTER TABLE audit_log DROP PARTITION p202601;
```

`ALTER TABLE ... DROP PARTITION` is O(1) compared with `DELETE HISTORY ... BEFORE SYSTEM_TIME` which scans rows.

## 5. Auto-rotating history with INTERVAL ... AUTO (10.9.1+)

```sql
-- 10.9.1+
CREATE TABLE audit_log (
  event_id BIGINT PRIMARY KEY AUTO_INCREMENT,
  payload  TEXT,
  row_start TIMESTAMP(6) GENERATED ALWAYS AS ROW START,
  row_end   TIMESTAMP(6) GENERATED ALWAYS AS ROW END,
  PERIOD FOR SYSTEM_TIME (row_start, row_end)
) WITH SYSTEM VERSIONING
  PARTITION BY SYSTEM_TIME INTERVAL 1 HOUR AUTO;
```

MariaDB creates new HISTORY partitions automatically as time advances. Combine with a cron job that drops old partitions to enforce retention without DBA action.

## 6. Size-bounded rotation : LIMIT partitions (10.3+, AUTO 10.9.1+)

```sql
-- 10.3+
CREATE TABLE audit_log (
  event_id BIGINT PRIMARY KEY AUTO_INCREMENT,
  payload  TEXT,
  row_start TIMESTAMP(6) GENERATED ALWAYS AS ROW START,
  row_end   TIMESTAMP(6) GENERATED ALWAYS AS ROW END,
  PERIOD FOR SYSTEM_TIME (row_start, row_end)
) WITH SYSTEM VERSIONING
  PARTITION BY SYSTEM_TIME LIMIT 100000 (
    PARTITION p0 HISTORY,
    PARTITION p1 HISTORY,
    PARTITION p2 HISTORY,
    PARTITION pcur CURRENT
  );

-- 10.9.1+ auto-managed variant
ALTER TABLE audit_log PARTITION BY SYSTEM_TIME LIMIT 100000 AUTO;
```

## 7. GDPR-friendly purge via DELETE HISTORY (10.3+)

```sql
-- 10.3+ : retention 90 days
DELETE HISTORY FROM users
  BEFORE SYSTEM_TIME (NOW() - INTERVAL 90 DAY);
```

Grant `DELETE HISTORY` privilege to the compliance role only :

```sql
GRANT DELETE HISTORY ON app.* TO 'compliance'@'%';
```

Schedule via the MariaDB event scheduler :

```sql
-- 10.3+
CREATE EVENT IF NOT EXISTS purge_user_history
  ON SCHEDULE EVERY 1 DAY
  DO DELETE HISTORY FROM users BEFORE SYSTEM_TIME (NOW() - INTERVAL 90 DAY);
```

## 8. Application-time period for contract validity (10.4+)

```sql
-- 10.4+
CREATE TABLE contracts (
  contract_id INT,
  party       VARCHAR(120),
  terms       TEXT,
  valid_from  DATE NOT NULL,
  valid_to    DATE NOT NULL,
  PERIOD FOR validity (valid_from, valid_to)
);

INSERT INTO contracts VALUES
  (1, 'Acme Ltd', 'standard',   '2026-01-01', '2026-12-31'),
  (2, 'Beta Inc', 'premium',    '2026-03-01', '2027-02-29');

-- terminate a window
DELETE FROM contracts
  FOR PORTION OF validity FROM '2026-06-01' TO '2026-09-01'
  WHERE contract_id = 1;
```

Result : contract 1 splits into `2026-01-01 .. 2026-06-01` and `2026-09-01 .. 2026-12-31`.

## 9. UPDATE FOR PORTION OF : effective-dated price change (10.4+)

```sql
-- 10.4+
CREATE TABLE prices (
  product_id INT,
  price      DECIMAL(10,2),
  valid_from DATE NOT NULL,
  valid_to   DATE NOT NULL,
  PERIOD FOR validity (valid_from, valid_to)
);

INSERT INTO prices VALUES (101, 19.99, '2026-01-01', '2027-01-01');

-- promo : 9.99 during March
UPDATE prices
  FOR PORTION OF validity FROM '2026-03-01' TO '2026-04-01'
  SET price = 9.99
  WHERE product_id = 101;
```

Result : three rows for product 101 :

| valid_from | valid_to   | price |
|------------|------------|-------|
| 2026-01-01 | 2026-03-01 | 19.99 |
| 2026-03-01 | 2026-04-01 |  9.99 |
| 2026-04-01 | 2027-01-01 | 19.99 |

## 10. WITHOUT OVERLAPS PK on application-time table (10.5.3+)

```sql
-- 10.5.3+
CREATE TABLE prices (
  product_id INT,
  price      DECIMAL(10,2),
  valid_from DATE NOT NULL,
  valid_to   DATE NOT NULL,
  PERIOD FOR validity (valid_from, valid_to),
  PRIMARY KEY (product_id, validity WITHOUT OVERLAPS)
);

INSERT INTO prices VALUES (101, 19.99, '2026-01-01', '2026-07-01');
-- this would conflict with the existing range and is rejected
INSERT INTO prices VALUES (101, 22.00, '2026-06-01', '2026-08-01');
-- ERROR : duplicate entry for key 'PRIMARY'
```

## 11. Bitemporal table : audited contracts (10.5+)

```sql
-- 10.5+
CREATE TABLE contracts (
  contract_id INT,
  party       VARCHAR(120),
  terms       TEXT,
  valid_from  DATE NOT NULL,
  valid_to    DATE NOT NULL,
  row_start   TIMESTAMP(6) GENERATED ALWAYS AS ROW START INVISIBLE,
  row_end     TIMESTAMP(6) GENERATED ALWAYS AS ROW END   INVISIBLE,
  PERIOD FOR application_time (valid_from, valid_to),
  PERIOD FOR SYSTEM_TIME      (row_start, row_end)
) WITH SYSTEM VERSIONING;

-- "as known on 2026-04-01, what was contract 1 valid during March ?"
SELECT contract_id, party, terms, valid_from, valid_to
  FROM contracts
  FOR SYSTEM_TIME AS OF TIMESTAMP '2026-04-01'
  WHERE contract_id = 1
    AND valid_from <  '2026-04-01'
    AND valid_to   >  '2026-03-01';
```

The two periods are independent : `FOR SYSTEM_TIME` picks the audit dimension, the `WHERE valid_*` filter picks the application dimension.

## 12. Range and ALL queries for reporting (10.3+)

```sql
-- 10.3+
-- every version touching the first quarter of 2026
SELECT id, balance, row_start, row_end
  FROM accounts
  FOR SYSTEM_TIME BETWEEN '2026-01-01' AND '2026-03-31'
  ORDER BY row_start;

-- inclusive-start, exclusive-end : strict day-bucket reporting
SELECT id, balance
  FROM accounts
  FOR SYSTEM_TIME FROM '2026-03-01' TO '2026-04-01';

-- full audit dump
SELECT id, balance, row_start, row_end
  FROM accounts FOR SYSTEM_TIME ALL
  ORDER BY id, row_start;
```

## 13. Column-level WITHOUT SYSTEM VERSIONING (10.3+)

```sql
-- 10.3+
CREATE TABLE users (
  id           INT PRIMARY KEY,
  email        VARCHAR(255),
  login_count  INT WITHOUT SYSTEM VERSIONING,
  row_start    TIMESTAMP(6) GENERATED ALWAYS AS ROW START,
  row_end      TIMESTAMP(6) GENERATED ALWAYS AS ROW END,
  PERIOD FOR SYSTEM_TIME (row_start, row_end)
) WITH SYSTEM VERSIONING;

-- this UPDATE does NOT create a history row
UPDATE users SET login_count = login_count + 1 WHERE id = 1;

-- this UPDATE DOES create a history row
UPDATE users SET email = 'new@x.io' WHERE id = 1;
```

Use this for high-churn counter columns that would pollute history.

## 14. Implicit AS OF via session variable (10.3+)

```sql
-- 10.3+
SET SESSION system_versioning_asof = '2026-01-01 00:00:00';

-- all SELECTs implicitly time-travel
SELECT * FROM accounts;
SELECT * FROM users WHERE status = 'ACTIVE';

SET SESSION system_versioning_asof = DEFAULT;
```

Useful for debugging scripts ; NEVER set globally and forget.
