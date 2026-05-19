---
name: mariadb-syntax-check-constraints
description: >
  Use when enforcing column or table invariants, validating JSON structure, replacing trigger-based validation, or migrating CHECK constraints from MySQL 8 or PostgreSQL.
  Prevents the common mistake of relying on app-layer validation, forgetting that NULL passes CHECK, conflating CHECK with NOT NULL, or expecting CHECK to fire on every UPDATE.
  Covers column-level CHECK, table-level CHECK, CHECK with JSON_VALID, NULL semantics (CHECK passes on NULL), interaction with INSERT IGNORE, performance impact, and when to choose CHECK vs trigger vs app-validation.
  Keywords: CHECK constraint, ALTER TABLE ADD CHECK, table constraint, column constraint, JSON_VALID CHECK, NULL semantics in CHECK, constraint vs trigger, why does my CHECK allow NULL, ER_CHECK_CONSTRAINT_VIOLATED, data integrity, schema invariant, drop constraint, check_constraint_checks, three-valued logic, SQL 3VL, INSERT IGNORE check, named constraint, how do I validate JSON, getting started with constraints
license: MIT
compatibility: "Designed for Claude Code. Requires MariaDB 10.6-LTS, 10.11-LTS, 11.x, 12.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# MariaDB CHECK Constraints

Deterministic guide for declaring, naming, dropping, and reasoning about `CHECK` constraints in MariaDB. CHECK constraints are enforced (not parsed-and-ignored) since MariaDB 10.2.1, which precedes MySQL 8 by several years. This skill applies to 10.6-LTS, 10.11-LTS, 11.x, and 12.x.

## Quick Reference

ALWAYS remember the following invariants:

- **CHECK passes when the predicate evaluates to NULL.** SQL three-valued logic (3VL) treats `NULL` as "unknown", and a CHECK only fails when the predicate is **explicitly FALSE**. A nullable column `col INT CHECK (col > 0)` will accept `NULL`. Combine with `NOT NULL` to require a value AND a positive value.
- **CHECK constraints are enforced since MariaDB 10.2.1**. Earlier versions parsed `CHECK` syntax but ignored it. MariaDB's enforcement predates MySQL 8.0 (released July 2018) by approximately 18 months.
- **Use `CHECK (JSON_VALID(col))` to validate JSON columns**. MariaDB's `JSON` type is a `LONGTEXT` alias, not native binary; without `JSON_VALID` you can insert literal garbage and break every downstream `JSON_EXTRACT`. (See L-005 / D-010.) From 10.4.3+, the `JSON` alias auto-applies `JSON_VALID` as a CHECK, but explicit `CHECK (JSON_VALID(col))` is required on `LONGTEXT`/`TEXT` columns used as JSON and recommended in all migration scripts.
- **INSERT IGNORE turns CHECK violation into a warning**. The row is **not** inserted, but no error is raised. Use plain `INSERT` when you need hard rejection.
- **Prefer CHECK over triggers for pure invariants.** CHECK is set-based, declarative, faster, and visible in `information_schema.CHECK_CONSTRAINTS`. Use triggers only when you need side-effects (audit row, modify other tables, derived columns) or cross-table validation.
- **Cross-table or sub-query CHECK is not allowed.** A CHECK predicate may only reference columns of the same row. Use a `BEFORE INSERT`/`BEFORE UPDATE` trigger with a `SIGNAL` if you must validate against another table.
- **Disable globally with `SET SESSION check_constraint_checks = OFF`** for bulk-load of legacy data, then re-enable. Disabled checks do not re-validate existing rows on re-enable.
- **`auto_increment` columns may not appear in a CHECK constraint** (per `mariadb.com/kb/en/constraint/`).
- **Only deterministic functions are permitted** in CHECK expressions. `NOW()`, `RAND()`, `UUID()`, `CURRENT_TIMESTAMP` ARE non-deterministic and MUST NOT appear in CHECK.

## Decision Tree : CHECK vs NOT NULL vs Trigger vs App-Layer

```
Need to enforce an invariant on a column?
|
+-- Just "value must be present"
|   --> Use NOT NULL. Do not add CHECK (col IS NOT NULL).
|
+-- Single-row predicate on one or more columns of THIS row
|   |
|   +-- Predicate is a deterministic expression (range, regex via REGEXP, enum-like IN-list, JSON_VALID)
|   |   --> Use CHECK.
|   |
|   +-- Predicate calls NOW(), UUID(), or any non-deterministic function
|       --> Use BEFORE INSERT/UPDATE trigger.
|
+-- Predicate must look up another table or aggregate
|   --> Use BEFORE INSERT/UPDATE trigger with SIGNAL SQLSTATE '45000'.
|       (CHECK cannot reference other tables.)
|
+-- Predicate involves business workflow (state machine, "manager must approve")
    --> Validate in application layer or stored procedure.
        CHECK is for data invariants, not workflow.
```

## Decision Tree : CHECK Alone vs CHECK + NOT NULL

