---
name: mariadb-syntax-sql-ddl
description: >
  Use when creating or altering tables, choosing an ALTER algorithm, dropping objects safely, working with sequences, virtual/persistent columns, invisible columns, or partitioning.
  Prevents the costly COPY-algorithm ALTER on production, dropping a table referenced by FK without cascade, picking PERSISTENT when VIRTUAL is enough, or misuse of INSTANT-DROP-COLUMN locking semantics.
  Covers CREATE TABLE patterns, ALTER algorithms INPLACE/COPY/INSTANT/NOCOPY, atomic DDL (10.6+), sequences (10.3+), generated columns (VIRTUAL vs PERSISTENT), invisible columns (10.3+), partitioning (RANGE/LIST/HASH/KEY/COLUMNS), RENAME TABLE atomic-rename.
  Keywords: CREATE TABLE, ALTER TABLE, DROP TABLE, ALGORITHM=INPLACE, ALGORITHM=COPY, ALGORITHM=INSTANT, ALGORITHM=NOCOPY, sequences, AUTO_INCREMENT, virtual columns, persistent columns, stored columns, invisible columns, partitioning, RENAME TABLE, atomic DDL, online DDL, instant ADD COLUMN, instant DROP COLUMN, how do I add a column without locking, schema migration, blue-green deployment, schema change, alter table is hanging, my migration is slow, partition pruning, foreign key partitioned table
license: MIT
compatibility: "Designed for Claude Code. Requires MariaDB 10.6-LTS, 10.11-LTS, 11.x, 12.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# MariaDB SQL DDL

Definition statements that change the schema : CREATE, ALTER, DROP, RENAME, plus sequences, generated columns, invisible columns, and partitioning. Read this before any schema migration on a production-size table.

## Quick Reference

- ALWAYS specify `ALGORITHM=` and `LOCK=` on every `ALTER TABLE` so the server fails fast instead of silently falling back to COPY.
- ALWAYS prefer `ALGORITHM=INSTANT` for ADD COLUMN (10.3+) and DROP COLUMN (10.4+) when the table allows it. Metadata-only change, no rewrite.
- ALWAYS use `ALGORITHM=INPLACE, LOCK=NONE` for adding non-INSTANT-able indexes on big tables.
- NEVER default to `ALGORITHM=COPY` on production-size tables : rebuilds the whole table, blocks DML in earlier versions. From 11.2+ COPY supports `LOCK=NONE` but still rewrites everything.
- NEVER add an `INSTANT` column to a `ROW_FORMAT=COMPRESSED` table or to a table with a FULLTEXT index : both reject INSTANT.
- ALWAYS choose `VIRTUAL` for generated columns unless you need an FK pointing at the column, replication of the computed value, or skip the recompute cost on heavy reads. PERSISTENT/STORED is synonyms but PERSISTENT is the canonical MariaDB keyword.
- ALWAYS use atomic `RENAME TABLE old TO retired, new TO old;` for blue-green deployments. Multi-table RENAME is atomic within a single statement (10.6+ atomic DDL).
- NEVER use `ALTER VIEW` and expect rebuild : views require `CREATE OR REPLACE VIEW`.
- Sequences are MariaDB-only (10.3+), Oracle/PostgreSQL-compatible syntax. MySQL has no sequences.
- Partitioned tables CANNOT have foreign keys and CANNOT be referenced by foreign keys : hard architectural boundary.
- Atomic DDL is rollback-safe on server crash (10.6+) for single-table DDL. Multi-table DROP is crash-safe but not fully atomic.

## Decision Trees

### Picking an ALTER algorithm

```
Is the change metadata-only (ADD COLUMN at any position 10.4+, DROP COLUMN 10.4+, RENAME COLUMN/INDEX 10.5.3+, modify ENUM by appending, set DEFAULT) ?
   YES : ALGORITHM=INSTANT, LOCK=NONE
   NO  : continue

Is the change an ADD INDEX, DROP INDEX, or column modification that does not rewrite clustered-index keys ?
   YES : ALGORITHM=INPLACE, LOCK=NONE
   NO  : continue

Does the change require rebuilding the clustered index (change PK, change column type, change ROW_FORMAT) ?
   YES on small table  : ALGORITHM=COPY, LOCK=SHARED, run during maintenance window
   YES on big table    : use pt-online-schema-change or gh-ost ; never inline COPY on prod
```

### Generated column : VIRTUAL or PERSISTENT

```
Do you need an FK pointing at the generated column ?
   YES : PERSISTENT (FK only allowed on stored)
   NO  : continue

Do you replicate row-based and need the computed value applied identically on the replica ?
   YES : PERSISTENT
   NO  : continue

Is the read path heavy and the expression expensive ?
   YES : PERSISTENT (trade disk for CPU)
   NO  : VIRTUAL (default ; no disk cost)
```

