# MariaDB SQL DDL : Anti-Patterns

Nine real-world DDL anti-patterns with root cause, observed symptoms, and the correct alternative.

## 1. ALTER TABLE without ALGORITHM hint on a huge table

### Anti-pattern
```sql
ALTER TABLE orders ADD COLUMN refunded_at TIMESTAMP(6) NULL;
```

### Why it fails
With no explicit `ALGORITHM=`, the server picks `DEFAULT` and may resolve it to `COPY` if anything blocks INSTANT (compressed row format, FULLTEXT index, older version). On a 100M-row table, COPY rewrites every row to a temp file, blocks all DML until 11.2+, and runs for hours.

### Symptoms
- `SHOW PROCESSLIST` shows the ALTER in `copy to tmp table` state.
- Connection pool exhausts because DML threads wait for the metadata lock.
- Disk consumption doubles for the duration.

### Fix
```sql
ALTER TABLE orders
  ADD COLUMN refunded_at TIMESTAMP(6) NULL,
  ALGORITHM=INSTANT,
  LOCK=NONE;
```

If the server cannot honour INSTANT it errors out (1845) before touching data. You then evaluate the trade-off explicitly instead of discovering it at 03:00.

## 2. ALTER TABLE on a `ROW_FORMAT=COMPRESSED` table expecting INSTANT

### Anti-pattern
```sql
ALTER TABLE archive_log ADD COLUMN replayed BOOL NOT NULL DEFAULT 0,
  ALGORITHM=INSTANT;
-- ERROR 1845 : ALGORITHM=INSTANT is not supported for this operation
```

### Why it fails
`ROW_FORMAT=COMPRESSED` is explicitly excluded from INSTANT ADD COLUMN (KB documented). Same for tables that contain a hidden `FTS_DOC_ID` introduced by FULLTEXT INDEX.

### Fix
Either accept INPLACE/COPY, or change ROW_FORMAT first :
```sql
-- Option A : accept INPLACE
ALTER TABLE archive_log
  ADD COLUMN replayed BOOL NOT NULL DEFAULT 0,
  ALGORITHM=INPLACE,
  LOCK=NONE;

-- Option B : drop compression first (rewrites the table once, then enjoy INSTANT later)
ALTER TABLE archive_log
  ROW_FORMAT=DYNAMIC,
  ALGORITHM=COPY,
  LOCK=SHARED;
```

## 3. DROP TABLE without backup or verification window

### Anti-pattern
```sql
DROP TABLE customer_old;
```

### Why it fails
Once dropped, the only recovery is from a backup. If the table was still referenced by an obscure cron job, a replica, or a BI dashboard, the failure shows up hours later as missing data, not as a syntax error.

### Fix
Two-phase deprecation :
```sql
-- Phase 1 : rename, keep data around for the verification window
RENAME TABLE customer_old TO customer_old_retired_20260519;

-- Verify zero query traffic against the retired name :
-- SELECT digest_text FROM performance_schema.events_statements_history_long WHERE digest_text LIKE '%customer_old_retired%';

-- Phase 2 : drop after the verification window
DROP TABLE IF EXISTS customer_old_retired_20260519;
```

## 4. Sequence CACHE size mismatched to risk tolerance

### Anti-pattern
```sql
CREATE SEQUENCE invoice_seq CACHE 1000000;   -- 1M values cached per request
```

### Why it fails
On crash or normal restart, every unused cached value is lost. A million-cached-value sequence creates million-id gaps after every restart : the sequence is monotonic but extremely sparse.

The opposite extreme is also wrong :
```sql
CREATE SEQUENCE invoice_seq NOCACHE;   -- one disk write per NEXT VALUE FOR
```
Every NEXT VALUE FOR forces a disk write to the system tablespace, so a bulk INSERT generating IDs becomes IO-bound.

### Fix
Choose CACHE deliberately based on gap tolerance vs throughput :
```sql
-- Most OLTP : 100 is a reasonable middle ground
CREATE SEQUENCE invoice_seq START WITH 1 INCREMENT BY 1 CACHE 100 NOCYCLE;

-- High-throughput bulk-insert : 1000 (the default) is fine
CREATE SEQUENCE bulk_id_seq CACHE 1000 NOCYCLE;

-- Financial / monotonic-strict : NOCACHE if gaps are unacceptable
CREATE SEQUENCE legal_doc_seq NOCACHE NOCYCLE;
```

## 5. PERSISTENT generated column when VIRTUAL suffices

### Anti-pattern
```sql
ALTER TABLE event
  ADD COLUMN customer_id BIGINT UNSIGNED
    AS (JSON_VALUE(payload, '$.customer.id')) PERSISTENT;
```

### Why it fails
PERSISTENT stores the computed value on every row, increases table size, doubles the work on every UPDATE that touches the source expression, and is replicated in row-based replication. If the column is only used as an index target, the extra disk footprint is pure waste.

### Fix
```sql
ALTER TABLE event
  ADD COLUMN customer_id BIGINT UNSIGNED
    AS (JSON_VALUE(payload, '$.customer.id')) VIRTUAL;

ALTER TABLE event
  ADD KEY ix_event_customer (customer_id),
  ALGORITHM=COPY;
```

