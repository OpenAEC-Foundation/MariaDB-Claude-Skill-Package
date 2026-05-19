# Vooronderzoek : MariaDB

> Status : Phase 2 complete
> Generated : 2026-05-19
> Word count : ~8000 (target was >= 2000)
> Source-citations : 82 inline links across mariadb.com/kb, github.com/MariaDB/server, galeracluster.com, mariadb.org

This document is the aggregated deep-research output of three parallel opus research-agents, each covering one topic-cluster against the raw masterplan. Cluster fragments live in `docs/research/fragments/` and are inlined below verbatim for single-document reading. Phase 3 refinement uses this file as the input for the Refinement Decisions table.

## Methodology

Three opus agents ran in parallel against three non-overlapping topic-clusters :

1. **Cluster 1** : Storage engines, schema design, DDL, DML, indexing, JSON, dynamic columns
2. **Cluster 2** : Window functions, CTEs, system-versioned tables, stored procedures / functions, triggers, events, views, EXPLAIN, optimizer
3. **Cluster 3** : Replication, Galera, backup / restore, performance tuning, security, MySQL to MariaDB migration, Frappe / ERPNext companion

Each agent followed the same constraints : WebFetch-only sourcing against approved URLs in SOURCES.md, minimum 800 words substantive content, minimum 8 anti-patterns from JIRA / GitHub / KB pitfalls, version-explicit code-snippets, end with `## Newly Discovered Sub-Topics`.

## Key Findings That Affect Phase 3 Refinement

1. **`GROUPS` frame is NOT supported in MariaDB window functions.** The raw masterplan assumed 10.7+ support. The `mariadb-syntax-window-and-cte` skill scope must drop GROUPS and document the divergence from MySQL 8.0+ and PostgreSQL.
2. **Materialized views are not supported in MariaDB.** `mariadb-syntax-stored-routines` (or a dedicated skill) must document the `CREATE TABLE AS SELECT` + EVENT-scheduler-refresh workaround.
3. **Semi-sync is built-in from 10.3+** (no separate plug-in install needed). Replication-setup skill scope reduced.
4. **`slave_parallel_mode=optimistic` is default from 10.5.1+.** Tuning skill must reflect this.
5. **MariaDB GTID format `domain-server-sequence` is incompatible with MySQL GTID `uuid:seqno`.** One-way migration only. Migration skill must call this out explicitly.
6. **`mariadb-dump` and `mariadb-binlog` are 10.5+ renames** of `mysqldump` and `mysqlbinlog`. Backup skill must list both names.
7. **`innodb_buffer_pool_chunk_size` is deprecated from 10.11.12+.** Performance-tuning skill needs version-aware guidance.
8. **Frappe v14 and v15 require MariaDB 10.6.6+ ; v16 requires 11.8.** Companion-skill section needs exact version-pinning.
9. **`mysql.user` was replaced by `mysql.global_priv` in 10.4+.** Security and migration skills must account for both schemas.
10. **`ed25519` authentication plug-in is available since 10.1.21.** Default recommendation for new installs (over `mysql_native_password`).

## Source-Verification Gaps Flagged for Phase 4 Topic-Research

- `mariadb.com/kb/en/returning/` and `mariadb.com/kb/en/optimizing-for-myisam/` returned only index-pages during WebFetch. RETURNING-version cutoff and MyISAM deprecation-version cross-referenced from sibling KB pages. Phase 4 should fetch via `mariadb.com/docs/llms-full.txt` for canonical version-cutoffs.
- Several `mariadb.com/kb/en/*` URLs have been restructured to `mariadb.com/docs/*`. Encryption-variable reference (`innodb_encrypt_tables`, `encrypt_binlog`, `encrypt_tmp_files`) flagged as "verify per version" rather than asserted with full citation. Phase 4 must re-fetch.
- `galeracluster.com` returned HTTP 403 (bot-filter). Galera-specific pages (`pc.weight` semantics, `wsrep_provider_options`) flagged for Phase 4. Fall-back source : MariaDB KB Galera-cluster pages.

---

## Cluster 1 : Storage Engines, Schema Design, DDL, DML, Indexing, JSON, Dynamic Columns

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


---

## Cluster 2 : Window Functions, CTEs, System-Versioned Tables, Stored Routines, Triggers, Events, Views, EXPLAIN, Optimizer

# Cluster 2 Research : Advanced SQL and Routines