### Partitioning : type selection

```
Filter is a date / numeric range with predictable boundaries ?
   YES : RANGE or RANGE COLUMNS (multi-column variant)

Filter is a finite set of discrete values (region codes, tenant ids) ?
   YES : LIST or LIST COLUMNS

You want even distribution by hashing a key, no semantic boundary ?
   YES : HASH (LINEAR HASH for less-expensive partition addition)

You want hash-partitioning by primary key columns ?
   YES : KEY (or LINEAR KEY)

You need automatic time-period partitions tied to system-versioning ?
   YES : SYSTEM_TIME (MariaDB-only ; requires WITH SYSTEM VERSIONING)
```

## Patterns

### CREATE TABLE (modern default)

```sql
-- 10.6+ : production-grade table template
CREATE TABLE invoice (
  id           BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  customer_id  BIGINT UNSIGNED NOT NULL,
  amount       DECIMAL(12,2)   NOT NULL,
  status       ENUM('draft','sent','paid','void') NOT NULL DEFAULT 'draft',
  metadata     JSON            NULL CHECK (JSON_VALID(metadata)),
  created_at   TIMESTAMP(6)    NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  updated_at   TIMESTAMP(6)    NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),
  KEY ix_customer_status (customer_id, status),
  CONSTRAINT fk_invoice_customer FOREIGN KEY (customer_id) REFERENCES customer(id) ON DELETE RESTRICT
) ENGINE=InnoDB
  ROW_FORMAT=DYNAMIC
  DEFAULT CHARSET=utf8mb4
  COLLATE=utf8mb4_uca1400_ai_ci;
```

Always set CHARSET explicitly per table : 10.6 and 10.11 LTS stock builds still default to `latin1`, which silently breaks emoji and accented strings.

### Fast schema change : INSTANT ADD COLUMN

```sql
-- 10.4+ : metadata-only column add, no rewrite, no DML block
ALTER TABLE invoice
  ADD COLUMN paid_at TIMESTAMP(6) NULL AFTER created_at,
  ALGORITHM=INSTANT,
  LOCK=NONE;
```

The server errors out (HY000 1845) if INSTANT is impossible (compressed row format, FULLTEXT index present). No silent COPY fallback.

### Fast index add : INPLACE

```sql
-- 10.6+ : online index build, DML allowed during build
ALTER TABLE invoice
  ADD KEY ix_status_created (status, created_at),
  ALGORITHM=INPLACE,
  LOCK=NONE;
```

ADD INDEX is INPLACE-only ; it cannot be INSTANT because the index data structure must be populated.

### Multi-operation ALTER in one statement

```sql
-- 10.4+ : combine compatible operations to reduce one-touch overhead
ALTER TABLE invoice
  ADD COLUMN tax_amount DECIMAL(12,2) NOT NULL DEFAULT 0 AFTER amount,
  ADD COLUMN currency   CHAR(3)       NOT NULL DEFAULT 'EUR' AFTER tax_amount,
  ALGORITHM=INSTANT,
  LOCK=NONE;
```

All columns added in one metadata change. If any single sub-change rejects INSTANT, the whole statement errors out and nothing changes.

### Atomic table swap for blue-green deploys

```sql
-- 10.6+ : single-statement atomic swap, no client sees the wrong shape
RENAME TABLE
  invoice           TO invoice_v1,
  invoice_v2        TO invoice;
-- After verification :
DROP TABLE invoice_v1;
```

Within a single `RENAME TABLE` statement, all renames either succeed atomically or roll back together. Atomic for InnoDB, MyRocks, MyISAM, Aria.

### Sequence as MySQL-incompatible ID generator

```sql
-- 10.3+ MariaDB-only
CREATE SEQUENCE invoice_seq
  START WITH 1000
  INCREMENT BY 1
  MINVALUE   1000
  MAXVALUE   9999999999
  CACHE      1000
  NOCYCLE;

INSERT INTO invoice (id, customer_id, amount) VALUES
  (NEXT VALUE FOR invoice_seq, 42, 99.95);

-- Inspect the last value generated in this session
SELECT PREVIOUS VALUE FOR invoice_seq;
```

`NEXT VALUE FOR seq` is SQL-standard ; `NEXTVAL(seq)` is the legacy function form, both work. `CACHE 1000` pre-allocates 1000 ids per request : restart loses unused cached ids, which is a gap-creator. CACHE 1 maximises consistency at the cost of per-row overhead.

### Generated column with functional index (JSON path)