```
Should the column allow NULL?
|
+-- Yes, NULL is a valid "value-unknown" sentinel
|   --> CHECK (col > 0)           -- NULL passes, positive numbers pass.
|
+-- No, the column must always have a value AND satisfy the predicate
    --> col INT NOT NULL CHECK (col > 0)
        -- Now NULL is rejected by NOT NULL, negative/zero by CHECK.
```

This is the single most common mistake. ALWAYS pair `NOT NULL` with `CHECK` if the column is required.

## Core Grammar

### Column-level CHECK

```sql
-- 10.2.1+
CREATE TABLE accounts (
  id      INT PRIMARY KEY,
  balance DECIMAL(12,2) NOT NULL CHECK (balance >= 0)
);
```

The CHECK clause appears inline in the column definition. Auto-generated constraint name is of the form `CONSTRAINT_1`, `CONSTRAINT_2`, ... You CANNOT reference an auto-generated name reliably across DDL replays. ALWAYS name CHECK constraints explicitly when you anticipate later `DROP CONSTRAINT`.

### Table-level CHECK with explicit name

```sql
-- 10.2.1+
CREATE TABLE orders (
  id         INT PRIMARY KEY,
  ordered_at DATETIME NOT NULL,
  shipped_at DATETIME,
  CONSTRAINT ship_after_order CHECK (shipped_at IS NULL OR shipped_at >= ordered_at)
);
```

Table-level CHECK can reference multiple columns of the same row. Naming the constraint (`CONSTRAINT ship_after_order`) makes it droppable, queryable in `information_schema.CHECK_CONSTRAINTS`, and identifiable in error output.

### ADD CHECK to an existing table

```sql
-- 10.2.1+
ALTER TABLE accounts
  ADD CONSTRAINT balance_non_negative CHECK (balance >= 0);
```

ALTER TABLE evaluates the new CHECK against every existing row. If any existing row fails the predicate, the ALTER fails. Use `SET SESSION check_constraint_checks = OFF;` only when you have audited the existing data and accept the inconsistency; the disabled state does not re-validate later.

### DROP a CHECK constraint

```sql
-- 10.2.1+
ALTER TABLE accounts DROP CONSTRAINT balance_non_negative;
```

Find auto-generated names via:

```sql
SELECT CONSTRAINT_NAME, CHECK_CLAUSE
  FROM information_schema.CHECK_CONSTRAINTS
 WHERE TABLE_SCHEMA = DATABASE() AND TABLE_NAME = 'accounts';
```

## CHECK and JSON

MariaDB JSON is a `LONGTEXT` alias, not a native binary type. WITHOUT a `CHECK (JSON_VALID(col))`, any string can be inserted, and `JSON_EXTRACT` will silently misbehave. (See D-010.)

```sql
-- 10.2.1+ (CHECK enforcement)
-- 10.4.3+ adds auto-CHECK on the JSON alias type, but explicit is safer
CREATE TABLE events (
  id      INT PRIMARY KEY,
  payload LONGTEXT NOT NULL CHECK (JSON_VALID(payload))
);
```

Note: `JSON_VALID(NULL)` returns `NULL`, which passes CHECK. If `payload` is nullable, malformed JSON is rejected but `NULL` is accepted. Use `NOT NULL` together with `CHECK (JSON_VALID(payload))` for "must be present AND valid JSON".

## CHECK and INSERT IGNORE

```sql
INSERT IGNORE INTO accounts (id, balance) VALUES (1, -50);
-- Row is NOT inserted; a warning is produced; no error is raised.
SHOW WARNINGS;
-- Warning | 4025 | CONSTRAINT `accounts.balance_non_negative` failed for `db`.`accounts`
```

ALWAYS use plain `INSERT` (not `INSERT IGNORE`) when you require hard failure on a CHECK violation. `REPLACE INTO` and `INSERT ... ON DUPLICATE KEY UPDATE` raise the violation as a regular error, not a warning.

## CHECK and Generated Columns

CHECK constraints on `VIRTUAL` generated columns are evaluated whenever the generated expression is evaluated (i.e. on row read for `VIRTUAL`, on row write for `PERSISTENT`/`STORED`). CHECK on a `PERSISTENT` generated column is evaluated on INSERT/UPDATE only and NEVER re-validates existing data when the underlying column changes through `ALTER`.

```sql
-- 10.2+
CREATE TABLE orders (
  id        INT PRIMARY KEY,
  qty       INT NOT NULL,
  unit_cost DECIMAL(10,2) NOT NULL,
  total     DECIMAL(12,2) AS (qty * unit_cost) PERSISTENT,
  CONSTRAINT total_positive CHECK (total > 0)
);
```

## Performance

CHECK constraints are evaluated once per INSERT and once per UPDATE that touches any column referenced in the predicate. Cost is proportional to predicate complexity:

- Simple comparisons (`col > 0`, `col IN ('a','b','c')`) are essentially free.
- Function calls (`JSON_VALID`, `REGEXP`) add measurable per-row cost on bulk inserts.
- Multi-column predicates do not require additional storage; they are stateless expressions.

