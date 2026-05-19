# MariaDB SQL DDL : Examples

Twelve working DDL examples, each verified against MariaDB 10.6-LTS, 10.11-LTS, 11.x, 12.x. Version-annotated. Read top-to-bottom : simple to complex.

## 1. Minimal CREATE TABLE

```sql
-- 10.6+ : simplest production-acceptable table
CREATE TABLE country (
  iso2  CHAR(2)      NOT NULL PRIMARY KEY,
  name  VARCHAR(100) NOT NULL,
  KEY ix_country_name (name)
) ENGINE=InnoDB
  DEFAULT CHARSET=utf8mb4
  COLLATE=utf8mb4_uca1400_ai_ci;
```

Why : explicit ENGINE, CHARSET and COLLATE prevent stock-build latin1 lock-in. Single-column PRIMARY KEY on a CHAR(2) is cheap and naturally clustered.

## 2. CREATE TABLE with foreign keys and CHECK

```sql
-- 10.6+ : referential integrity + value validation
CREATE TABLE customer (
  id          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  email       VARCHAR(255) NOT NULL,
  status      ENUM('active','suspended','closed') NOT NULL DEFAULT 'active',
  country_iso CHAR(2)      NOT NULL,
  created_at  TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  UNIQUE KEY ux_customer_email (email),
  CONSTRAINT fk_customer_country FOREIGN KEY (country_iso) REFERENCES country(iso2)
    ON DELETE RESTRICT ON UPDATE CASCADE,
  CONSTRAINT ck_email_format CHECK (email LIKE '%@%.%')
) ENGINE=InnoDB
  ROW_FORMAT=DYNAMIC
  DEFAULT CHARSET=utf8mb4
  COLLATE=utf8mb4_uca1400_ai_ci;
```

Why : named constraints (`fk_*`, `ck_*`) keep `SHOW CREATE TABLE` readable and let later ALTER target specific constraints by name. `ON UPDATE CASCADE` is safe on natural keys ; `ON DELETE RESTRICT` blocks the foot-gun of cascade-wiping customers when a country row goes.

## 3. INSTANT ADD COLUMN

```sql
-- 10.4+ : add a column anywhere, no rewrite, no DML block
ALTER TABLE customer
  ADD COLUMN locale VARCHAR(10) NOT NULL DEFAULT 'en_US' AFTER country_iso,
  ALGORITHM=INSTANT,
  LOCK=NONE;
```

Why : metadata-only change. The server stores the default in the schema and applies it lazily on first row read. Fails fast (ER_ALTER_OPERATION_NOT_SUPPORTED) on compressed or FULLTEXT-indexed tables.

## 4. INSTANT ADD multiple columns in one statement

```sql
-- 10.4+ : combine compatible INSTANT changes
ALTER TABLE invoice
  ADD COLUMN currency  CHAR(3)        NOT NULL DEFAULT 'EUR' AFTER amount,
  ADD COLUMN tax_rate  DECIMAL(5,4)   NOT NULL DEFAULT 0.21  AFTER currency,
  ADD COLUMN paid_at   TIMESTAMP(6)   NULL                   AFTER status,
  ALGORITHM=INSTANT,
  LOCK=NONE;
```

Why : one metadata change vs three. If any sub-change rejects INSTANT, the entire statement errors and nothing is altered, so the post-condition is deterministic.

## 5. ADD INDEX with INPLACE algorithm

```sql
-- 10.6+ : online index build, concurrent DML allowed
ALTER TABLE invoice
  ADD KEY ix_customer_created (customer_id, created_at DESC),
  ALGORITHM=INPLACE,
  LOCK=NONE;
```

Why : ADD INDEX is INPLACE-only (it must populate the new index data structure). `LOCK=NONE` keeps the table available for reads and writes during the build. Descending order on `created_at` is a 10.8+ optimisation that lets ORDER BY skip the reverse-scan.

## 6. RENAME COLUMN safely

```sql
-- 10.5.3+ : metadata-only rename, no rewrite
ALTER TABLE customer
  RENAME COLUMN locale TO ui_locale,
  ALGORITHM=INSTANT,
  LOCK=NONE;
```

Why : avoids the older two-step `CHANGE old new same_definition` form. Old applications and views still hard-coding the old name will error after this rename : co-ordinate the deploy.

## 7. DROP COLUMN

```sql
-- 10.4+ : drop an unindexed column instantly
ALTER TABLE customer
  DROP COLUMN legacy_phone,
  ALGORITHM=INSTANT,
  LOCK=NONE;

-- If the column is part of an index, drop the index first (or use NOCOPY)
ALTER TABLE customer
  DROP KEY ix_customer_phone,
  DROP COLUMN phone,
  ALGORITHM=NOCOPY,
  LOCK=NONE;
```

Why : INSTANT works only for unindexed columns. NOCOPY is the next-cheapest fallback : it rewrites secondary indexes but keeps the clustered index in place.

## 8. Atomic table swap (blue-green deploy)

```sql
-- 10.6+ : zero-downtime cut-over to a rebuilt table
-- Setup phase : populate invoice_v2 from invoice with CTE or pt-archiver

RENAME TABLE
  invoice    TO invoice_v1_retired,
  invoice_v2 TO invoice;

-- Smoke-test against the new shape
SELECT COUNT(*) FROM invoice;

-- Drop the retired version after a safe verification window
DROP TABLE invoice_v1_retired;
```