```sql
-- 10.2+ : VIRTUAL column over JSON path, indexed
ALTER TABLE event
  ADD COLUMN customer_id BIGINT UNSIGNED
    AS (JSON_VALUE(payload, '$.customer.id')) VIRTUAL,
  ADD KEY ix_event_customer (customer_id),
  ALGORITHM=COPY;
```

VIRTUAL re-evaluates on read but the index is materialised. ADD INDEX on a VIRTUAL column requires `ALGORITHM=COPY` (KB-documented limitation).

### Invisible column for soft-deprecation

```sql
-- 10.3+ : hide a legacy column from SELECT * without breaking explicit queries
ALTER TABLE customer
  MODIFY legacy_phone VARCHAR(20) INVISIBLE,
  ALGORITHM=INSTANT,
  LOCK=NONE;
```

`SELECT *` skips the column. `SELECT legacy_phone` still works. Marker for "remove in next major release" without immediate ALTER + app-deploy coordination.

### Range partition by date

```sql
-- 10.6+ : RANGE COLUMNS supports multi-column and date types
CREATE TABLE event_log (
  id          BIGINT UNSIGNED AUTO_INCREMENT,
  occurred_at DATETIME(6)     NOT NULL,
  payload     JSON            NULL CHECK (JSON_VALID(payload)),
  PRIMARY KEY (id, occurred_at)
) ENGINE=InnoDB
  PARTITION BY RANGE COLUMNS (occurred_at) (
    PARTITION p2024 VALUES LESS THAN ('2025-01-01'),
    PARTITION p2025 VALUES LESS THAN ('2026-01-01'),
    PARTITION p2026 VALUES LESS THAN ('2027-01-01'),
    PARTITION pmax  VALUES LESS THAN (MAXVALUE)
  );
```

The PK must include the partitioning column (`occurred_at`) : a hard MariaDB rule. Foreign keys are forbidden on partitioned tables.

### Drop table safely

```sql
-- IF EXISTS suppresses the no-such-table error ; useful in idempotent migrations
DROP TABLE IF EXISTS staging_import_2024;

-- Multi-table drop : crash-safe but not fully atomic in 10.6+
DROP TABLE IF EXISTS tmp_a, tmp_b, tmp_c;
```

If any table is referenced by a FK from another table, the DROP errors out (`Cannot delete or update a parent row : a foreign key constraint fails`). Drop child tables first or `SET FOREIGN_KEY_CHECKS=0` during controlled maintenance only.

## Atomic DDL (10.6+)

Single-statement DDL is rollback-safe across server crash : `CREATE TABLE` (except `CREATE OR REPLACE`), `ALTER TABLE`, single-table `DROP TABLE`, `RENAME TABLE`, `CREATE VIEW`, `CREATE SEQUENCE`, `CREATE TRIGGER`, `DROP TRIGGER`. The `ddl_recovery.log` tracks in-progress operations ; on restart MariaDB completes or reverses each entry up to 3 retries.

NOT fully atomic (crash-safe but partial-completion possible) : multi-table DROP TABLE, `CREATE OR REPLACE TABLE`, DROP DATABASE.

Storage-engine support : InnoDB, MyRocks, MyISAM, Aria. NOT supported : S3 engine, partitioning engine (some operations).

## Cross-References

- `mariadb-syntax-sql-dml` for INSERT/UPDATE/DELETE patterns including RETURNING and ON DUPLICATE KEY UPDATE.
- `mariadb-syntax-indexing` for index-type details, leftmost-prefix, descending indexes, ignored indexes.
- `mariadb-syntax-json` for JSON validation, JSON_VALUE/JSON_QUERY, functional indexes.
- `mariadb-impl-schema-design` for naming conventions, PK choice (BIGINT vs UUID), charset strategy.
- `mariadb-impl-migration` for online schema-change tools (pt-online-schema-change, gh-ost).
- `mariadb-core-storage-engines` for ROW_FORMAT trade-offs and engine selection.

## Reference Files

- `references/methods.md` : complete CREATE/ALTER/DROP/RENAME grammar, sequence functions, partitioning expressions.
- `references/examples.md` : 12+ working DDL examples covering every category listed in this skill.
- `references/anti-patterns.md` : 9 anti-patterns from real production incidents with root-cause and fix.

## Sources

Verified against : `mariadb.com/kb/en/alter-table/`, `mariadb.com/kb/en/instant-add-column-for-innodb/`, `mariadb.com/kb/en/create-sequence/`, `mariadb.com/kb/en/generated-columns/`, `mariadb.com/kb/en/invisible-columns/`, `mariadb.com/kb/en/partitioning-types-overview/`, `mariadb.com/kb/en/partition-pruning-and-selection/`, `mariadb.com/kb/en/rename-table/`, `mariadb.com/kb/en/atomic-ddl/`.