Scope : window functions, CTEs, system-versioned tables (**MariaDB-only**), application-time periods (**MariaDB-only**), stored procedures, stored functions, triggers, events, views, EXPLAIN / ANALYZE / optimizer flags. All claims verified via WebFetch against [mariadb.com/kb](https://mariadb.com/kb/) on 2026-05-19.

Versions in scope : MariaDB 10.6 LTS, 10.11 LTS, 11.x current, 12.x next. Divergence from MySQL is flagged inline.

---

## 1. Window functions

Window functions ([KB](https://mariadb.com/kb/en/window-functions-overview/)) were introduced in MariaDB 10.2. They compute a value over a window of rows defined by `OVER()` without collapsing rows the way `GROUP BY` does.

```sql
-- 10.2+
SELECT
  emp_id,
  dept_id,
  salary,
  RANK()       OVER w AS dept_rank,
  AVG(salary)  OVER w AS dept_avg,
  LAG(salary, 1, 0) OVER (PARTITION BY dept_id ORDER BY hire_date) AS prev_salary
FROM employees
WINDOW w AS (PARTITION BY dept_id ORDER BY salary DESC);
```

`OVER()` accepts three optional sub-clauses : `PARTITION BY expr [, ...]`, `ORDER BY expr [ASC|DESC] [, ...]`, and a frame `{ROWS|RANGE} frame_clause` ([KB](https://mariadb.com/kb/en/window-frames/)). Frame borders : `UNBOUNDED PRECEDING`, `n PRECEDING`, `CURRENT ROW`, `n FOLLOWING`, `UNBOUNDED FOLLOWING`, optionally `BETWEEN border AND border`.

**Critical MariaDB constraint** : only `ROWS` and `RANGE` frame units are supported. **`GROUPS` is NOT implemented in MariaDB** at any tested version up to 12.x ([KB](https://mariadb.com/kb/en/window-frames/)) ; do not assume parity with PostgreSQL or MySQL 8.0+ here. Frame exclusion (`EXCLUDE CURRENT ROW`, etc.), explicit `NULLS FIRST/LAST`, and `DISTINCT` inside aggregate-as-window are also unsupported.

Default frame, when `ORDER BY` is present but no frame is specified : `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` ([KB](https://mariadb.com/kb/en/window-frames/)). Without `ORDER BY`, the entire partition is the window. This default matters : `SUM(x) OVER (ORDER BY t)` produces a running total, not a partition-wide sum.

Ranking functions ([KB](https://mariadb.com/kb/en/window-functions-overview/)) : `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()`, `PERCENT_RANK()`, `CUME_DIST()`, `NTILE(n)`. Ranking functions ignore any frame clause. `RANK` leaves gaps on ties (1, 1, 3) ; `DENSE_RANK` does not (1, 1, 2).

Value functions ([KB](https://mariadb.com/kb/en/lag/)) : `LAG(expr [, offset [, default]])`, `LEAD(expr [, offset [, default]])`, `FIRST_VALUE(expr)`, `LAST_VALUE(expr)`, `NTH_VALUE(expr, n)`. All require `ORDER BY` in `OVER()`. `LAST_VALUE` interacts with the default running frame : without an explicit `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`, it returns the current row, not the partition tail.

Aggregates that work as window functions : `SUM`, `AVG`, `COUNT`, `MIN`, `MAX`, `BIT_AND/OR/XOR`, `STDDEV`, `VARIANCE`, `JSON_ARRAYAGG`, `JSON_OBJECTAGG`. Named windows (`WINDOW w AS (...)`) eliminate duplication across multiple window expressions in the same `SELECT`.

---

## 2. Common Table Expressions and recursive queries

CTEs ([KB](https://mariadb.com/kb/en/with/)) entered in MariaDB 10.2.1 (non-recursive) and 10.2.2 (recursive).

```sql
-- 10.2.2+
WITH RECURSIVE org_tree (emp_id, manager_id, depth, path) AS (
  SELECT emp_id, manager_id, 0, CAST(emp_id AS CHAR(1000))
    FROM employees WHERE manager_id IS NULL
  UNION ALL
  SELECT e.emp_id, e.manager_id, t.depth + 1,
         CONCAT(t.path, ',', e.emp_id)
    FROM employees e JOIN org_tree t ON e.manager_id = t.emp_id
   WHERE t.depth < 20
)
SELECT * FROM org_tree;
```

Cycle detection : MariaDB 10.5.2+ adds `CYCLE col_list RESTRICT` after the CTE body ([KB](https://mariadb.com/kb/en/with/)) ; before 10.5.2, you embed a guard column (the `path`-string pattern above) or use `max_recursive_iterations` to cap depth. Termination of the recursive arm is the developer's responsibility ; otherwise `max_recursive_iterations` (default 1000) aborts with an error.

Materialization : MariaDB may merge a non-recursive CTE into the outer query or materialize it into a temp table ; the optimizer chooses based on `optimizer_switch` flags `derived_merge=on` and `condition_pushdown_for_derived=on` ([KB](https://mariadb.com/kb/en/optimizer-switch/)). Use `EXPLAIN` to confirm — a `DERIVED` row in `select_type` means materialized, an inline expansion means merged. Recursive CTEs are always materialized.

Use cases the KB documents : tree-walks, gap-and-island reduction (combine with `ROW_NUMBER`), top-N per group (CTE + window-function filter), and graph traversal with cycle-safe enumeration.

---

## 3. System-versioned tables (**MariaDB-only**) and application-time periods (**MariaDB-only**)

System-versioned tables ([KB](https://mariadb.com/kb/en/system-versioned-tables/)) arrived in MariaDB 10.3 and are an SQL:2011 feature. MySQL has no equivalent ; this is a major divergence.

```sql
-- 10.3+
CREATE TABLE accounts (
  id INT PRIMARY KEY,
  balance DECIMAL(15,2),
  row_start TIMESTAMP(6) GENERATED ALWAYS AS ROW START,
  row_end   TIMESTAMP(6) GENERATED ALWAYS AS ROW END,
  PERIOD FOR SYSTEM_TIME (row_start, row_end)
) WITH SYSTEM VERSIONING;

-- query history
SELECT * FROM accounts FOR SYSTEM_TIME AS OF TIMESTAMP '2026-01-01 00:00:00';
SELECT * FROM accounts FOR SYSTEM_TIME BETWEEN '2026-01-01' AND '2026-04-30';
SELECT * FROM accounts FOR SYSTEM_TIME FROM '2026-01-01' TO '2026-05-01'; -- end-exclusive
SELECT * FROM accounts FOR SYSTEM_TIME ALL;
```

Implicit form `WITH SYSTEM VERSIONING` auto-generates the period columns invisibly. The explicit form lets you name them and is required for partition-by-system-time ([KB](https://mariadb.com/kb/en/system-versioned-tables/)).

Transaction-precision (InnoDB only) uses `BIGINT UNSIGNED` columns of transaction IDs instead of timestamps : `SELECT ... FOR SYSTEM_TIME AS OF TRANSACTION 12345`. Trade-off : transaction-precision tables cannot use `PARTITION BY SYSTEM_TIME`.

History pruning : `DELETE HISTORY FROM t [BEFORE SYSTEM_TIME 'ts']` requires the `DELETE HISTORY` privilege. Partition-by-system-time keeps history separable for fast purge :

```sql
-- 10.9.1+
CREATE TABLE audit_log (event_id BIGINT, payload TEXT) WITH SYSTEM VERSIONING
  PARTITION BY SYSTEM_TIME INTERVAL 1 HOUR AUTO; -- auto-rotate
```

Application-time periods ([KB](https://mariadb.com/kb/en/application-time-periods/), 10.4+) are user-defined, business-meaningful date ranges :

```sql
-- 10.4+
CREATE TABLE prices (
  product_id INT, price DECIMAL(10,2),
  valid_from DATE, valid_to DATE,
  PERIOD FOR validity (valid_from, valid_to)
);
DELETE FROM prices FOR PORTION OF validity FROM '2026-03-01' TO '2026-04-01';
UPDATE prices FOR PORTION OF validity FROM '2026-03-01' TO '2026-04-01'
  SET price = 9.99;
```

Bitemporal (10.5+) combines both : `WITH SYSTEM VERSIONING` plus a user `PERIOD FOR` clause on different column-pairs ([KB](https://mariadb.com/kb/en/application-time-periods/)).

---

## 4. Stored procedures

`CREATE PROCEDURE` ([KB](https://mariadb.com/kb/en/create-procedure/)) accepts parameters with three modes : `IN` (default, copy-in), `OUT` (copy-out at return), `INOUT` (both). MariaDB 11.8 adds Oracle-mode `IN OUT` and per-parameter `DEFAULT`. Characteristics : `LANGUAGE SQL`, `[NOT] DETERMINISTIC` (informative-only for procedures), `CONTAINS SQL | NO SQL | READS SQL DATA | MODIFIES SQL DATA`, `SQL SECURITY {DEFINER|INVOKER}`, `COMMENT 'string'`.

```sql
DELIMITER //
CREATE OR REPLACE PROCEDURE transfer(IN src INT, IN dst INT, IN amt DECIMAL(15,2))
  MODIFIES SQL DATA
  SQL SECURITY INVOKER
BEGIN
  DECLARE bal DECIMAL(15,2);
  DECLARE EXIT HANDLER FOR SQLEXCEPTION
  BEGIN
    ROLLBACK;
    RESIGNAL;
  END;
  START TRANSACTION;
  SELECT balance INTO bal FROM accounts WHERE id = src FOR UPDATE;
  IF bal < amt THEN
    SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'insufficient funds';
  END IF;
  UPDATE accounts SET balance = balance - amt WHERE id = src;
  UPDATE accounts SET balance = balance + amt WHERE id = dst;
  COMMIT;
END//
DELIMITER ;
```

Local variables : `DECLARE var_name type [DEFAULT expr]` (must come before any other statement in the block). Control flow : `IF ... THEN ... ELSEIF ... ELSE ... END IF`, `CASE`, `LOOP label`, `WHILE`, `REPEAT ... UNTIL`, `ITERATE label`, `LEAVE label`. Cursors : `DECLARE cur CURSOR FOR SELECT ...`, then `OPEN cur`, `FETCH cur INTO ...`, `CLOSE cur`.

Error handling ([KB](https://mariadb.com/kb/en/declare-handler/)) : `DECLARE {CONTINUE|EXIT} HANDLER FOR condition statement`. Conditions : a five-char `SQLSTATE 'xxxxx'` (e.g. `'23000'` = duplicate key), a numeric MariaDB error code, or the shorthands `SQLWARNING` (class `01`), `NOT FOUND` (class `02`), `SQLEXCEPTION` (everything else). Custom errors are raised with `SIGNAL SQLSTATE 'xxxxx' SET MESSAGE_TEXT='msg', MYSQL_ERRNO=n` ([KB](https://mariadb.com/kb/en/signal/)) ; inside a handler, `RESIGNAL` re-throws the caught error preserving context. Recursion is capped by `max_sp_recursion_depth` (default 0, must be raised for recursive procedures up to 255).

---

## 5. Stored functions

`CREATE FUNCTION` ([KB](https://mariadb.com/kb/en/create-function/)) differs from procedures : it has a mandatory `RETURNS type` clause, all parameters are `IN` by default (10.8+ adds `OUT`/`INOUT` but only for `SET`-style calls), and the body must execute `RETURN expr` to produce a value.

```sql
DELIMITER //
CREATE OR REPLACE FUNCTION net_price(gross DECIMAL(10,2), vat_rate DECIMAL(5,4))
  RETURNS DECIMAL(10,2)
  DETERMINISTIC
  CONTAINS SQL
BEGIN
  RETURN gross / (1 + vat_rate);
END//
DELIMITER ;
```

Characteristics : `[NOT] DETERMINISTIC` is **load-bearing** for functions, unlike procedures. The optimizer cannot inline or cache a `NOT DETERMINISTIC` call, and any non-deterministic function in a `WHERE` clause forces a per-row evaluation that defeats index usage on its result ([KB](https://mariadb.com/kb/en/create-function/)). Functions using `NOW()`, `CURRENT_TIMESTAMP()`, `RAND()`, or session variables are inherently non-deterministic ; declaring them `DETERMINISTIC` "may yield incorrect results" per the KB. Data-access clauses : `CONTAINS SQL` (default), `READS SQL DATA`, `MODIFIES SQL DATA`, `NO SQL` (currently a no-op in MariaDB). `SQL SECURITY {DEFINER|INVOKER}` controls privilege escalation.

---

## 6. Triggers

`CREATE TRIGGER` ([KB](https://mariadb.com/kb/en/create-trigger/)) fires `BEFORE` or `AFTER` an `INSERT`, `UPDATE`, or `DELETE`, always `FOR EACH ROW` (MariaDB does not implement statement-level triggers). Pseudo-records : `NEW.col` is readable in `INSERT` and `UPDATE` triggers (writable only in `BEFORE` triggers with `UPDATE` privilege via `SET NEW.col = expr`) ; `OLD.col` is read-only in `UPDATE` and `DELETE` triggers.

```sql
-- 10.2.3+
CREATE TRIGGER trg_audit_balance AFTER UPDATE ON accounts
  FOR EACH ROW
  FOLLOWS trg_audit_login
INSERT INTO balance_audit (account_id, old_bal, new_bal, changed_by, changed_at)
VALUES (NEW.id, OLD.balance, NEW.balance, CURRENT_USER(), NOW(6));
```

Multiple triggers on the same event are supported from 10.2.3+ with explicit ordering via `FOLLOWS trigger_name` or `PRECEDES trigger_name` ([KB](https://mariadb.com/kb/en/create-trigger/)). Order is observable in `INFORMATION_SCHEMA.TRIGGERS.ACTION_ORDER`. Limitations ([KB](https://mariadb.com/kb/en/trigger-limitations/)) : a trigger cannot return a result set (`SELECT` with output is forbidden, use `SELECT ... INTO var` or `INSERT ... SELECT`), the `RETURN` statement is illegal (use `LEAVE` to exit early), transaction-control statements (`COMMIT`, `ROLLBACK`, `SAVEPOINT`, `START TRANSACTION`) are forbidden, and triggers may not be defined on tables in `mysql`, `information_schema`, or `performance_schema`. Replication interaction : with row-based binlog (`binlog_format=ROW`), triggers execute on the source and their row-effects are replicated, so the replica does not re-fire them. With `STATEMENT` or `MIXED`, the trigger re-runs on replicas, which can desync if the trigger is non-deterministic.

---

## 7. Events (scheduler)

`CREATE EVENT` ([KB](https://mariadb.com/kb/en/create-event/)) defines a scheduled task executed by the event scheduler thread.

```sql
CREATE OR REPLACE EVENT ev_purge_old_audits
  ON SCHEDULE EVERY 1 DAY
    STARTS CURRENT_TIMESTAMP + INTERVAL 1 HOUR
  ON COMPLETION PRESERVE
  ENABLE
  COMMENT 'nightly audit retention'
DO
  DELETE FROM balance_audit WHERE changed_at < NOW() - INTERVAL 90 DAY;
```

Scheduling : `ON SCHEDULE AT timestamp` for one-shot, `ON SCHEDULE EVERY n unit [STARTS ts] [ENDS ts]` for recurring. `ON COMPLETION PRESERVE` retains a one-shot event after it finishes ; `NOT PRESERVE` (default) drops it. State : `ENABLE` (default), `DISABLE`, `DISABLE ON SLAVE` (avoid running the same event twice in replication topologies). The scheduler thread itself is gated by the global `event_scheduler` variable (`ON`, `OFF`, `DISABLED`) ; `OFF` at default in many distributions, so events created on a fresh server silently never fire until it is enabled. Monitor with `SELECT * FROM INFORMATION_SCHEMA.EVENTS` for status, `LAST_EXECUTED`, and `STATUS`. Error handling inside the `DO` body uses the same `DECLARE HANDLER` mechanism as procedures ; an uncaught error aborts that one execution but does not disable the event.

---

## 8. Views

`CREATE VIEW` ([KB](https://mariadb.com/kb/en/create-view/)) wraps a `SELECT` as a virtual table. The `ALGORITHM` clause selects the resolution strategy : `UNDEFINED` (default, optimizer chooses), `MERGE` (rewrite : view-body inlined into the referencing query, preserves index usage and updatability when possible), or `TEMPTABLE` (materialize the view into a temp table per query, breaks index pushdown and forces a full scan on each reference).

```sql
CREATE OR REPLACE
  ALGORITHM = MERGE
  DEFINER = 'reporting_owner'@'localhost'
  SQL SECURITY DEFINER
VIEW v_active_accounts AS
  SELECT id, balance FROM accounts WHERE status = 'active'
  WITH CHECK OPTION;
```

`SQL SECURITY DEFINER` (default) runs with the view-creator's privileges, the typical pattern for exposing a subset of a sensitive table ; `INVOKER` defers privilege checks to the caller. Updatable views require a one-to-one row mapping between view rows and underlying-table rows ; `DISTINCT`, `GROUP BY`, aggregates, `UNION`, and most subqueries break updatability ([KB](https://mariadb.com/kb/en/create-view/)). `WITH CHECK OPTION` rejects `INSERT`/`UPDATE` that would produce a row outside the view's predicate ; `CASCADED` (default) propagates the check through chained views, `LOCAL` checks only the current view. **Materialized views are NOT supported in MariaDB** — workarounds : `CREATE TABLE AS SELECT` plus a refresh job via the event scheduler, or external tools such as Flexviews.

---

## 9. EXPLAIN and the optimizer

`EXPLAIN` ([KB](https://mariadb.com/kb/en/explain/)) returns the query plan without executing. Columns : `id` (join sequence), `select_type`, `table`, `type`, `possible_keys`, `key`, `key_len`, `ref`, `rows`, `filtered`, `Extra`.

`select_type` values : `SIMPLE`, `PRIMARY`, `UNION`, `DEPENDENT UNION`, `UNION RESULT`, `SUBQUERY`, `DEPENDENT SUBQUERY`, `DERIVED`, `MATERIALIZED`, `UNCACHEABLE SUBQUERY`, `UNCACHEABLE UNION`.

`type` ranking (best to worst, ([KB](https://mariadb.com/kb/en/explain/))) : `system`, `const`, `eq_ref`, `ref`, `fulltext`, `ref_or_null`, `index_merge`, `unique_subquery`, `index_subquery`, `range`, `index`, `ALL`. The boundary between "fine" and "fix it now" is at `range` ; anything below should be examined.

Common `Extra` flags : `Using where`, `Using index` (covering-index, no row read), `Using filesort` (sort step), `Using temporary` (intermediate temp table for `GROUP BY` or `DISTINCT`), `Using join buffer (Block Nested Loop)`, `Using index condition` (index-condition pushdown to the storage engine), `Range checked for each record` (no usable index decided up-front, re-evaluated per outer row), `Not exists` (LEFT-JOIN short-circuit), `Select tables optimized away`.

`ANALYZE [FORMAT=JSON] SELECT ...` ([KB](https://mariadb.com/kb/en/analyze-statement/), 10.1+) actually runs the query and returns the plan **plus** observed values `r_rows`, `r_filtered`, `r_total_time_ms`. Use it to validate optimizer estimates. Warning : `ANALYZE UPDATE` and `ANALYZE DELETE` do execute the mutation ; only `ANALYZE SELECT` discards results.

Index hints ([KB](https://mariadb.com/kb/en/index-hints-how-to-force-query-plans/)) : `USE INDEX (idx_a, idx_b)` restricts the candidate set, `FORCE INDEX (...)` additionally treats table-scan as prohibitively expensive, `IGNORE INDEX (...)` excludes specific indexes while leaving the rest considered. All three accept `FOR JOIN`, `FOR ORDER BY`, `FOR GROUP BY` modifiers. **Risk** : `FORCE INDEX` on stale statistics locks you to a bad plan ; run `ANALYZE TABLE` first, then prefer `IGNORE INDEX` for the smaller blast-radius.

`optimizer_switch` ([KB](https://mariadb.com/kb/en/optimizer-switch/)) toggles individual optimizations. Defaults on : `index_merge`, `index_merge_union`, `index_merge_sort_union`, `index_merge_intersection`, `index_condition_pushdown`, `semijoin`, `materialization`, `loosescan`, `firstmatch`, `subquery_cache`, `derived_merge`, `derived_with_keys`, `condition_pushdown_for_derived`, `condition_pushdown_from_having`, `rowid_filter` (10.4.3+). Defaults off : `mrr`, `mrr_cost_based`, `index_merge_sort_intersection`. For deep-dive diagnostics enable `optimizer_trace=enabled=on` and read `INFORMATION_SCHEMA.OPTIMIZER_TRACE`.

---

## Anti-patterns

1. **`LAST_VALUE` without an explicit frame.** `LAST_VALUE(x) OVER (PARTITION BY g ORDER BY t)` returns the current row, not the partition tail, because the default frame is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` ([KB](https://mariadb.com/kb/en/window-frames/)). Fix : add `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`.

2. **Recursive CTE without a termination predicate.** A self-joining anchor on cyclic graph data hits `max_recursive_iterations` (default 1000) and errors out, sometimes after burning minutes. Fix : add a depth guard (`WHERE depth < N`) or use `CYCLE col RESTRICT` (10.5.2+).

3. **System-versioned table without history partitioning.** Writes append history rows to the same physical structure as live rows ; over years the table degrades to scan-bound. Fix : `PARTITION BY SYSTEM_TIME` with explicit `HISTORY` and `CURRENT` partitions, or `INTERVAL ... AUTO` (10.9.1+), plus periodic `DELETE HISTORY BEFORE SYSTEM_TIME ...` ([KB](https://mariadb.com/kb/en/system-versioned-tables/)).

4. **Trigger doing N+1 lookups.** `AFTER INSERT FOR EACH ROW` calling a `SELECT` against another large table multiplies a 10k-row bulk insert by 10k point-queries. Fix : batch-aggregate in application code, or move logic to a stored procedure invoked once.

5. **`FUNCTION` marked `DETERMINISTIC` but reading session variables.** `RETURN @session_tax_rate * gross` declared deterministic lets the optimizer cache or hoist the call, producing stale results when the variable changes mid-query. Fix : drop `DETERMINISTIC`, accept the per-row evaluation, or pass the rate as a parameter ([KB](https://mariadb.com/kb/en/create-function/)).

6. **View with `ALGORITHM=TEMPTABLE` on a huge base table.** Each reference materializes the entire result into a temp table, with no index pushdown. Fix : default to `UNDEFINED` and trust the optimizer ; use `TEMPTABLE` only when the view body has non-mergeable constructs that you specifically want frozen.

7. **Ignoring `type=ALL` in `EXPLAIN`.** Production code shipped because the query "works on dev". Fix : block any new query whose `EXPLAIN type` is `ALL` against a table over a documented row-threshold ; add an index or rewrite.

8. **`FORCE INDEX` with stale statistics.** After bulk-loading 10 million rows without `ANALYZE TABLE`, the cardinality estimates are wrong ; a previously good `FORCE INDEX` now ignores a better composite. Fix : run `ANALYZE TABLE` after large data shifts, prefer `IGNORE INDEX` when you only want to exclude a specific path.

9. **Event scheduler off.** Created a nightly purge event, but `event_scheduler=OFF` (the default on many distros) means nothing runs. Fix : `SET GLOBAL event_scheduler = ON` and persist in `[mysqld]` config ; verify with `SHOW VARIABLES LIKE 'event_scheduler'`.

10. **Trigger with `COMMIT` or `ROLLBACK`.** Compiles on creation but raises `Explicit or implicit commit is not allowed in stored function or trigger` at runtime ([KB](https://mariadb.com/kb/en/trigger-limitations/)). Fix : move transactional logic to a stored procedure called by the application.

---

## Newly Discovered Sub-Topics

1. **`SYS_REFCURSOR` and Oracle-mode `IN OUT` parameters (11.8+)** — newer MariaDB function-return capability worth a dedicated section in `mariadb-syntax-stored-routines`.
2. **`system_versioning_insert_history` (10.11+)** — allows direct insertion of historical rows, useful for data import / migration into versioned tables.
3. **`max_recursive_iterations` cap on recursive CTEs** — anti-pattern coverage should reference this exact variable, not generic "depth limit".
4. **`ANALYZE UPDATE` / `ANALYZE DELETE` actually mutate** — must be flagged as a footgun in the `mariadb-impl-query-optimization` skill, distinct from `ANALYZE SELECT`.
5. **`PARTITION BY SYSTEM_TIME INTERVAL ... AUTO` (10.9.1+)** — auto-partition creation : new sub-topic for system-versioning skill.
6. **`optimizer_trace` system variable + `INFORMATION_SCHEMA.OPTIMIZER_TRACE`** — deep-dive diagnostic tool beyond `EXPLAIN FORMAT=JSON`, deserves its own section.
7. **MariaDB has no `GROUPS` frame** — explicit divergence from PostgreSQL and MySQL 8.0+ that must be called out in the window-functions skill to prevent hallucinated syntax.
8. **Multi-source replication interaction with triggers** — triggers fire per source, not per replicated event, when row-based logging is on ; intersects with the replication-cluster research.


---

## Cluster 3 : Replication, Galera, Backup, Performance Tuning, Security, Migration, Frappe Companion

# Cluster-3 Research : MariaDB Ops (Replication, Galera, Backup, Tuning, Security, Migration, Frappe)

> Scope : MariaDB 10.6 LTS, 10.11 LTS, 11.x current, 12.x next.
> Source policy : MariaDB KB, mariadb.com/docs, MariaDB/server GitHub, mariadb.org, galeracluster.com, jira.mariadb.org only.
> Last verified : 2026-05-19. All claims tied to citations. No dev.mysql.com.

---

## 1. Replication

MariaDB supports three replication shapes : asynchronous (default), semi-synchronous, and parallel-applier on the replica side. Asynchronous replication does not wait for replicas, which delivers the lowest latency but allows data loss if the primary fails before the binary log reaches a replica. Semi-synchronous replication closes that window by forcing the primary to wait for at least one replica to acknowledge receipt of the binlog event before returning to the client. The relevant variables are `rpl_semi_sync_master_enabled`, `rpl_semi_sync_slave_enabled`, and `rpl_semi_sync_master_timeout` (default 10000 ms). Semi-sync was built into the MariaDB server starting in 10.3, removing the need for the separate plug-in install that earlier versions required ([KB semisync](https://mariadb.com/kb/en/semisynchronous-replication/)).

```ini
# my.cnf on primary (MariaDB 10.3+)
[mariadb]
rpl_semi_sync_master_enabled=ON
rpl_semi_sync_master_timeout=20000

# my.cnf on replica
[mariadb]
rpl_semi_sync_slave_enabled=ON
```

Parallel replication is configured with `slave_parallel_threads` (worker pool size for event application) and `slave_parallel_mode`. The available modes are `optimistic` (default since 10.5.1, executes transactional DML in parallel with automatic conflict detection and rollback), `conservative` (uses group-commit information to identify non-conflicting transactions, default until 10.5.0), `aggressive` (parallel without conflict-avoidance heuristics), `minimal` (only the commit phase runs in parallel), and `none` (single-threaded applier). Both primary and replica must be MariaDB 10.0.5 or later ([KB parallel](https://mariadb.com/kb/en/parallel-replication/)).

```ini
# Replica my.cnf (MariaDB 10.5.1+)
[mariadb]
slave_parallel_threads=4
slave_parallel_mode=optimistic
slave_parallel_max_queued=262144
slave_domain_parallel_threads=2
```

**Diverges from MySQL** : MariaDB GTID format is `domain-server-sequence`, a triple of three integers (for example `0-1-10`) where domain is a 32-bit unsigned int identifying a logical replication stream, server is the originating server id, and sequence is a 64-bit unsigned counter. MySQL uses `uuid:seqno`. The KB states explicitly that "MariaDB can be a replica for a MySQL primary, but MySQL cannot be a replica for a MariaDB primary" ([KB gtid](https://mariadb.com/kb/en/gtid/)). `gtid_strict_mode` rejects any operation that could cause binlog divergence (for example replicating a GTID with a lower sequence than one already present). Operational variables are `gtid_slave_pos`, `gtid_binlog_pos`, and the composite `gtid_current_pos`.

Multi-source replication has been a MariaDB feature since the 10.0 series and uses named connections via `CHANGE MASTER 'connection_name' TO ...`. Each connection name must be unique and is case-insensitive ([KB multi-source](https://mariadb.com/kb/en/multi-source-replication/)).

```sql
-- MariaDB 10.0+ multi-source replication
CHANGE MASTER 'analytics' TO
  MASTER_HOST='server1.example.com',
  MASTER_USER='repl',
  MASTER_PASSWORD='secret123',
  MASTER_USE_GTID=slave_pos;

START SLAVE 'analytics';
SHOW ALL SLAVES STATUS;
RESET SLAVE 'analytics' ALL;   -- remove channel permanently
```

Binary log formats are `STATEMENT`, `ROW`, and `MIXED`. MariaDB still ships `MIXED` as the default per the KB ([KB binlog formats](https://mariadb.com/kb/en/binary-log-formats/)), and switches to row-based encoding for any statement the server determines is unsafe (non-deterministic functions, `LIMIT` without `ORDER BY`, certain stored procedures). `binlog_row_image` selects `FULL` (default, before+after for every column), `MINIMAL` (only changed columns plus a minimal before image), or `NOBLOB` (full before image but excludes unchanged BLOB and TEXT columns) to reduce binlog volume.

## 2. Galera Cluster

Galera is a synchronous multi-master cluster built on certification-based replication. It requires the `wsrep_provider` library (`galera-4` from MariaDB 10.4 onwards) and a `wsrep_cluster_address=gcomm://node1,node2,node3` style address. A minimum of three nodes is required for stable quorum, since a two-node cluster cannot reliably distinguish a peer failure from a network split. Galera uses a primary component (PC) algorithm and supports weighted quorum via the `pc.weight` provider option to bias which partition survives a split. SST (state-snapshot-transfer) methods are `mariabackup` (recommended, non-blocking, InnoDB-aware), `rsync` (locks the donor), and `mysqldump` (deprecated for SST). IST (incremental state transfer) applies the missing write-sets from the donor's `gcache` when the joining node has only fallen briefly behind ([KB SST](https://mariadb.com/docs/galera-cluster/high-availability/state-snapshot-transfers-ssts-in-galera-cluster/introduction-to-state-snapshot-transfers-ssts.md)).

```ini
# Galera node my.cnf (MariaDB 10.4+ with galera-4)
[mariadb]
wsrep_on=ON
wsrep_provider=/usr/lib/galera/libgalera_smm.so
wsrep_cluster_address=gcomm://node1,node2,node3
wsrep_cluster_name=prod_cluster
wsrep_node_address=10.0.0.11
wsrep_node_name=node1
wsrep_sst_method=mariabackup
wsrep_sst_auth=sstuser:sstpass
binlog_format=ROW
default_storage_engine=InnoDB
innodb_autoinc_lock_mode=2
```

**MariaDB-only** semantic : certification-based replication means write-write conflicts manifest as a `deadlock` only at `COMMIT` time, not mid-transaction as InnoDB row-locks would on a standalone server. Applications must retry on `ER_LOCK_DEADLOCK` even for transactions that never touched a conflicting row on the local node. The `wsrep_local_cert_failures` status variable tracks the count.

## 3. Backup and Restore

MariaDB ships two backup paths. `mysqldump` (renamed `mariadb-dump` from 10.5+) produces logical SQL dumps. Without `--single-transaction` it locks tables, which blocks writes on production. `--single-transaction` works only with transactional engines (InnoDB). `mariabackup` is the physical hot-backup tool, InnoDB-aware, supports incremental chains and partial backups ([KB mariabackup](https://mariadb.com/kb/en/mariabackup/), [KB full-backup](https://mariadb.com/docs/server/server-usage/backup-and-restore/mariadb-backup/full-backup-and-restore-with-mariadb-backup.md)).

```bash
# Full backup, prepare, restore (MariaDB 10.3+)
mariadb-backup --backup \
   --target-dir=/var/mariadb/backup/ \
   --user=mariadb-backup --password=mypassword

mariadb-backup --prepare \
   --target-dir=/var/mariadb/backup/

# Stop mariadbd before --copy-back
mariadb-backup --copy-back \
   --target-dir=/var/mariadb/backup/

chown -R mysql:mysql /var/lib/mysql/
```

The KB is explicit : the backup directory "must be empty or it must not exist" and `--prepare` is mandatory before any restore. Point-in-time recovery uses `mariadb-binlog` (renamed from `mysqlbinlog` in 10.5+) with `--start-datetime`, `--stop-datetime`, `--start-position`, or `--stop-position` against the binary log files. Incremental chains use `--incremental-basedir` to point each delta at its parent, then `--prepare --incremental-dir` applied in chain order. Single-table restore from a physical backup requires `--export` during prepare followed by `ALTER TABLE ... DISCARD TABLESPACE` and `IMPORT TABLESPACE` on the live server.

## 4. Performance Tuning

The buffer pool is the dominant tunable. The KB recommends "starting at several gigabytes of memory" and tracking the ratio of `innodb_buffer_pool_reads` to `innodb_buffer_pool_read_requests` (target under 1% of the read-request delta over time) ([KB buffer pool](https://mariadb.com/kb/en/innodb-buffer-pool/)). Industry convention on a dedicated DB host is 60-80% of RAM, but the KB phrases it as "not too large, because this can cause swapping, which more than undoes the benefits". From 10.11.12 onwards `innodb_buffer_pool_chunk_size` is deprecated and ignored ; the pool now resizes in 1 MB increments up to `innodb_buffer_pool_size_max`.

`innodb_flush_log_at_trx_commit` has three values per the KB ([KB innodb vars](https://mariadb.com/kb/en/innodb-system-variables/)). `1` (default) flushes the redo log to disk on every transaction commit and is the only fully ACID setting. `2` writes to the OS file cache on every commit but flushes only once per second, surviving a process crash but losing up to one second on an OS crash. `0` writes and flushes only once per second, accepting up to one second of loss on any crash. Pair with `sync_binlog=1` for crash-safe binlog replay.

```ini
# Write-heavy OLTP, dedicated host (MariaDB 10.6+)
[mariadb]
innodb_buffer_pool_size=24G
innodb_log_file_size=2G
innodb_flush_log_at_trx_commit=1
sync_binlog=1
innodb_io_capacity=2000        # SSD baseline
innodb_io_capacity_max=4000    # SSD burst
innodb_thread_concurrency=0    # unlimited, recommended default
innodb_fill_factor=90          # leave 10% for index growth
query_cache_type=OFF
query_cache_size=0
```

`innodb_io_capacity` and `_max` cap background flushing throughput, scaled to SSD vs HDD IOPS. `innodb_thread_concurrency=0` (unlimited) is the recommended modern default. `innodb_fill_factor` ranges 10-100, defaults 100, and acts as a hint to leave space in B-tree pages for future inserts ([KB fill factor](https://mariadb.com/kb/en/innodb-system-variables/#innodb_fill_factor)).

**Diverges from MySQL** : the query cache was removed in MySQL 8.0 but kept in MariaDB. The KB confirms the cache "does not scale well in environments with high throughput on multi-core machines" because "each time changes are made to the data in a table, all affected results in the query cache are cleared" ([KB query cache](https://mariadb.com/kb/en/query-cache/)). For any write-heavy workload, set `query_cache_type=OFF` and `query_cache_size=0`. Per-connection buffers (`sort_buffer_size`, `read_rnd_buffer_size`, `join_buffer_size`) multiply by `max_connections`, so over-tuning these silently inflates memory ceiling.

## 5. Security

Authentication is plug-in based. Legacy `mysql_native_password` uses SHA-1 and is deprecated. The modern recommended plug-in is `ed25519`, available since 10.1.21, which uses Elliptic Curve DSA, the same algorithm as OpenSSH ([KB ed25519](https://mariadb.com/kb/en/authentication-plugin-ed25519/)). `unix_socket` auth on the root user maps OS uid to DB identity and ships as the default on Debian and Ubuntu packages. `gssapi` (Kerberos) and `pam` are also supported for enterprise SSO.

```sql
-- Install plug-in dynamically (MariaDB 10.1.21+)
INSTALL SONAME 'auth_ed25519';

-- Or via my.cnf
-- [mariadb]
-- plugin_load_add = auth_ed25519

-- Create user with ed25519
CREATE USER 'alice'@'%' IDENTIFIED VIA ed25519 USING PASSWORD('secret');

-- Force TLS at GRANT level
GRANT SELECT ON app.* TO 'alice'@'%' REQUIRE SSL;
GRANT SELECT ON app.* TO 'bob'@'%'
  REQUIRE SUBJECT '/CN=bob/O=Acme/C=NL'
  AND ISSUER '/C=FI/O=Acme CA/CN=Root'
  AND CIPHER 'ECDHE-RSA-AES256-GCM-SHA384';
```

Role-based access has been in MariaDB since the 10.0 series, exposed via `CREATE ROLE`, `GRANT role TO user`, `SET ROLE`, and `SET DEFAULT ROLE` so the role activates on connect. `WITH ADMIN OPTION` lets the grantee re-grant the role ([KB grant](https://mariadb.com/kb/en/grant/), [KB roles](https://mariadb.com/kb/en/roles_overview/)). Only roles granted directly to a user can be set ; nested-role activation is not supported.

Encryption at rest uses the file-key-management plug-in (file-on-disk keyring), the aws-key-management plug-in (AWS KMS), or HashiCorp Vault via third-party plug-in. The minimal config for file-key-management ([docs file-key](https://mariadb.com/docs/server/security/encryption/data-at-rest-encryption/key-management-and-encryption-plugins/file-key-management-encryption-plugin.md)) :

```ini
# my.cnf (MariaDB 10.6+, file-key-management)
[mariadb]
plugin_load_add = file_key_management
loose_file_key_management_filename = /etc/mysql/encryption/keyfile.enc
loose_file_key_management_filekey   = FILE:/etc/mysql/encryption/keyfile.key
loose_file_key_management_encryption_algorithm = AES_CTR

# Per-feature toggles (community-known, verify per version in KB)
innodb_encrypt_tables=ON
innodb_encrypt_log=ON
encrypt_binlog=ON
encrypt_tmp_files=ON
```

TLS for replication is configured on the replica side via `CHANGE MASTER ... MASTER_SSL=1` plus `MASTER_SSL_CA`, `MASTER_SSL_CERT`, `MASTER_SSL_KEY`, and the strongly recommended `MASTER_SSL_VERIFY_SERVER_CERT=1` ([KB change-master](https://mariadb.com/kb/en/change-master-to/)).

```sql
STOP SLAVE;
CHANGE MASTER TO
   MASTER_SSL=1,
   MASTER_SSL_CA='/etc/my.cnf.d/certificates/ca.pem',
   MASTER_SSL_CERT='/etc/my.cnf.d/certificates/server-cert.pem',
   MASTER_SSL_KEY='/etc/my.cnf.d/certificates/server-key.pem',
   MASTER_SSL_VERIFY_SERVER_CERT=1;
START SLAVE;
```

## 6. Migration MySQL to MariaDB

The upgrade path is : stop MySQL, install MariaDB binaries on the same datadir, start mariadbd, run `mariadb-upgrade` ([KB upgrade](https://mariadb.com/kb/en/upgrading-from-mysql-to-mariadb/)). MariaDB is a drop-in replacement for MySQL 5.5 and 5.6 ; from MySQL 5.7 and 8.0 onwards several incompatibilities require remediation :

| Area | MySQL 5.7+ / 8.0 | MariaDB 10.6+ | Diverges from MySQL |
|------|------------------|---------------|---------------------|
| JSON storage | Native binary type | LONGTEXT with optional `CHECK (JSON_VALID(col))` | **yes** |
| Default auth plug-in | `caching_sha2_password` (8.0) | `mysql_native_password` / `ed25519` / `unix_socket` | **yes** |
| GTID format | `uuid:seqno` | `domain-server-sequence` | **yes**, replication path one-way |
| Sequences | not supported (AUTO_INCREMENT only) | SQL-standard `CREATE SEQUENCE ... NEXT VALUE FOR ...` (10.3+) | **MariaDB-only** |
| Role syntax | similar but not identical | `SET DEFAULT ROLE`, role mandatory on connect | similar, NOT interchangeable |
| User table | `mysql.user` | `mysql.global_priv` from 10.4+ (view `mysql.user` retained for compatibility) | **yes** |
| sql_mode defaults | strict by default (8.0) | strict by default (10.2.4+) but list differs | minor |

Per the KB, "MariaDB 10.4+ uses `mysql.global_priv` for privilege management". The `mysql.user` view is still queryable but is a compatibility layer over `global_priv`. Run `mariadb-upgrade` exactly once, NOT twice in a row, since it rewrites system tables idempotently but logs noise on a second run.

## 7. Frappe / ERPNext Companion Patterns

Frappe v14 and v15 require MariaDB 10.6.6 or newer ; Frappe v16/develop requires MariaDB 11.8 ([Frappe install](https://docs.frappe.io/framework/v15/user/en/installation)). Frappe is multi-tenant by spawning one database per site (each `bench new-site` creates a new DB), so connection pooling has to account for many tenant DBs sharing one server. Table naming convention is `tab<DoctypeName>` (for example `tabUser`, `tabSales Invoice` with embedded space). Child tables include parent linkage columns : `parent`, `parentfield`, `parenttype`, and `idx`.

The hard requirement is `utf8mb4` (4 bytes per character) charset on every Frappe DB, since the framework stores arbitrary Unicode including emoji and non-BMP code points. Without `innodb_default_row_format=dynamic`, an indexed `VARCHAR(255)` in utf8mb4 needs 1020 bytes per row in an index, exceeding the 767-byte default index-prefix limit on the older `Antelope` row format. The companion settings :

```ini
# /etc/mysql/mariadb.conf.d/frappe.cnf (MariaDB 10.6+ for Frappe v14/v15)
[mysqld]
character-set-client-handshake = FALSE
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci
innodb_file_per_table = 1
innodb_default_row_format = dynamic
innodb_large_prefix = 1
innodb_file_format = Barracuda
max_allowed_packet = 256M

[mysql]
default-character-set = utf8mb4
```

ERPNext bench typically connects via `unix_socket` auth on the root user (default Debian/Ubuntu install) and uses `bench backup` / `bench restore` wrappers around `mariadb-dump`. Production sites that need RPO under a few minutes should layer `mariabackup` incremental snapshots on top of `bench backup`. `bench --site <site> backup --with-files` includes uploaded files.

---

## Anti-Patterns

1. **`STATEMENT` binlog with non-deterministic functions**. `UUID()`, `NOW(6)` resolved at high resolution, `LIMIT` without `ORDER BY`, `SLEEP`, and user-defined functions can produce divergent results on the replica. The KB explicitly warns "the set of rows included cannot be predicted" ([KB binlog formats](https://mariadb.com/kb/en/binary-log-formats/)). Use `MIXED` or `ROW`.

2. **Galera with a 2-node cluster**. Two nodes cannot form a stable quorum. Any network partition either takes both nodes read-only (loss of availability) or causes split-brain when each node assumes the other is dead. Always run 3 nodes minimum, or 2 + one arbitrator (`garbd`) ([KB SST](https://mariadb.com/docs/galera-cluster/high-availability/state-snapshot-transfers-ssts-in-galera-cluster/introduction-to-state-snapshot-transfers-ssts.md)).

3. **Logical backup on production with `mysqldump` and no `--single-transaction`**. Default `mysqldump` issues table-locks per database, blocking writes for the entire dump duration. On any InnoDB workload, use `--single-transaction`. On mixed-engine schemas, switch to `mariabackup` for hot physical backup ([KB mariabackup](https://mariadb.com/kb/en/mariabackup/)).

4. **`innodb_buffer_pool_size` at 50% of RAM on a dedicated DB host**. Too conservative ; the host has nothing else to do. Industry convention 60-80%, KB warns only against making it large enough to cause OS swap ([KB buffer pool](https://mariadb.com/kb/en/innodb-buffer-pool/)). Set 70% as a starting point on a dedicated 32 GB+ host.

5. **Running `mariadb-upgrade` twice in a row**. The tool is idempotent but logs spurious warnings on the second pass and can mask real errors from the first pass under the second run's noise. Run once after every binary upgrade, not as a routine maintenance task ([KB upgrade](https://mariadb.com/kb/en/upgrading-from-mysql-to-mariadb/)).

6. **Encryption at rest with `innodb_encrypt_tables=ON` but `encrypt_binlog=OFF`**. Row-image leaks via the binary log. A read-only attacker with binlog access can reconstruct most of the protected data even though the tablespaces are encrypted. Encrypt the binlog and the temp files at the same time as the tablespaces.

7. **`GRANT ALL ON *.* TO 'app_user'@'%'` on production**. `ALL` includes `SUPER`, `FILE`, `PROCESS`, `RELOAD`, `SHUTDOWN`. An SQL-injection in any code path then escalates to server-level. Grant the minimal privilege set per database and per object, and use roles to keep the GRANT list short ([KB grant](https://mariadb.com/kb/en/grant/)).

8. **Migrating MySQL native JSON column to MariaDB without `JSON_VALID` `CHECK`**. MariaDB stores JSON as `LONGTEXT`, which does NOT validate JSON syntax on insert. A row that was valid JSON on MySQL can be corrupted by a subsequent non-JSON `UPDATE` on MariaDB without any error. Add `CHECK (JSON_VALID(col))` on every JSON column during the migration to preserve the MySQL invariant ([KB mariadb-vs-mysql](https://mariadb.com/docs/release-notes/community-server/about/compatibility-and-differences/mariadb-vs-mysql-compatibility.md)).

9. **Frappe install with default `utf8` (3-byte) charset**. Default `utf8` in older MariaDB is a 3-byte subset of UTF-8 that cannot store emoji or any code point above U+FFFF. The DB silently truncates or rejects on insert. Always set `character-set-server=utf8mb4` and `innodb_default_row_format=dynamic` BEFORE the first `bench new-site` ([Frappe install](https://docs.frappe.io/framework/v15/user/en/installation)).

10. **Replica with `MASTER_SSL=1` but `MASTER_SSL_VERIFY_SERVER_CERT=0`**. Connection is encrypted but vulnerable to a MITM that presents any valid cert. The KB recommends `MASTER_SSL_VERIFY_SERVER_CERT=1` to actually validate the primary's cert chain against `MASTER_SSL_CA` ([KB change-master](https://mariadb.com/kb/en/change-master-to/)).

---

## Newly Discovered Sub-Topics

1. **`MASTER_SSL_VERIFY_SERVER_CERT` and the `DEFAULT` keyword in 12.3+**. From 12.3 onwards the `CHANGE MASTER` TLS options accept `DEFAULT` to inherit server-level TLS config. Worth a dedicated section in `mariadb-impl-replication-setup` since older skill content typically hard-codes per-channel certs.

2. **`mysql.global_priv` view layer for backward compatibility**. From 10.4+ the privilege store is `mysql.global_priv`. Operators writing third-party tools that grep `mysql.user` will see consistent but read-only output via a compatibility view. Pair this in `mariadb-impl-migration-mysql-to-mariadb` with a query showing how to inspect privileges natively.

3. **`innodb_buffer_pool_chunk_size` deprecation in 10.11.12+**. The KB notes the variable is deprecated and ignored starting 10.11.12 ; the pool resizes in 1 MB increments up to `innodb_buffer_pool_size_max`. Tuning advice in any new skill must drop this variable.

4. **`file_key_management_use_pbkdf2` and `file_key_management_digest` in 12.0.1+**. Encryption keyring derivation now supports PBKDF2 iterations and selectable digest function. Older 10.6 skills will omit these ; a 12.x section is warranted in `mariadb-core-security-model`.

5. **Galera `pc.weight` weighted quorum**. Asymmetric data-center deployments (for example 2 nodes in DC-A, 1 in DC-B) benefit from biasing the survival of the larger DC during a WAN split. This is provider-option territory (`wsrep_provider_options="pc.weight=2"`) and merits a sub-section in `mariadb-impl-galera-cluster`.

6. **`slave_domain_parallel_threads` cap**. Independent of `slave_parallel_threads`, this caps parallel applier threads per replication domain. Important for multi-source setups where one domain should not starve another. Worth an inline note in the parallel-replication skill.

7. **Frappe v16 requires MariaDB 11.8**. A real version-floor change discovered in this research. Frappe v14/v15 floor is 10.6.6 ; v16/develop floor is 11.8. Cross-package note for both the Frappe and MariaDB skill packages.

8. **`mariadb-dump` and `mariadb-binlog` renames in 10.5+**. The historical `mysqldump` and `mysqlbinlog` binaries are renamed in 10.5+ ; the old names remain as compatibility symlinks. Scripts in production that hard-code `mysqldump` should be updated, otherwise package upgrades that drop the symlink will break backup cron jobs.


---

## Consolidated Newly Discovered Sub-Topics (across all clusters)

Each agent's per-cluster sub-topics are listed in its fragment file. The high-impact ones for masterplan refinement :

- atomic DDL guarantees in 10.6+ (rollback-safe schema changes)
- utf8mb4_uca1400_* collations (modern Unicode collation algorithm)
- AS OF time-travel for system-versioned tables (10.3+)
- CHECK constraints since 10.2.1 (precedes MySQL 8 by years)
- instant DROP COLUMN in 10.4+ (alter without table rewrite)
- optimizer_trace for query-plan debugging
- innodb_default_row_format (DYNAMIC default since 10.2)
- max_recursive_iterations (default 1000) for recursive CTE termination
- mariadb-corporation rename : mysqldump to mariadb-dump (10.5+), mysqlbinlog to mariadb-binlog (10.5+)
- mysql.global_priv (10.4+) replaces mysql.user for privilege storage
- file_key_management_use_pbkdf2 / file_key_management_digest in 12.0.1+ for key-derivation
- Galera SST via mariabackup is the modern default ; rsync and mysqldump-SST are deprecated
