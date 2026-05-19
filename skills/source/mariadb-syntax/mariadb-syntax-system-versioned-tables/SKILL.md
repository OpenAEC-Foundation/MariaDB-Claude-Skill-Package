---
name: mariadb-syntax-system-versioned-tables
description: >
  Use when building audit trails, GDPR-style history retention, time-travel queries, bitemporal data models, or migrating manual history-table patterns to native temporal tables.
  Prevents the common mistake of system-versioned tables without HISTORY PARTITION (unbounded growth), confusing system-time with application-time, or missing DELETE HISTORY for GDPR compliance.
  Covers WITH SYSTEM VERSIONING (10.3+), generated row_start/row_end columns, FOR SYSTEM_TIME AS OF/BETWEEN/FROM TO/ALL, partition-by-system-time, INTERVAL AUTO (10.9.1+), application-time periods (10.4+), bitemporal combining (10.5+), DELETE HISTORY BEFORE SYSTEM_TIME.
  Keywords: system-versioned, system versioning, temporal table, time travel, AS OF, audit trail, GDPR, history retention, bitemporal, application-time period, PERIOD FOR, row_start, row_end, DELETE HISTORY, MariaDB only, FOR SYSTEM_TIME, FOR PORTION OF, history partition, transaction precision, row level precision, point in time query, how do I track row history, why is my history table growing, how do I purge audit history
license: MIT
compatibility: "Designed for Claude Code. Requires MariaDB 10.6-LTS, 10.11-LTS, 11.x, 12.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# MariaDB System-Versioned Tables and Application-Time Periods

Native row-history, point-in-time queries, and bitemporal modelling. A MariaDB-only feature set since 10.3+. MySQL has no equivalent ; PostgreSQL has periods but no system-versioning. Read this skill before hand-rolling audit tables, before designing GDPR-compliant history retention, or before assuming MySQL behaviour applies.

## Quick Reference

- `WITH SYSTEM VERSIONING` is MariaDB-only. MySQL has neither system-versioning nor application-time periods. PostgreSQL has periods but no system-time. NEVER port these patterns to either without a rewrite.
- Default `row_start` / `row_end` precision is transaction-id (`BIGINT UNSIGNED`) on InnoDB. ALWAYS declare them as `TIMESTAMP(6)` when you need wall-clock time-travel.
- Transaction-precision tables CANNOT use `PARTITION BY SYSTEM_TIME`. Choose timestamp precision when you plan to partition.
- Without `PARTITION BY SYSTEM_TIME` the history grows in the same physical structure as live rows. ALWAYS plan retention up front.
- `DELETE HISTORY FROM t BEFORE SYSTEM_TIME 'ts'` is the GDPR-friendly purge. Requires the `DELETE HISTORY` privilege.
- `TRUNCATE TABLE` is BLOCKED on system-versioned tables (error 4137). Use `DELETE HISTORY FROM t` to wipe history, `DELETE FROM t` to remove current rows.
- Plain `SELECT * FROM t` returns only current rows. NEVER add `FOR SYSTEM_TIME AS OF NOW()` to "be safe" ; it is redundant and slower.
- Application-time periods (10.4+) describe business-time (`valid_from`/`valid_to`). System-time describes when the row was written. NEVER confuse the two.
- Bitemporal = `WITH SYSTEM VERSIONING` + a user `PERIOD FOR application_time(...)` clause. Since 10.5+. `FOR PORTION OF` works on application-time only, NEVER on `system_time`.
- `INTERVAL ... AUTO` (10.9.1+) auto-rotates history partitions. Prefer it on 10.11+ over manual `ALTER TABLE ... REORGANIZE PARTITION`.

## Decision Tree : When to Use Each Feature

```
Need to track changes over time ?
|
+- Track "when was this row written / corrected" (audit log, soft-delete, time-travel debugging)
|     => WITH SYSTEM VERSIONING        (system-time)
|
+- Track "when does this fact apply in the real world" (contracts, prices effective Jan-Mar)
|     => PERIOD FOR application_time   (application-time, 10.4+)
|
+- Both : audited contracts, regulatory bitemporal model
      => WITH SYSTEM VERSIONING + PERIOD FOR application_time  (10.5+)
```

