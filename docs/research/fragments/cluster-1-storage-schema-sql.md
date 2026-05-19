# Cluster 1 : Storage, Schema, SQL (Research Fragment)

Scope : storage engines, schema design, DDL, DML, indexing, JSON, dynamic columns. Target versions : MariaDB 10.6-LTS, 10.11-LTS, 11.x current, 12.x next. Every claim verified via WebFetch against approved sources in `SOURCES.md`.

---

## Storage Engines

MariaDB ships multiple pluggable storage engines, each tuned for a different access pattern. Choosing the wrong engine is the single most expensive schema mistake and very hard to reverse without downtime.

**InnoDB** is the default engine since MariaDB 10.2 and is the only engine to choose for general transactional workloads. It is ACID-compliant, uses row-level locking, supports MVCC via undo logs, enforces foreign keys, and has full crash recovery via the redo log. Default page size is 16 KB ([KB](https://mariadb.com/kb/en/innodb/)). The default row format is DYNAMIC ([KB](https://mariadb.com/kb/en/innodb-row-formats/)).

**Aria** is the crash-safe MyISAM replacement. All system tables in `mysql.*` are Aria, and internal on-disk temporary tables also use Aria because its cache is faster than MyISAM for GROUP BY and DISTINCT ([KB](https://mariadb.com/kb/en/aria-storage-engine/)). Crash-safety is provided by `ROW_FORMAT=PAGE` (default). Aria offers no transactions and no MVCC, so it is unsuitable for concurrent user data; use it only when InnoDB is unavailable.

**MyISAM** is legacy. Table-level locking, no transactions, no MVCC, no foreign keys. Still in the binary for backward compatibility, but should never be picked for new tables ([KB](https://mariadb.com/kb/en/myisam-storage-engine/)).

**ColumnStore** is a columnar OLAP engine that lives in a separate distribution (MariaDB Enterprise / community ColumnStore packages) and is not part of the standard MariaDB Server install ([KB](https://mariadb.com/kb/en/mariadb-columnstore/)). Use it for analytical queries over very large datasets, never for OLTP. Row-by-row UPDATE / DELETE is supported but slow by design.

**Spider** is a sharding / federation engine built into MariaDB since 10.0.4. It maps a local table onto a remote backend, enabling horizontal partitioning across servers. Its high-availability features were deprecated in 10.7.5, so combine Spider with Galera or replication for HA ([KB](https://mariadb.com/kb/en/spider-storage-engine-overview/)). **MariaDB-only**.

**CONNECT** federates external data sources (CSV, XML, ODBC, JSON files, other databases) as SQL tables.

**MEMORY** keeps everything in RAM, only safe for ephemeral cache tables, supports hash indexes.

**Decision tree** : need ACID + concurrency → InnoDB. Need system-table or temp-table internals → Aria (managed by MariaDB itself). Need analytical scans over billions of rows → ColumnStore. Need horizontal sharding → Spider. Anything else → InnoDB.

```sql
-- 10.6+ : explicit engine, charset, row format
CREATE TABLE orders (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  created_at TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6)
) ENGINE=InnoDB
  ROW_FORMAT=DYNAMIC
  DEFAULT CHARSET=utf8mb4
  COLLATE=utf8mb4_uca1400_ai_ci;
```

## Schema Design

Naming conventions are not enforced by the engine, but `snake_case` table and column names avoid backtick churn and case-folding issues on case-insensitive filesystems. For primary keys, prefer `BIGINT UNSIGNED AUTO_INCREMENT` for single-node workloads. For distributed inserts use `UUID` (BINARY(16) with `UUID_TO_BIN(..., 1)` swap-flag) or ULID/UUIDv7 for time-ordered keys that preserve B-tree locality. Avoid `CHAR(36)` UUID strings : 36 bytes vs 16 binary bytes wastes ~125 % space per row and bloats every secondary index.

InnoDB row format DYNAMIC is the modern default and stores long variable-length columns off-page, which is required for indexed columns longer than 767 bytes. COMPRESSED uses zlib, costs CPU on every read, and **cannot use INSTANT ALTER TABLE** ([KB](https://mariadb.com/kb/en/innodb-row-formats/)). COMPACT is the older default with ~20 % space savings over REDUNDANT ([KB](https://mariadb.com/kb/en/innodb-row-formats/)). REDUNDANT exists only for MySQL 4.x compatibility.

Default charset migrated from `latin1` to `utf8mb4` in MariaDB 11.6, with collation `utf8mb4_uca1400_ai_ci` ([KB](https://mariadb.com/kb/en/character-set-and-collation-overview/)). For 10.6 and 10.11 LTS, the historical default is still `latin1` on stock builds (Debian/Ubuntu distros pin `utf8mb4` regardless). Always set the charset explicitly per table to avoid silent latin1 lock-in.

**Generated columns** exist since MariaDB 10.2 in both VIRTUAL (computed on read) and PERSISTENT/STORED (materialised on disk) flavours. Both can be indexed ([KB](https://mariadb.com/kb/en/generated-columns/)). The canonical pattern for JSON functional indexes : create a VIRTUAL column over `JSON_VALUE(doc, '$.path')`, then index that column.

**Invisible columns** since MariaDB 10.3 ([KB](https://mariadb.com/kb/en/invisible-columns/)). `SELECT *` skips them, but explicit `SELECT col` shows them. Use case : add a surrogate PK to a legacy table without breaking applications that hard-code column lists. `INVISIBLE NOT NULL` columns must have a DEFAULT. **MariaDB-only**, MySQL has no equivalent.

**Sequences** since MariaDB 10.3 ([KB](https://mariadb.com/kb/en/create-sequence/)). Syntactically Oracle / PostgreSQL compatible, not MySQL compatible (MySQL 8 has no sequences).

```sql
-- 10.3+ MariaDB-only
CREATE SEQUENCE invoice_seq
  START WITH 1000
  INCREMENT BY 1
  MINVALUE 1000
  MAXVALUE 9999999999
  CACHE 1000
  NOCYCLE;

INSERT INTO invoice (id) VALUES (NEXTVAL(invoice_seq));
SELECT PREVIOUS VALUE FOR invoice_seq;
```

## DDL

`ALTER TABLE` supports three algorithms : COPY (rewrite the whole table), INPLACE (modify in place, allow DML), and INSTANT (metadata-only, no data copy). Instant ADD COLUMN landed in MariaDB 10.3, with restrictions removed in 10.4 so columns can be added at any position, not just at the end ([KB](https://mariadb.com/kb/en/instant-add-column-for-innodb/)). INSTANT is unavailable for `ROW_FORMAT=COMPRESSED` and for tables with FULLTEXT indexes.

Always specify hints explicitly to fail fast when an unsafe algorithm is chosen :

```sql
-- 10.4+ : metadata-only column add, no rewrite
ALTER TABLE invoice
  ADD COLUMN paid_at TIMESTAMP(6) NULL AFTER created_at,
  ALGORITHM=INSTANT, LOCK=NONE;
```

If the server cannot honour INSTANT it errors out instead of silently falling back to COPY. For blue-green deployments use atomic `RENAME TABLE a TO a_old, b TO a;` which swaps tables in a single statement.

Partitioning supports RANGE, LIST, HASH, KEY, LINEAR HASH, LINEAR KEY, RANGE COLUMNS, LIST COLUMNS, and SYSTEM_TIME ([KB](https://mariadb.com/kb/en/partitioning-types-overview/)). **Foreign keys are forbidden in partitioned tables** : partitioned tables cannot have FKs and cannot be referenced by FKs ([KB](https://mariadb.com/kb/en/foreign-keys/)). This is a hard architectural boundary.

## DML

`INSERT ... ON DUPLICATE KEY UPDATE` is the standard upsert. The `VALUES(col)` function inside the UPDATE clause references the row that would have been inserted ([KB](https://mariadb.com/kb/en/insert-on-duplicate-key-update/)). With multiple unique indexes the behaviour is undefined : "only the first is updated. It is not recommended to use this statement on tables with more than one unique index." This is a real footgun.

`REPLACE INTO` is **DELETE + INSERT** under the hood, which means : (a) ON DELETE CASCADE fires, potentially wiping unrelated rows, (b) auto-increment burns a new value every time, (c) DELETE and INSERT triggers both fire ([KB](https://mariadb.com/kb/en/replace/)). Prefer `INSERT ... ON DUPLICATE KEY UPDATE`.

`INSERT IGNORE` silently coerces invalid values to the closest valid value and inserts them anyway ([KB](https://mariadb.com/kb/en/insert-ignore/)). NOT NULL with no value becomes empty string or zero. This masks data-quality bugs. Use it only when duplicates on a UNIQUE index are the explicit, documented behaviour.

`RETURNING` is supported on DELETE since MariaDB 10.0 and on INSERT since 10.5. UPDATE RETURNING is **not yet supported** in 10.6 / 10.11 LTS. Useful for fetching auto-generated IDs in a single round-trip without `LAST_INSERT_ID()`. **MariaDB-only** ; MySQL 8 has no RETURNING.

```sql
-- 10.5+ : single round-trip insert with generated id
INSERT INTO customer (name) VALUES ('Acme')
  RETURNING id, created_at;
```

Multi-table UPDATE and DELETE with JOIN syntax are supported : `UPDATE a JOIN b ON a.id=b.a_id SET a.x = b.x WHERE ...`. Combine with `ORDER BY ... LIMIT` for safe batch updates.

## Indexing

InnoDB and Aria default to B-tree indexes. MEMORY engine supports hash indexes (constant-time equality lookups, useless for range scans). MyISAM and InnoDB both support full-text indexing ; InnoDB FULLTEXT landed in 10.0.5 ([KB](https://mariadb.com/kb/en/full-text-indexes/)). Spatial indexing (R-tree) is available via `SPATIAL INDEX` on `GEOMETRY` columns in InnoDB and MyISAM.

The leftmost-prefix rule : a composite index `(a, b, c)` covers queries filtering on `(a)`, `(a, b)`, and `(a, b, c)`, but not on `(b)` or `(b, c)` alone. Column order is therefore critical : put the highest-selectivity equality predicate first, range predicates last.

Prefix indexes (`INDEX(name(20))`) are required for indexing TEXT and BLOB and useful for trimming index size on long VARCHAR. Descending indexes (`CREATE INDEX ix ON t(col DESC)`) are available since MariaDB 10.8 and let the optimiser avoid a backward scan for `ORDER BY col DESC` queries.

Ignored indexes (`ALTER TABLE t ALTER INDEX ix IGNORED`) are MariaDB 10.6+ ([KB](https://mariadb.com/kb/en/ignored-indexes/)). The index is still maintained but invisible to the optimiser : useful for "is this index actually needed?" experiments before a destructive DROP. MySQL 8 calls this feature "invisible indexes", which is semantically equivalent.

Read `EXPLAIN` carefully. `Using index condition` (ICP) means the WHERE predicate is pushed down to the storage engine, reducing row reads. `Using where` means a post-filter is applied after the row is fetched. `Using filesort` and `Using temporary` are both red flags indicating the optimiser cannot satisfy the ORDER BY / GROUP BY from an index ([KB](https://mariadb.com/kb/en/explain/)). Fix by adding a composite index that matches the query's filter + sort columns.

## JSON (MariaDB-specific divergence)

The biggest MariaDB-vs-MySQL footgun. **In MariaDB, JSON is an alias for LONGTEXT** ([KB](https://mariadb.com/kb/en/json-data-type/)). MySQL stores JSON in a compact binary format ; MariaDB stores plain text. Consequences :

- Row-based replication does not work for JSON columns between MySQL master and MariaDB replica. Use statement-based replication or convert to TEXT.
- Comparison semantics differ : MariaDB compares JSON as strings, MySQL compares structurally.
- Validation is opt-in via `CHECK (JSON_VALID(col))`. Without that constraint, MariaDB will happily store invalid JSON.

```sql
-- 10.2+ : JSON column with validation
CREATE TABLE event (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  payload JSON NOT NULL CHECK (JSON_VALID(payload))
) ENGINE=InnoDB;
```

Core functions : `JSON_VALID`, `JSON_EXTRACT`, `JSON_VALUE`, `JSON_QUERY`, `JSON_SET`, `JSON_INSERT`, `JSON_REPLACE`, `JSON_REMOVE`, `JSON_MERGE_PATCH`, `JSON_MERGE_PRESERVE`. `JSON_VALUE` returns scalars ; `JSON_QUERY` returns objects or arrays. `JSON_MERGE` is the deprecated alias and silently behaves like `JSON_MERGE_PRESERVE`, not `JSON_MERGE_PATCH` ([KB](https://mariadb.com/kb/en/json-functions/)).

`JSON_TABLE` (10.6+) projects JSON arrays into virtual relational rows with `NESTED PATH` for hierarchical structures ([KB](https://mariadb.com/kb/en/json_table/)) :

```sql
-- 10.6+ : JSON_TABLE projection
SELECT jt.name, jt.size
FROM products,
  JSON_TABLE(products.attrs, '$.variants[*]' COLUMNS (
    name VARCHAR(50) PATH '$.name',
    NESTED PATH '$.sizes[*]' COLUMNS (
      size VARCHAR(10) PATH '$'
    )
  )) AS jt;
```

JSONPath syntax : `$` is root, `$.key` selects a field, `$.arr[0]` selects an array element, `$.arr[*]` selects all elements, `$**.key` recursively descends.

## Dynamic Columns (MariaDB-only)

Dynamic columns predate JSON in MariaDB and store arbitrary attribute sets inside a BLOB column ([KB](https://mariadb.com/kb/en/dynamic-columns/)). All eight functions : `COLUMN_CREATE`, `COLUMN_GET`, `COLUMN_ADD`, `COLUMN_DELETE`, `COLUMN_EXISTS`, `COLUMN_LIST`, `COLUMN_CHECK`, `COLUMN_JSON`. Values carry their own type tags ; nesting is supported up to 10 levels deep. **Not available in MySQL.**

```sql
-- MariaDB-only : dynamic-column read
SELECT COLUMN_GET(attrs, 'color' AS CHAR), COLUMN_JSON(attrs)
FROM product
WHERE COLUMN_EXISTS(attrs, 'discontinued') = 0;
```

When to choose what : JSON when interoperability and JSONPath ergonomics matter ; dynamic columns when the data is internal, binary-efficient, and you never need standards-compliant JSON output. For new schemas, **default to JSON with `CHECK (JSON_VALID())`** ; dynamic columns are mature but a niche choice. Migration path : `UPDATE t SET j = COLUMN_JSON(dc)` then drop the dynamic-column blob.

---

## Anti-Patterns

1. **MyISAM for user data**. Row-locking is absent, crash recovery is absent, FKs are absent. Symptoms : every concurrent INSERT serialises on a table-lock, a power-cut corrupts indexes, referential integrity is enforced in application code (badly). Fix : `ALTER TABLE x ENGINE=InnoDB;` and audit every legacy table.

2. **JSON without CHECK(JSON_VALID())**. Because MariaDB JSON is LONGTEXT, an `INSERT INTO t (j) VALUES ('not even json')` succeeds silently and breaks every downstream `JSON_EXTRACT`. Fix : always add the CHECK constraint on schema creation.

3. **REPLACE INTO on parent tables**. `REPLACE INTO parent(id, …) VALUES(1, …)` deletes the existing row 1, which cascades through every child with ON DELETE CASCADE, then inserts a fresh row 1 with no children. Fix : use `INSERT … ON DUPLICATE KEY UPDATE`.

4. **INSERT IGNORE for silent dedup**. Hides constraint violations, type-coerces invalid values, and produces a successful return code with an empty error log. Fix : explicit `INSERT … ON DUPLICATE KEY UPDATE` or pre-check with a SELECT.

5. **Composite index in wrong column order**. `INDEX(created_at, customer_id)` cannot serve `WHERE customer_id = ?` queries because of the leftmost-prefix rule. The optimiser scans the whole table while the index looks fine in `SHOW CREATE TABLE`. Fix : put the equality column first, then the range column.

6. **Char(36) UUIDs as primary keys**. 36 ASCII characters × 4 bytes (utf8mb4) = 144 bytes per row in every secondary index, vs 16 bytes for BINARY(16). On a 100 M row table this is the difference between a 2 GB and a 14 GB index. Fix : `BINARY(16)` with `UUID_TO_BIN(uuid, 1)` so the time-low/high swap preserves clustering.

7. **`latin1` lock-in on stock 10.6 / 10.11**. Default charset on upstream binaries is still `latin1` for those LTS versions ; an unsuspecting `CREATE TABLE` without explicit charset gets `latin1_swedish_ci` and emoji break with `Incorrect string value`. Fix : set `character-set-server=utf8mb4` in `[mysqld]` and always declare `DEFAULT CHARSET=utf8mb4` per table.

8. **Foreign keys on a partitioned table**. The KB is explicit : "Partitioned tables cannot contain foreign keys, and cannot be referenced by a foreign key." Adding a partition spec to a child table silently removes its FK enforcement. Fix : either accept the loss of FK and enforce in app code, or skip partitioning and use archiving instead.

9. **Multiple UNIQUE indexes with `ON DUPLICATE KEY UPDATE`**. The KB warns : "If more than one unique index is matched, only the first is updated." Updates land on the wrong row, others are silently ignored. Fix : design a single composite UNIQUE that captures the intent, or split into two statements behind a transaction.

10. **`ALTER TABLE` without explicit `ALGORITHM=`**. The server picks COPY for any unsupported INSTANT change and the table is rewritten under a metadata lock. Production tables freeze for minutes to hours. Fix : always pass `ALGORITHM=INSTANT, LOCK=NONE` (or `INPLACE`) so the server fails fast instead of silently degrading.

---

## Newly Discovered Sub-Topics

These items were not in the raw masterplan but emerged during verification and deserve a place in Phase 3 planning :

- **System-versioned (temporal) tables** : `WITH SYSTEM VERSIONING` clause and `SYSTEM_TIME` partitioning. Built-in row-history since MariaDB 10.3, with bitemporal application-time + system-time combo since 10.4. Distinct skill, not just a DDL footnote.
- **Atomic DDL** : MariaDB 10.6 made DDL crash-safe via the binary log so partial CREATE / DROP / ALTER no longer leave orphan `.frm` files. Affects how migration scripts handle failure.
- **`utf8mb4_uca1400_*` collations** : new in 11.x, Unicode 14.0 aware, fixes ordering anomalies in `utf8mb4_general_ci` and `utf8mb4_unicode_ci`. Migration story between LTS releases is non-trivial.
- **`AS OF` time-travel queries** : `SELECT … FROM t FOR SYSTEM_TIME AS OF '2025-01-01'` on versioned tables. Native point-in-time read without recovery from backup.
- **CHECK constraints as universal validators** : not only for JSON. Since 10.2 CHECK is enforced (was parsed-but-ignored before). Replaces a lot of trigger-based validation logic.
- **`InnoDB` instant DROP COLUMN** : added in 10.4. Before 10.4, DROP COLUMN forced a full rebuild even on InnoDB. Important for online migrations on 10.4+ vs 10.3.
- **Optimizer trace** (`SET optimizer_trace='enabled=on'; SELECT … FROM information_schema.optimizer_trace`) gives full visibility into why a particular plan was chosen, beyond `EXPLAIN`.
- **`innodb_default_row_format`** server-variable governs new-table defaults. Auditing this on legacy servers is part of a healthy migration to DYNAMIC.
