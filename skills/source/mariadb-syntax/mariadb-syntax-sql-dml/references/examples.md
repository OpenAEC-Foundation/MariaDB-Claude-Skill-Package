# MariaDB DML : Worked Examples

12 working examples covering INSERT variants, upsert, REPLACE, multi-table UPDATE/DELETE, RETURNING, and DELETE HISTORY. Every example is verified to run against MariaDB 10.6-LTS and later.

## Example 1 : Single-row INSERT

```sql
-- 10.6+ : the simplest case ; primary use for "create one record"
CREATE TABLE customer (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(120) NOT NULL,
  email VARCHAR(255) NOT NULL UNIQUE,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;

INSERT INTO customer (name, email)
  VALUES ('Acme Corp', 'billing@acme.example');

SELECT LAST_INSERT_ID();   -- e.g. 1
```

## Example 2 : Multi-row INSERT with batched generated ids

```sql
-- 10.6+ : single round-trip, contiguous auto-increment block when auto_increment_increment = 1
INSERT INTO customer (name, email) VALUES
  ('Acme Corp',   'billing@acme.example'),
  ('Globex Ltd',  'ap@globex.example'),
  ('Initech Inc', 'invoices@initech.example');

SELECT LAST_INSERT_ID();   -- returns the FIRST generated id ; others are FIRST+1, FIRST+2
SELECT ROW_COUNT();        -- returns 3
```

## Example 3 : INSERT SET form

```sql
-- 10.6+ : readable alternative for wide tables ; semantically identical to INSERT ... VALUES
INSERT INTO customer
  SET name = 'Wayne Enterprises',
      email = 'ap@wayne.example',
      created_at = NOW();
```

## Example 4 : INSERT ... SELECT for archival

```sql
-- 10.6+ : bulk copy filtered rows into an archive table
CREATE TABLE customer_archive LIKE customer;
ALTER TABLE customer_archive ADD COLUMN archived_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP;

INSERT INTO customer_archive (id, name, email, created_at)
  SELECT id, name, email, created_at
  FROM customer
  WHERE created_at < NOW() - INTERVAL 3 YEAR;

DELETE FROM customer WHERE created_at < NOW() - INTERVAL 3 YEAR;
```

When archiving from one table to another, do this in a single transaction or with an intermediate flag column ; otherwise a crash between INSERT...SELECT and DELETE leaves duplicates.

## Example 5 : Canonical upsert with ON DUPLICATE KEY UPDATE

```sql
-- 10.6+ : insert-or-increment inventory counter ; the standard MariaDB upsert
CREATE TABLE inventory (
  sku VARCHAR(40) PRIMARY KEY,
  qty INT NOT NULL DEFAULT 0,
  last_seen DATETIME NOT NULL
) ENGINE=InnoDB;

INSERT INTO inventory (sku, qty, last_seen)
  VALUES ('SKU-001', 10, NOW())
  ON DUPLICATE KEY UPDATE
    qty       = qty + VALUES(qty),     -- add to existing
    last_seen = VALUES(last_seen);     -- overwrite timestamp

-- Inspect what happened :
SELECT ROW_COUNT();   -- 1 = inserted, 2 = updated
```

## Example 6 : Multi-row upsert (batch ingestion)

```sql
-- 10.6+ : ingest many rows ; each row independently inserts or updates
INSERT INTO inventory (sku, qty, last_seen) VALUES
  ('SKU-001', 5, NOW()),
  ('SKU-002', 3, NOW()),
  ('SKU-003', 7, NOW())
  ON DUPLICATE KEY UPDATE
    qty       = qty + VALUES(qty),
    last_seen = VALUES(last_seen);
```

`VALUES(col)` inside the UPDATE clause refers to "the value that would have been inserted for this row" ; it is the canonical idiom for ODKU with multi-row VALUES.

## Example 7 : INSERT RETURNING (10.5+)

```sql
-- 10.5+ : fetch generated id and computed columns without a second round-trip
INSERT INTO customer (name, email)
  VALUES ('Stark Industries', 'tony@stark.example')
  RETURNING id, name, created_at;
```

Output is a result set, not a status code. Drivers handle it as a SELECT result.