```
Choosing system-time precision ?
|
+- Wall-clock time travel ("show me the row as of 2026-01-01")  => TIMESTAMP(6)
|
+- Transaction-precise auditing, no time-travel UI              => BIGINT UNSIGNED (default)
|
+- Need PARTITION BY SYSTEM_TIME                                 => TIMESTAMP(6) (mandatory)
```

```
History retention strategy ?
|
+- Small table, hand-purge ok            => DELETE HISTORY FROM t BEFORE SYSTEM_TIME 'ts'
|
+- Predictable retention window          => PARTITION BY SYSTEM_TIME INTERVAL 1 MONTH
|
+- Auto-rotate, no DBA action             => PARTITION BY SYSTEM_TIME INTERVAL 1 HOUR AUTO  (10.9.1+)
|
+- Size-bounded rotation                  => PARTITION BY SYSTEM_TIME LIMIT 100000
```

## Core Syntax

### Simplest form

```sql
-- 10.3+
CREATE TABLE accounts (
  id      INT PRIMARY KEY,
  balance DECIMAL(15,2)
) WITH SYSTEM VERSIONING;
```

`row_start` and `row_end` are implicit. The columns are hidden from `SELECT *` and addressed via pseudo-columns `ROW_START` / `ROW_END`. Default precision is `BIGINT UNSIGNED` transaction-id on InnoDB.

### Explicit timestamp-precision form

```sql
-- 10.3+
CREATE TABLE accounts (
  id        INT PRIMARY KEY,
  balance   DECIMAL(15,2),
  row_start TIMESTAMP(6) GENERATED ALWAYS AS ROW START,
  row_end   TIMESTAMP(6) GENERATED ALWAYS AS ROW END,
  PERIOD FOR SYSTEM_TIME (row_start, row_end)
) WITH SYSTEM VERSIONING;
```

ALWAYS choose this form when you need :

- Human-readable timestamps in time-travel queries
- `PARTITION BY SYSTEM_TIME` (transaction-precision rules out partitioning)
- Compatibility with SQL:2011 standard wording

### Adding versioning to an existing table

```sql
-- 10.3+
ALTER TABLE accounts ADD SYSTEM VERSIONING;

-- with explicit timestamp columns
ALTER TABLE accounts
  ADD COLUMN row_start TIMESTAMP(6) GENERATED ALWAYS AS ROW START,
  ADD COLUMN row_end   TIMESTAMP(6) GENERATED ALWAYS AS ROW END,
  ADD PERIOD FOR SYSTEM_TIME (row_start, row_end),
  ADD SYSTEM VERSIONING;
```

### Removing versioning

```sql
ALTER TABLE accounts DROP SYSTEM VERSIONING;
```

History rows survive as ordinary rows visible to plain `SELECT`. ALWAYS purge with `DELETE HISTORY` first if compliance requires the history to be discarded.

## Time-Travel Queries : FOR SYSTEM_TIME

```sql
-- 10.3+
SELECT * FROM accounts FOR SYSTEM_TIME AS OF TIMESTAMP '2026-01-01 00:00:00';
SELECT * FROM accounts FOR SYSTEM_TIME AS OF TRANSACTION 12345;
SELECT * FROM accounts FOR SYSTEM_TIME BETWEEN '2026-01-01' AND '2026-04-30';
SELECT * FROM accounts FOR SYSTEM_TIME FROM '2026-01-01' TO '2026-05-01';  -- end-exclusive
SELECT * FROM accounts FOR SYSTEM_TIME ALL;
```

Semantic differences :

| Clause | Inclusivity | Use case |
|--------|-------------|----------|
| `AS OF expr` | point-in-time | "show me row state at this instant" |
| `BETWEEN a AND b` | inclusive on both ends | "all versions overlapping this window" |
| `FROM a TO b` | inclusive-start, exclusive-end | half-open ranges, day-bucket reporting |
| `ALL` | every version ever recorded | full audit dump |

`AS OF TRANSACTION n` ONLY works when the table uses BIGINT-precision (transaction-id) versioning.

The session variable `system_versioning_asof` applies an implicit `AS OF` to all queries in the session. Use sparingly ; it is a debugging convenience, NOT a production access pattern.

## PARTITION BY SYSTEM_TIME

Without partitioning, history rows are appended to the same physical pages as current rows. Over months this degrades scans on current data. Use one of three rotation strategies :

