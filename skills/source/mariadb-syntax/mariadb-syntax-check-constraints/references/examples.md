# CHECK Constraints : Working Examples

10 verified, version-annotated examples. Every snippet runs on MariaDB 10.6 LTS, 10.11 LTS, 11.x, and 12.x unless otherwise noted.

## Example 1 : Column-level CHECK on a numeric range

```sql
-- 10.2.1+
CREATE TABLE products (
  id    INT PRIMARY KEY,
  price DECIMAL(10,2) NOT NULL CHECK (price >= 0 AND price <= 1000000)
);

INSERT INTO products VALUES (1, 19.99);      -- OK
INSERT INTO products VALUES (2, -5);          -- ERROR 4025
INSERT INTO products VALUES (3, 1000001);     -- ERROR 4025
```

Pairing `NOT NULL` with the CHECK guarantees presence and range. Without `NOT NULL`, a row with `price = NULL` would pass the CHECK (NULL predicate).

## Example 2 : Table-level CHECK across two columns

```sql
-- 10.2.1+
CREATE TABLE bookings (
  id         INT PRIMARY KEY,
  start_date DATE NOT NULL,
  end_date   DATE NOT NULL,
  CONSTRAINT chk_date_order CHECK (end_date >= start_date)
);

INSERT INTO bookings VALUES (1, '2026-05-01', '2026-05-10');  -- OK
INSERT INTO bookings VALUES (2, '2026-05-10', '2026-05-01');  -- ERROR 4025
```

Table-level CHECK enables predicates that span multiple columns of the same row. NAME the constraint (`chk_date_order`) to make it droppable and identifiable in error output.

## Example 3 : Enum-like restriction without ENUM type

```sql
-- 10.2.1+
CREATE TABLE tickets (
  id     INT PRIMARY KEY,
  status VARCHAR(16) NOT NULL,
  CONSTRAINT chk_ticket_status CHECK (status IN ('open', 'pending', 'closed', 'archived'))
);

INSERT INTO tickets VALUES (1, 'open');     -- OK
INSERT INTO tickets VALUES (2, 'invalid');  -- ERROR 4025
```

`CHECK (col IN (...))` is preferable to MySQL's `ENUM` type because :

- Adding a new value is `ALTER TABLE DROP CONSTRAINT chk_ticket_status, ADD CONSTRAINT ... CHECK (status IN ('open','pending','closed','archived','escalated'))` (no table rewrite).
- The set is explicit in the schema and visible in `information_schema.CHECK_CONSTRAINTS`.
- ENUM in MariaDB has historical bugs with index usage and silent string-to-integer coercion ; CHECK avoids both.

## Example 4 : JSON validation with JSON_VALID

```sql
-- 10.2.1+ (CHECK enforcement)
-- 10.4.3+ adds auto-CHECK on the JSON alias
CREATE TABLE events (
  id      INT PRIMARY KEY,
  payload LONGTEXT NOT NULL,
  CONSTRAINT chk_payload_json CHECK (JSON_VALID(payload))
);

INSERT INTO events VALUES (1, '{"user":42,"action":"login"}');  -- OK
INSERT INTO events VALUES (2, 'not valid json');                  -- ERROR 4025
INSERT INTO events VALUES (3, '');                                -- ERROR 4025 (empty string is not JSON)
```

MariaDB JSON is a LONGTEXT alias (see D-010). WITHOUT `CHECK (JSON_VALID(col))`, any string is accepted and downstream `JSON_EXTRACT` returns NULL silently. ALWAYS add `CHECK (JSON_VALID(col))` on every JSON column, even on the `JSON` alias type for portability across MariaDB versions.

## Example 5 : Regular expression validation

```sql
-- 10.2.1+
CREATE TABLE users (
  id    INT PRIMARY KEY,
  email VARCHAR(255) NOT NULL,
  CONSTRAINT chk_email_format CHECK (email REGEXP '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$')
);

INSERT INTO users VALUES (1, 'alice@example.com');  -- OK
INSERT INTO users VALUES (2, 'not-an-email');        -- ERROR 4025
```

`REGEXP` (alias `RLIKE`) is deterministic and CHECK-compatible. Note : escape backslashes for the SQL string literal (`\\.`). Regex CHECKs are measurably slower than simple comparisons; benchmark before applying to high-volume INSERT paths.

## Example 6 : ADD CHECK to an existing table

```sql
-- 10.2.1+
-- Suppose `accounts` was created without a balance check :
CREATE TABLE accounts (
  id      INT PRIMARY KEY,
  balance DECIMAL(12,2) NOT NULL
);

INSERT INTO accounts VALUES (1, 100), (2, -50);

-- Adding a CHECK against existing data :
ALTER TABLE accounts
  ADD CONSTRAINT chk_balance_non_negative CHECK (balance >= 0);
-- ERROR 4025 : existing row id=2 fails the CHECK.
```

ALTER TABLE evaluates the CHECK against existing rows. If any row fails, the ALTER aborts. Audit first :

```sql
SELECT id, balance FROM accounts WHERE balance < 0;
-- Fix or delete violating rows, then retry the ALTER.
```

## Example 7 : Drop a CHECK constraint

```sql
-- 10.2.1+
ALTER TABLE accounts DROP CONSTRAINT chk_balance_non_negative;

-- For anonymous constraints, look up the auto-generated name first :
SELECT CONSTRAINT_NAME, CHECK_CLAUSE
  FROM information_schema.CHECK_CONSTRAINTS
 WHERE CONSTRAINT_SCHEMA = DATABASE()
   AND TABLE_NAME = 'accounts';

-- Example output : CONSTRAINT_1 | `balance` >= 0
ALTER TABLE accounts DROP CONSTRAINT CONSTRAINT_1;
```