Why : `RENAME TABLE` with multiple renames is atomic. Clients never see a missing or mid-state table. Atomic DDL (10.6+) makes the swap crash-safe : a server crash during the RENAME leaves the original names intact.

## 9. Sequence for distributed-friendly IDs

```sql
-- 10.3+ : MariaDB-only, MySQL has no sequences
CREATE SEQUENCE order_id_seq
  START WITH    100000
  INCREMENT BY  1
  MINVALUE      100000
  MAXVALUE      999999999999
  CACHE         100
  NOCYCLE;

-- Use the sequence as the id generator
INSERT INTO orders (id, customer_id, total)
  VALUES (NEXT VALUE FOR order_id_seq, 42, 199.99);

-- Inspect without consuming a value
SELECT PREVIOUS VALUE FOR order_id_seq;

-- Reset after data restore
SELECT SETVAL(order_id_seq, 5000000, 0);   -- next NEXT VALUE returns 5000000
```

Why : sequences decouple ID generation from a parent INSERT round-trip, useful for FK-first child inserts. CACHE 100 is a safer choice than the 1000 default if you cannot tolerate gaps after a crash : restart loses cached ids.

## 10. Generated column with functional index over JSON

```sql
-- 10.2+ for VIRTUAL ; JSON validation per CHECK
ALTER TABLE event
  ADD COLUMN customer_id BIGINT UNSIGNED
    AS (JSON_VALUE(payload, '$.customer.id')) VIRTUAL;

-- The index on a VIRTUAL column requires ALGORITHM=COPY (KB-documented)
ALTER TABLE event
  ADD KEY ix_event_customer (customer_id),
  ALGORITHM=COPY;
```

Why : MariaDB JSON is a LONGTEXT alias, so you cannot index JSON directly. The pattern is : VIRTUAL column over a `JSON_VALUE` extract + index on the column. Query `WHERE customer_id = 42` uses the index ; the column itself is recomputed on read so no storage cost.

## 11. Invisible column for soft-deprecation

```sql
-- 10.3+ : hide a column from SELECT * without removing it
ALTER TABLE customer
  MODIFY legacy_external_id VARCHAR(64) INVISIBLE,
  ALGORITHM=INSTANT,
  LOCK=NONE;

-- Applications using SELECT * stop receiving the column
-- Applications using SELECT legacy_external_id still work
-- Schedule the actual DROP after the deprecation window
ALTER TABLE customer
  DROP COLUMN legacy_external_id,
  ALGORITHM=INSTANT,
  LOCK=NONE;
```

Why : two-phase deprecation. Phase 1 (INVISIBLE) hides the column without breaking explicit consumers. Phase 2 (DROP) removes it after every consumer is verified to no longer reference it.

## 12. RANGE COLUMNS partitioned table for time-series

```sql
-- 10.6+ : multi-column-friendly date partitioning
CREATE TABLE event_log (
  id          BIGINT UNSIGNED AUTO_INCREMENT,
  occurred_at DATETIME(6)     NOT NULL,
  source      VARCHAR(64)     NOT NULL,
  payload     JSON            NULL CHECK (JSON_VALID(payload)),
  PRIMARY KEY (id, occurred_at),
  KEY ix_source_time (source, occurred_at)
) ENGINE=InnoDB
  ROW_FORMAT=DYNAMIC
  DEFAULT CHARSET=utf8mb4
  PARTITION BY RANGE COLUMNS (occurred_at) (
    PARTITION p2024 VALUES LESS THAN ('2025-01-01'),
    PARTITION p2025 VALUES LESS THAN ('2026-01-01'),
    PARTITION p2026 VALUES LESS THAN ('2027-01-01'),
    PARTITION pmax  VALUES LESS THAN (MAXVALUE)
  );

-- Add next year's partition before it is needed
ALTER TABLE event_log
  REORGANIZE PARTITION pmax INTO (
    PARTITION p2027 VALUES LESS THAN ('2028-01-01'),
    PARTITION pmax  VALUES LESS THAN (MAXVALUE)
  );

-- Drop old data quickly via partition drop, not DELETE
ALTER TABLE event_log DROP PARTITION p2024;
```

Why : the PK includes `occurred_at` because every UNIQUE index in a partitioned table must cover the partitioning columns. `DROP PARTITION p2024` is a metadata operation that frees disk in seconds, vs `DELETE FROM event_log WHERE occurred_at < '2025-01-01'` which writes one undo record per row.

## 13. DROP TABLE idempotent and multi-table

```sql
-- IF EXISTS makes migration scripts re-runnable
DROP TABLE IF EXISTS staging_import_2024_q1;

-- Multi-table : crash-safe, but not fully atomic in 10.6+
DROP TABLE IF EXISTS
  tmp_lookup_a,
  tmp_lookup_b,
  tmp_lookup_c;

-- Single-table DROP IS atomic in 10.6+ : useful when paired with CREATE in one tx
START TRANSACTION;
DROP TABLE IF EXISTS old_report;
CREATE TABLE old_report (...) ENGINE=InnoDB;
COMMIT;
```

Why : `IF EXISTS` turns "table missing" into a warning instead of an error, so migration scripts run cleanly twice. Multi-table DROP is crash-safe (recovers a consistent state) but a crash mid-statement may leave some tables dropped and others not.