VIRTUAL recomputes on read but the index is materialised. Use PERSISTENT only when you need : an FK pointing at the column, row-based replication of the computed value, or read-heavy queries where the expression cost dominates.

## 6. Treating INVISIBLE as access control

### Anti-pattern
```sql
ALTER TABLE customer
  ADD COLUMN ssn VARCHAR(20) INVISIBLE;
-- "Now ssn is hidden from regular users"
```

### Why it fails
INVISIBLE is a `SELECT *` optimisation, not a security feature. Any user with SELECT privilege can `SELECT ssn FROM customer` and the column appears. `INFORMATION_SCHEMA.COLUMNS` also lists invisible columns.

### Fix
Use real access control :
```sql
-- Split sensitive data into its own table with restricted grants
CREATE TABLE customer_sensitive (
  customer_id BIGINT UNSIGNED PRIMARY KEY,
  ssn         VARCHAR(20) NOT NULL,
  CONSTRAINT fk_sensitive_customer FOREIGN KEY (customer_id) REFERENCES customer(id) ON DELETE CASCADE
);

REVOKE ALL ON customer_sensitive.* FROM 'app_readonly'@'%';
GRANT SELECT ON customer_sensitive TO 'compliance_team'@'%';
```

## 7. Partitioning by an expression that defeats pruning

### Anti-pattern
```sql
CREATE TABLE event_log (
  id BIGINT, occurred_at DATETIME(6) NOT NULL,
  PRIMARY KEY (id, occurred_at)
) PARTITION BY RANGE (YEAR(occurred_at)) (
  PARTITION p2024 VALUES LESS THAN (2025),
  PARTITION p2025 VALUES LESS THAN (2026),
  PARTITION pmax  VALUES LESS THAN (MAXVALUE)
);

SELECT * FROM event_log WHERE occurred_at >= '2025-01-01' AND occurred_at < '2025-02-01';
```

### Why it fails
The partition expression is `YEAR(occurred_at)`. A WHERE filter on `occurred_at` (not `YEAR(occurred_at)`) the optimiser cannot rewrite to a partition list, so it scans every partition. The original intent (prune to p2025) is lost.

`EXPLAIN PARTITIONS` shows `partitions: p2024,p2025,pmax` for what should be a single-partition scan.

### Fix
Partition on the column directly, using `RANGE COLUMNS` for date types :
```sql
CREATE TABLE event_log (
  id BIGINT, occurred_at DATETIME(6) NOT NULL,
  PRIMARY KEY (id, occurred_at)
) PARTITION BY RANGE COLUMNS (occurred_at) (
  PARTITION p2024 VALUES LESS THAN ('2025-01-01'),
  PARTITION p2025 VALUES LESS THAN ('2026-01-01'),
  PARTITION pmax  VALUES LESS THAN (MAXVALUE)
);
```

Now `WHERE occurred_at >= '2025-01-01'` prunes to `p2025,pmax`.

## 8. Expecting RENAME TABLE to be atomic across schemas with triggers

### Anti-pattern
```sql
RENAME TABLE app_v1.orders TO app_v2.orders;
-- ERROR 1435 : Trigger in wrong schema
```

### Why it fails
Cross-schema RENAME TABLE works at the file level but is blocked when the source table has triggers attached : triggers are schema-bound and cannot be transparently moved (error 1435). Views likewise produce error 1450.

### Fix
Two-step migration : drop triggers, rename, recreate triggers in the target schema :
```sql
-- 1. Capture trigger definitions
SHOW CREATE TRIGGER app_v1.trg_orders_audit;

-- 2. Drop triggers in the source schema
DROP TRIGGER app_v1.trg_orders_audit;

-- 3. Rename
RENAME TABLE app_v1.orders TO app_v2.orders;

-- 4. Recreate triggers in the target schema, adjusting any schema-qualified references
CREATE TRIGGER app_v2.trg_orders_audit AFTER INSERT ON app_v2.orders
  FOR EACH ROW INSERT INTO app_v2.audit_log VALUES (NEW.id, NOW());
```

For zero-downtime cross-schema migration, prefer dual-write + cutover at the application layer ; do not rely on RENAME TABLE alone.

## 9. ALTER on a view (views do not support ALTER VIEW for body changes)

### Anti-pattern
```sql
ALTER VIEW customer_summary
  AS SELECT id, name, total_spent FROM customer JOIN ...;
```

### Why it fails
`ALTER VIEW` exists in MariaDB syntax but is limited : you can change view attributes (DEFINER, SQL SECURITY, ALGORITHM) but the body re-definition is unreliable across versions and column-set changes are not propagated cleanly. The reliable, version-portable form is to drop-and-recreate.

### Fix
```sql
CREATE OR REPLACE VIEW customer_summary AS
  SELECT c.id, c.name, COALESCE(SUM(i.amount), 0) AS total_spent
    FROM customer c
    LEFT JOIN invoice i ON i.customer_id = c.id
    GROUP BY c.id, c.name;
```

`CREATE OR REPLACE VIEW` atomically replaces the view definition. It is also crash-safe under atomic DDL (10.6+). Use this pattern for every view body change ; reserve `ALTER VIEW` for permission/algorithm attribute tweaks.