NEVER replace a simple CHECK with a trigger "for performance." A CHECK is faster than the equivalent `BEFORE` trigger because the trigger requires a row-trigger context and SIGNAL handling, while CHECK is evaluated inline by the optimizer.

## Diff From MySQL 8

| Aspect | MariaDB 10.2.1+ | MySQL 8.0+ |
|---|---|---|
| First enforced version | 10.2.1 (2016) | 8.0.16 (2019) |
| Parsed but ignored in older versions | <= 10.2.0 (parsed only) | <= 8.0.15 (parsed only) |
| JSON type | LONGTEXT alias, requires `CHECK (JSON_VALID(col))` | Native binary, auto-validated |
| Auto-CHECK on JSON alias | 10.4.3+ | n/a (native type) |
| Disable globally | `SET check_constraint_checks = OFF` | `ALTER TABLE ... ALTER CHECK ... NOT ENFORCED` |
| Information schema view | `information_schema.CHECK_CONSTRAINTS` | `information_schema.CHECK_CONSTRAINTS` |

When migrating MySQL 8 -> MariaDB:

- All `CHECK` clauses are preserved syntactically.
- JSON columns lose native binary storage; ADD `CHECK (JSON_VALID(col))` to every JSON column to preserve the MySQL invariant. (See L-005.)
- `NOT ENFORCED` clause from MySQL is parsed but treated as ENFORCED in MariaDB. Audit migration scripts for `NOT ENFORCED` usage.

## Restrictions (Hard Rules)

NEVER include any of the following in a CHECK expression:

1. **Sub-queries.** `CHECK (id IN (SELECT id FROM other))` is rejected.
2. **References to columns of other tables.** CHECK predicates are row-local only.
3. **Non-deterministic functions.** `NOW()`, `CURRENT_TIMESTAMP`, `RAND()`, `UUID()`, `CONNECTION_ID()`, `USER()`, `CURRENT_USER()`, `LAST_INSERT_ID()` are all forbidden. Use a `BEFORE INSERT/UPDATE` trigger instead.
4. **Stored procedure or function calls that are not deterministic.** User-defined functions must be declared `DETERMINISTIC` to be permitted.
5. **`auto_increment` columns.** Per the KB constraint page, "auto_increment columns are not permitted in check constraints."
6. **Variables.** Session and user variables (`@var`, `@@session.*`) cannot be referenced.

## Disabling CHECK Globally (Bulk Load Only)

```sql
-- 10.2+
SET SESSION check_constraint_checks = OFF;
LOAD DATA INFILE '/tmp/legacy-dump.csv' INTO TABLE accounts ...;
SET SESSION check_constraint_checks = ON;
```

WARNING: re-enabling does NOT re-validate existing rows. Audit the bulk-loaded data manually:

```sql
SELECT * FROM accounts WHERE balance < 0;
```

## Common Errors

| SQLSTATE | Error code | Message fragment | Cause |
|---|---|---|---|
| HY000 | 4025 | `CONSTRAINT '...' failed for '...'.'...'` | CHECK predicate evaluated FALSE on insert/update |
| HY000 | 3819 | (legacy MySQL parity error code in some MariaDB builds) | Same as above on cross-compatibility builds |
| 42000 | 1064 | Syntax error near `CHECK` | CHECK on `auto_increment`, on sub-query, on non-deterministic function |

See `references/anti-patterns.md` for full reproductions.

## Cross-References

- `mariadb-syntax-json` : JSON storage as LONGTEXT, why `CHECK (JSON_VALID(col))` is mandatory.
- `mariadb-syntax-sql-ddl` : full DDL grammar, ALTER TABLE patterns.
- `mariadb-impl-schema-design` : when to choose CHECK vs trigger vs application validation.
- `mariadb-impl-migration-mysql-to-mariadb` : adding `CHECK (JSON_VALID(col))` during migration.
- `mariadb-errors-data-integrity` : decoding `CONSTRAINT ... failed` and recovery patterns.

## References

- `references/methods.md` : complete CHECK grammar, NULL semantics tables, MySQL 8 comparison, sql_mode interactions.
- `references/examples.md` : 10+ working CHECK constraint patterns with version annotations.
- `references/anti-patterns.md` : 7 real anti-patterns with reproductions and corrections.

## Sources

- MariaDB KB : Constraint reference, https://mariadb.com/kb/en/constraint/ (verified 2026-05-19)
- MariaDB KB : ALTER TABLE, https://mariadb.com/kb/en/alter-table/ (verified 2026-05-19)
- MariaDB KB : JSON_VALID, https://mariadb.com/kb/en/json_valid/ (verified 2026-05-19)
- MariaDB KB : INSERT IGNORE, https://mariadb.com/kb/en/insert-ignore/ (verified 2026-05-19)
- Vooronderzoek : `docs/research/vooronderzoek-mariadb.md` lines 236, 799 (consolidated sub-topic).