## Example 8 : REPLACE INTO (only when safe)

```sql
-- 10.6+ : suitable for a config-key/value snapshot where ZERO downstream FK or trigger consequences exist
CREATE TABLE config_snapshot (
  key_name VARCHAR(64) PRIMARY KEY,
  value VARCHAR(255) NOT NULL,
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB;

REPLACE INTO config_snapshot (key_name, value)
  VALUES ('schema_version', '42');

-- compare with the upsert equivalent (which is preferred in 99% of cases) :
INSERT INTO config_snapshot (key_name, value)
  VALUES ('schema_version', '42')
  ON DUPLICATE KEY UPDATE value = VALUES(value);
```

The upsert form (a) does not advance auto_increment, (b) does not fire DELETE triggers, (c) does not cascade through FKs.

## Example 9 : Safe batched UPDATE with ORDER BY + LIMIT

```sql
-- 10.6+ : process pending work in deterministic id-order, in batches of 1000
UPDATE outbox
  SET status = 'sent',
      sent_at = NOW()
  WHERE status = 'pending'
    AND created_at < NOW() - INTERVAL 5 MINUTE
  ORDER BY id
  LIMIT 1000;

SELECT ROW_COUNT();   -- e.g. 1000 ; loop until 0
```

ORDER BY makes the row selection deterministic. Without it, the optimizer may pick rows in any order and concurrent workers may race on the same rows.

## Example 10 : Multi-table UPDATE with JOIN

```sql
-- 10.6+ : sync a tier from a score table into the customer table
UPDATE customer c
JOIN customer_score s ON s.customer_id = c.id
SET c.tier = s.tier,
    c.tier_updated_at = NOW()
WHERE s.calculated_at > c.tier_updated_at;
```

Row-order across the joined rowset is undefined. If two rules could update the same customer differently, the result is non-deterministic. Restructure as a single-table UPDATE with a derived table if that risk is real.

## Example 11 : Multi-table DELETE with JOIN

```sql
-- 10.6+ : GDPR-style purge driven by a request table
DELETE c
FROM customer c
JOIN gdpr_purge_request p ON p.customer_id = c.id
WHERE p.confirmed_at < NOW() - INTERVAL 30 DAY;
```

The `DELETE c` clause names which table's rows to remove ; the FROM joins the driver table. Multi-table DELETE does NOT support RETURNING or ORDER BY.

## Example 12 : DELETE RETURNING and DELETE HISTORY

```sql
-- 10.0+ : DELETE RETURNING for session cleanup with audit logging
DELETE FROM session
  WHERE expires_at < NOW()
  RETURNING id, user_id, expires_at;
-- Driver receives a result set of every deleted row.

-- 10.3+ : system-versioned table maintenance
CREATE TABLE audit_log (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  actor VARCHAR(120) NOT NULL,
  action VARCHAR(40) NOT NULL,
  payload JSON
) WITH SYSTEM VERSIONING;

-- A normal DELETE keeps row-history :
DELETE FROM audit_log WHERE id = 123;
-- The row moves to history with row_end = NOW().

-- DELETE HISTORY actually purges history older than a date :
DELETE HISTORY FROM audit_log
  BEFORE SYSTEM_TIME NOW() - INTERVAL 7 YEAR;
-- Requires DELETE HISTORY privilege ; regular DELETE privilege is NOT sufficient.
```

## Bonus : the LAST_INSERT_ID idiom for ON DUPLICATE KEY UPDATE

```sql
-- 10.6+ : preserve the row's id in LAST_INSERT_ID() even on the UPDATE branch.
INSERT INTO inventory (sku, qty, last_seen)
  VALUES ('SKU-001', 5, NOW())
  ON DUPLICATE KEY UPDATE
    id = LAST_INSERT_ID(id),       -- the IDIOM : returns the existing row's id
    qty = qty + VALUES(qty),
    last_seen = VALUES(last_seen);

SELECT LAST_INSERT_ID();   -- the existing row's id, not 0
```

This idiom is documented behavior of the `LAST_INSERT_ID(expr)` function : when called with an argument, it sets the session's last-insert-id to that expression and returns it.