```sql
-- 10.3+ : explicit partitions
CREATE TABLE audit_log (event_id BIGINT, payload TEXT) WITH SYSTEM VERSIONING
  PARTITION BY SYSTEM_TIME (
    PARTITION p_hist HISTORY,
    PARTITION p_cur  CURRENT
  );

-- 10.3+ : size-bounded rotation
CREATE TABLE audit_log (event_id BIGINT, payload TEXT) WITH SYSTEM VERSIONING
  PARTITION BY SYSTEM_TIME LIMIT 100000 (
    PARTITION p0 HISTORY,
    PARTITION p1 HISTORY,
    PARTITION pcur CURRENT
  );

-- 10.3+ : time-bounded rotation
CREATE TABLE audit_log (event_id BIGINT, payload TEXT) WITH SYSTEM VERSIONING
  PARTITION BY SYSTEM_TIME INTERVAL 1 MONTH (
    PARTITION p0 HISTORY,
    PARTITION p1 HISTORY,
    PARTITION p2 HISTORY,
    PARTITION pcur CURRENT
  );

-- 10.9.1+ : auto-managed time-bounded partitions
CREATE TABLE audit_log (event_id BIGINT, payload TEXT) WITH SYSTEM VERSIONING
  PARTITION BY SYSTEM_TIME INTERVAL 1 HOUR AUTO;

-- 10.9.1+ : auto with start anchor and seed-count
CREATE TABLE audit_log (event_id BIGINT, payload TEXT) WITH SYSTEM VERSIONING
  PARTITION BY SYSTEM_TIME INTERVAL 1 MONTH
    STARTS '2026-01-01 00:00:00' AUTO PARTITIONS 12;

-- 10.9.1+ : auto size-bounded
CREATE TABLE audit_log (event_id BIGINT, payload TEXT) WITH SYSTEM VERSIONING
  PARTITION BY SYSTEM_TIME LIMIT 1000 AUTO;
```

Rules :

- Exactly ONE `CURRENT` partition is required.
- At least ONE `HISTORY` partition is required.
- `LIMIT n` partitions are filled up to `n` rows, then the next HISTORY partition receives subsequent rows.
- `INTERVAL` partitions rotate by wall-clock ; the `STARTS` clause sets the anchor.
- `AUTO` (10.9.1+) creates new HISTORY partitions automatically as needed. Toggle with `ALTER TABLE ... PARTITION BY SYSTEM_TIME INTERVAL 1 HOUR AUTO;` (enable) or by re-issuing without `AUTO` (disable).
- ALWAYS combine `PARTITION BY SYSTEM_TIME` with timestamp-precision columns. Transaction-precision is incompatible with partitioning.

## History Pruning : DELETE HISTORY

```sql
-- 10.3+
DELETE HISTORY FROM accounts;                                   -- wipes all history
DELETE HISTORY FROM accounts BEFORE SYSTEM_TIME '2025-01-01';   -- timestamp purge
DELETE HISTORY FROM accounts BEFORE SYSTEM_TIME TRANSACTION 9999;
```

Requires the `DELETE HISTORY` privilege, which is separate from `DELETE`. ALWAYS grant it explicitly to GDPR-compliance roles ; NEVER fold it into the default app-user role.

Partitioned tables let you drop entire history partitions in O(1), which is cheaper than `DELETE HISTORY` :

```sql
-- 10.3+
ALTER TABLE audit_log DROP PARTITION p0;
```

## Application-Time Periods (10.4+)

```sql
-- 10.4+
CREATE TABLE prices (
  product_id INT,
  price      DECIMAL(10,2),
  valid_from DATE NOT NULL,
  valid_to   DATE NOT NULL,
  PERIOD FOR validity (valid_from, valid_to)
);

-- chop a period out
DELETE FROM prices
  FOR PORTION OF validity FROM '2026-03-01' TO '2026-04-01';

-- update a window only
UPDATE prices
  FOR PORTION OF validity FROM '2026-03-01' TO '2026-04-01'
  SET price = 9.99;
```

`PERIOD FOR period_name (start_col, end_col)` rules :