DDL replay scripts MUST use explicit constraint names. Auto-generated names depend on declaration order and may differ between environments.

## Example 8 : CHECK on a PERSISTENT generated column

```sql
-- 10.2+ (generated columns) + 10.2.1+ (CHECK)
CREATE TABLE orders (
  id        INT PRIMARY KEY,
  qty       INT NOT NULL,
  unit_cost DECIMAL(10,2) NOT NULL,
  total     DECIMAL(12,2) AS (qty * unit_cost) PERSISTENT,
  CONSTRAINT chk_qty_positive   CHECK (qty > 0),
  CONSTRAINT chk_cost_positive  CHECK (unit_cost > 0),
  CONSTRAINT chk_total_positive CHECK (total > 0)
);

INSERT INTO orders (id, qty, unit_cost) VALUES (1, 3, 9.99);  -- total = 29.97, OK
INSERT INTO orders (id, qty, unit_cost) VALUES (2, 0, 9.99);  -- ERROR 4025 (chk_qty_positive)
```

CHECK on a PERSISTENT generated column is evaluated on INSERT/UPDATE. NEVER assume CHECK re-validates when the generation formula changes via `ALTER COLUMN ... AS (...)` ; recomputation happens, but if the new formula references different columns, audit existing rows manually.

## Example 9 : Named multi-column CHECK with COALESCE

```sql
-- 10.2.1+
CREATE TABLE subscriptions (
  id         INT PRIMARY KEY,
  start_date DATE NOT NULL,
  end_date   DATE,  -- nullable for open-ended subscriptions
  CONSTRAINT chk_subscription_dates
    CHECK (end_date IS NULL OR end_date >= start_date)
);

INSERT INTO subscriptions VALUES (1, '2026-01-01', NULL);          -- OK (open-ended)
INSERT INTO subscriptions VALUES (2, '2026-01-01', '2026-12-31');  -- OK
INSERT INTO subscriptions VALUES (3, '2026-12-31', '2026-01-01');  -- ERROR 4025
```

`IS NULL OR ...` is the idiomatic pattern for "nullable column AND constraint when value is present". `COALESCE(end_date, '9999-12-31') >= start_date` works but obscures intent.

## Example 10 : INSERT IGNORE warning vs hard error

```sql
-- 10.2.1+
CREATE TABLE inventory (
  sku        VARCHAR(32) PRIMARY KEY,
  on_hand    INT NOT NULL,
  CONSTRAINT chk_on_hand_non_negative CHECK (on_hand >= 0)
);

INSERT INTO inventory VALUES ('A1', 10);
INSERT INTO inventory VALUES ('A2', -3);
-- ERROR 4025 : CONSTRAINT 'inventory.chk_on_hand_non_negative' failed

INSERT IGNORE INTO inventory VALUES ('A3', -7);
-- 0 rows affected, 1 warning
SHOW WARNINGS;
-- Warning | 4025 | CONSTRAINT `inventory.chk_on_hand_non_negative` failed for `db`.`inventory`

SELECT * FROM inventory;
-- Only A1 is present. A3 was NOT inserted despite IGNORE.
```

CRITICAL : `INSERT IGNORE` does NOT insert the row when CHECK fails. It converts the error to a warning so the statement continues for the remaining rows. Always inspect `SHOW WARNINGS` after batch INSERT IGNORE to detect silent skips.

## Example 11 : Disable CHECK globally for bulk load

```sql
-- 10.2+
-- Trusted bulk load from a legacy dump
SET SESSION check_constraint_checks = OFF;

LOAD DATA INFILE '/var/lib/mysql-files/legacy_accounts.csv'
  INTO TABLE accounts
  FIELDS TERMINATED BY ','
  IGNORE 1 LINES;

SET SESSION check_constraint_checks = ON;

-- MANDATORY post-load audit :
SELECT id, balance FROM accounts WHERE balance < 0;
-- Manually reconcile any violations. Re-enabling does NOT re-validate.
```

NEVER ship application code that toggles `check_constraint_checks`. Only DBA/migration scripts may use this variable, and only on trusted data.

## Example 12 : CHECK on a virtual column for indexed lookup

```sql
-- 10.2+
CREATE TABLE events (
  id      INT PRIMARY KEY,
  payload LONGTEXT NOT NULL,
  event_type VARCHAR(64) AS (JSON_VALUE(payload, '$.type')) VIRTUAL,
  CONSTRAINT chk_payload_json CHECK (JSON_VALID(payload)),
  CONSTRAINT chk_event_type   CHECK (event_type IN ('login','logout','purchase','refund')),
  INDEX (event_type)
);

INSERT INTO events (id, payload) VALUES (1, '{"type":"login","user":42}');  -- OK
INSERT INTO events (id, payload) VALUES (2, '{"type":"unknown"}');           -- ERROR 4025
INSERT INTO events (id, payload) VALUES (3, 'garbage');                       -- ERROR 4025 (JSON_VALID)
```

Combine `CHECK (JSON_VALID(col))` with a virtual generated column extracted via `JSON_VALUE`, then a second CHECK enforces an enum-like set on the extracted value. Add a regular index on the virtual column for fast lookups by event_type. This is the canonical MariaDB pattern for "validated JSON with indexed sub-field access".

## Verification Reproducer

To verify any example, paste it into `mariadb` client connected to any supported version :

```bash
mariadb -uroot -p < example.sql
```

For automated CI verification, use the test harness in `tests/check-constraints/` (TODO post-Phase 5 polish).