- Both columns MUST be `DATE`, `DATETIME`, or `TIMESTAMP` of the same width.
- Both columns are implicitly `NOT NULL`.
- An invisible CHECK constraint enforces `start_col < end_col`.
- `TIME` and `YEAR` columns are NOT allowed.

`FOR PORTION OF` semantics on DELETE :

| Range vs row | Result |
|--------------|--------|
| Complete overlap | row removed |
| Partial overlap | row shrunk to non-overlap remainder |
| Range fully inside row | row split into two rows |

`FROM ... TO ...` expressions MUST be constants (not parameters, not expressions referencing the row). Multi-table DELETE is NOT supported under `FOR PORTION OF`.

`UPDATE ... FOR PORTION OF` rules :

- Cannot SET the two period columns themselves.
- Cannot reference period values in the SET expression.
- `FROM ... TO ...` must be constant.

`WITHOUT OVERLAPS` constraint (10.5.3+) :

```sql
-- 10.5.3+
ALTER TABLE prices ADD PRIMARY KEY (product_id, validity WITHOUT OVERLAPS);
```

Prevents two rows with the same `product_id` from having overlapping `validity` periods.

## Bitemporal Tables (10.5+)

Combine `WITH SYSTEM VERSIONING` and `PERIOD FOR application_time(...)` on different column-pairs :

```sql
-- 10.5+
CREATE TABLE contracts (
  contract_id  INT,
  party        VARCHAR(120),
  terms        TEXT,
  valid_from   DATE NOT NULL,
  valid_to     DATE NOT NULL,
  row_start    TIMESTAMP(6) GENERATED ALWAYS AS ROW START INVISIBLE,
  row_end      TIMESTAMP(6) GENERATED ALWAYS AS ROW END   INVISIBLE,
  PERIOD FOR application_time (valid_from, valid_to),
  PERIOD FOR SYSTEM_TIME      (row_start, row_end)
) WITH SYSTEM VERSIONING;
```

Read this as :

- system-time : "when did the database know this row" (audit dimension)
- application-time : "when is the contract effective in the real world" (business dimension)

NEVER use `FOR PORTION OF system_time` ; it raises a syntax error. Only application-time supports portion-of DML.

## ALTER and Schema Evolution

`system_versioning_alter_history` (`ERROR` | `KEEP`) controls whether `ALTER TABLE` on a versioned table is rejected or allowed with history-accuracy compromised :

```sql
-- 10.3+
SET SESSION system_versioning_alter_history = KEEP;
ALTER TABLE accounts ADD COLUMN currency CHAR(3);
SET SESSION system_versioning_alter_history = DEFAULT;
```

ALWAYS read the variable's caveat : the history rows acquire the new column with `NULL` (or the column's default). You CANNOT reconstruct historical values for columns that did not exist at the time.

NEVER drop `row_start` or `row_end` columns directly. Drop versioning first :

```sql
ALTER TABLE accounts DROP SYSTEM VERSIONING;
ALTER TABLE accounts DROP COLUMN row_start, DROP COLUMN row_end, DROP PERIOD FOR SYSTEM_TIME;
```

## Column-Level Versioning

Exclude columns from versioning when their churn is high but irrelevant to audit (e.g. cached counters) :

```sql
-- 10.3+
CREATE TABLE users (
  id           INT PRIMARY KEY,
  email        VARCHAR(255),
  login_count  INT WITHOUT SYSTEM VERSIONING
) WITH SYSTEM VERSIONING;
```

A change to `login_count` does NOT produce a history row.

## Pseudo-Columns and Implicit Columns

When `row_start` and `row_end` are implicit (simplest-form `WITH SYSTEM VERSIONING` without an explicit declaration), they remain hidden from `SELECT *` but are addressable :

```sql
SELECT id, ROW_START, ROW_END FROM accounts FOR SYSTEM_TIME ALL;
```

MariaDB 11.7+ supports converting implicit columns to explicit `row_start`/`row_end` columns without rebuild ; older versions require `ALTER TABLE ... ADD COLUMN ...` with rebuild and re-population.

## Reference Files

- `references/methods.md` : complete grammar for all clauses, variables, privileges, version table.
- `references/examples.md` : 10+ working CREATE / SELECT / DELETE / UPDATE examples with version annotations.
- `references/anti-patterns.md` : 8+ real-world traps with WHY they fail and the correct alternative.
