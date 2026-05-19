# methods : the 10-dimension schema-review procedure

This file is the complete, deterministic review procedure. Each dimension is a set of `IF condition THEN severity` rules. Apply every dimension to every table in the supplied schema. Never skip a dimension.

Severity definitions :

- **BLOCKER** : the schema must not ship. Verdict becomes FAIL.
- **WARNING** : the schema works but carries a known, documented cost or footgun. Verdict becomes PASS WITH WARNINGS.
- **SUGGESTION** : a maintainability or stylistic improvement. Does not change a PASS verdict.
- **NOTE** : informational only ; no severity, included for context.

---

## How to run a review

1. Parse the supplied input. It is one of : a set of `CREATE TABLE` statements, a migration script (`ALTER TABLE` / `CREATE TABLE`), or a schema dump.
2. For each table, evaluate dimensions 1 through 10 in order.
3. Collect every finding into one list.
4. Sort findings : BLOCKER, then WARNING, then SUGGESTION, then NOTE.
5. Emit the findings table (see SKILL.md "Output Format").
6. Emit the verdict line.

Never invent findings. Every finding maps to an explicit rule below.

---

## Dimension 1 : Storage engine

Canonical skill : `mariadb-core-storage-engines`

- IF `ENGINE=MyISAM` AND the table has a `FOREIGN KEY`, OR appears in a transactional workload, OR has columns implying concurrent writes (status, updated_at) THEN **BLOCKER**. Fix : `ALTER TABLE t ENGINE=InnoDB;`. MyISAM has table-level locking, no transactions, no crash recovery, no foreign keys.
- IF `ENGINE=MEMORY` (HEAP) AND the table is not an explicit ephemeral cache THEN **BLOCKER**. MEMORY loses all rows on server restart.
- IF `ENGINE=Aria` on user-facing data THEN **WARNING**. Aria is crash-safe but has no transactions or MVCC ; it is intended for system tables and on-disk temp tables. Fix : use InnoDB for user data.
- IF no `ENGINE` clause is present THEN **NOTE** : the engine defaults to InnoDB since MariaDB 10.2. Acceptable, but recommend an explicit `ENGINE=InnoDB` for clarity.
- IF `ENGINE=ColumnStore` AND the table is queried by single-row primary-key lookups THEN **WARNING**. ColumnStore is analytical-only and cannot serve OLTP efficiently.
- IF `ENGINE=InnoDB` THEN **PASS** for this dimension.

---

## Dimension 2 : Primary key

Canonical skill : `mariadb-impl-schema-design`

- IF a table has no `PRIMARY KEY` and no equivalent `UNIQUE NOT NULL` THEN **BLOCKER**. InnoDB synthesises a hidden 6-byte clustered key that cannot be referenced, indexed, or used by replication efficiently ; row-based replication on a PK-less table forces a full table scan per change.
- IF the primary key is `CHAR(36)` or `VARCHAR(36)` holding a UUID string THEN **WARNING**. A 36-character UUID in utf8mb4 costs up to 144 bytes and is copied into every secondary index. Fix : store as `BINARY(16)` with `UUID_TO_BIN(uuid, 1)` so the time-low / time-high swap preserves B-tree clustering.
- IF the primary key is a random (non-time-ordered) UUID even as `BINARY(16)` THEN **SUGGESTION** : prefer UUIDv7 / ULID for time-ordered keys that preserve insert locality and reduce page splits.
- IF the primary key is `BIGINT UNSIGNED AUTO_INCREMENT` for a single-node workload THEN **PASS**.
- IF a wide composite key (4+ columns, total width over ~40 bytes) is used as the clustered PRIMARY KEY THEN **SUGGESTION** : a wide clustered key bloats every secondary index ; consider a narrow surrogate PK plus a UNIQUE constraint on the natural key.

---

## Dimension 3 : Indexing

Canonical skill : `mariadb-syntax-indexing`

- IF a composite index `(a, b, c)` exists but the query pattern filters by equality on `b` (not `a`) THEN **WARNING**. The leftmost-prefix rule means `(a, b, c)` cannot serve `WHERE b = ?`. Fix : reorder columns so the highest-selectivity equality predicate is leftmost, range predicates last.
- IF two indexes exist where one is a strict leftmost-prefix of the other (for example `INDEX(a)` and `INDEX(a, b)`) THEN **SUGGESTION** : the shorter index is redundant ; the longer index already serves `WHERE a = ?`. Fix : drop the prefix index.
- IF a `FOREIGN KEY` column has no index THEN **WARNING**. MariaDB auto-creates an index for a FK if none exists, but an explicit, well-named index documents intent and lets you control column order in a composite. Verify the FK column is the leftmost column of some index ; a parent-table `DELETE` does a full child scan otherwise.
- IF a `TEXT` or `BLOB` column is indexed without a prefix length (`INDEX(col)` instead of `INDEX(col(20))`) THEN **BLOCKER** : MariaDB rejects an unbounded index on TEXT / BLOB with error 1170.
- IF no index supports a stated `ORDER BY` or `GROUP BY` and the table is large THEN **SUGGESTION** : add a composite index matching filter columns plus sort columns to avoid `Using filesort` / `Using temporary`.

---

## Dimension 4 : Charset and collation

Canonical skill : `mariadb-errors-encoding-and-collation`

- IF a text column or table uses `CHARSET=utf8` or `CHARSET=utf8mb3` THEN **WARNING**. `utf8` in MariaDB is the 3-byte `utf8mb3` ; it cannot store 4-byte characters (emoji, many CJK extensions) and triggers `Incorrect string value` on insert. Fix : `utf8mb4`.
- IF a table uses `CHARSET=latin1` with no documented reason THEN **WARNING**. On stock 10.6 / 10.11 binaries the server default is still `latin1`, so a `CREATE TABLE` with no explicit charset silently locks in `latin1_swedish_ci`. Fix : declare `DEFAULT CHARSET=utf8mb4` per table and set `character-set-server=utf8mb4`.
- IF columns in tables that are joined or compared use different collations THEN **WARNING** : a collation mismatch on a join predicate prevents index use (`Illegal mix of collations` or a silent full scan). Fix : use one collation consistently, for example `utf8mb4_uca1400_ai_ci`.
- IF the schema uses `utf8mb4` consistently THEN **PASS**.

---

## Dimension 5 : Normalization

Canonical skill : `mariadb-impl-schema-design`

- IF a table has a repeating group of columns (`phone1`, `phone2`, `phone3` or `addr_line1`, `addr_line2`, `addr_line3` used as independent values) THEN **WARNING** : this is an un-normalized repeating group. Fix : extract into a child table with a foreign key.
- IF a single column stores a comma-separated or delimited list of values THEN **WARNING** : non-atomic columns break indexing, joins, and referential integrity. Fix : a junction table.
- IF the same descriptive attribute is copied across many rows of a table (a denormalized copy) with no stated performance reason THEN **SUGGESTION** : note the redundancy ; denormalization is valid but must be a deliberate, documented choice.
- IF a table mixes two distinct entity types behind a `type` discriminator with many always-NULL columns per type THEN **SUGGESTION** : consider table-per-type or a cleaner single-table-inheritance design.

---

## Dimension 6 : Multi-tenant

Canonical skill : `mariadb-impl-schema-design`

- Detect a tenant column : a column named `tenant_id`, `org_id`, `organisation_id`, `company`, `account_id`, or similar that appears across most tables.
- IF a tenant column is detected AND a tenant-scoped table has no index whose leftmost column is the tenant column THEN **WARNING** : every tenant-scoped query (`WHERE tenant_id = ? AND ...`) does a full scan or relies on a non-leading index. Fix : make `tenant_id` the leftmost column of the relevant composite indexes.
- IF a tenant column is detected AND the `PRIMARY KEY` of a tenant-scoped table does not lead with the tenant column THEN **SUGGESTION** : a tenant-leading clustered key co-locates each tenant's rows on disk and improves cache locality. This is a recommendation, not a blocker.
- IF a tenant column is detected AND a `FOREIGN KEY` crosses tenants (a child row can reference a parent of a different tenant) THEN **WARNING** : add the tenant column to the FK or enforce tenant equality with a CHECK / trigger.

---

## Dimension 7 : Naming

Canonical skill : `mariadb-impl-schema-design`

- IF table or column names mix conventions (some `CamelCase`, some `snake_case`) THEN **SUGGESTION** : pick one convention, prefer `snake_case` to avoid case-folding issues on case-insensitive filesystems.
- IF an identifier is a MariaDB reserved word (`order`, `group`, `key`, `status` is not reserved but `condition` is, etc.) used without backticks in the DDL THEN **BLOCKER** : the DDL fails to parse. Fix : rename, or backtick everywhere (renaming is preferred).
- IF table names match the `tab<Doctype>` pattern (`tabUser`, `tabSales Invoice`) THEN **NOTE** : this is a Frappe / ERPNext-generated schema. Do NOT recommend renaming ; the framework owns these names. Review the other dimensions but treat naming as out of scope.
- IF a column name does not describe its content (`field1`, `data`, `value` on a wide table) THEN **SUGGESTION** : use descriptive names.

---

## Dimension 8 : Constraints

Canonical skill : `mariadb-syntax-check-constraints`

- IF a column has a clearly finite value domain (a `status`, `state`, `kind`, `type` column with a known small set) and no `CHECK` constraint and no `ENUM` THEN **SUGGESTION** : add `CHECK (status IN ('open','closed','cancelled'))` to enforce the invariant in the engine, not the application.
- IF a child table holds a column that obviously references a parent table's key (`customer_id` in an `orders` table) and there is no `FOREIGN KEY` THEN **WARNING** : referential integrity is left to application code. Fix : add the `FOREIGN KEY ... REFERENCES ...` constraint.
- IF a numeric column that must be non-negative (`quantity`, `price`, `age`) has no `CHECK (col >= 0)` and is not `UNSIGNED` THEN **SUGGESTION** : enforce the bound with `UNSIGNED` or a `CHECK`.
- CHECK constraints are enforced in MariaDB 10.2.1+ ; before 10.2.1 they were parsed and ignored. Assume enforcement on all target versions (10.6+).

---

## Dimension 9 : Data types

Canonical skill : `mariadb-syntax-sql-ddl`

- IF a money / price / amount / balance column uses `FLOAT` or `DOUBLE` THEN **BLOCKER** : binary floating point cannot represent decimal fractions exactly, so sums and comparisons drift. Fix : `DECIMAL(p, s)`, for example `DECIMAL(13, 2)`.
- IF a column with a small bounded set of short values (country code, currency code, ISO language) uses `VARCHAR(255)` THEN **SUGGESTION** : size it to the real maximum (`CHAR(2)`, `CHAR(3)`) ; oversized VARCHAR inflates temp-table and sort-buffer allocations.
- IF a column that is always short (under ~255 chars : a name, a slug, an email) uses `TEXT` THEN **SUGGESTION** : use `VARCHAR(n)`. `TEXT` cannot have an inline `DEFAULT` (before 10.2 ; allowed as an expression default 10.2+) and is stored off-page, costing an extra read.
- IF a boolean flag uses `VARCHAR` holding `'Y'` / `'N'` or `'true'` / `'false'` THEN **SUGGESTION** : use `TINYINT(1)` / `BOOLEAN`.
- IF a timestamp column uses `INT` holding a Unix epoch THEN **SUGGESTION** : `TIMESTAMP` or `DATETIME` is time-zone aware and self-documenting ; an `INT` epoch loses sub-second precision and readability.
- IF an `ENUM` is used for a value set that changes frequently THEN **SUGGESTION** : changing an `ENUM` requires an `ALTER TABLE` ; a lookup table with a FK is more flexible.

---

## Dimension 10 : JSON

Canonical skill : `mariadb-syntax-json`

MariaDB JSON is a LONGTEXT alias, not native binary. Use `CHECK (JSON_VALID(col))` for structure ; use functional indexes on virtual columns for index access.

- IF a `JSON` column has no `CHECK (JSON_VALID(col))` constraint THEN **WARNING** : because JSON is LONGTEXT, `INSERT INTO t (doc) VALUES ('not even json')` succeeds silently and breaks every downstream `JSON_EXTRACT` / `JSON_VALUE`. Fix : add `CHECK (JSON_VALID(doc))` at table creation.
- IF a `JSON` column is queried by a JSON path in a `WHERE` clause and no virtual column with a functional index exists for that path THEN **SUGGESTION** : MariaDB cannot index inside a JSON document directly. Fix : add a `VIRTUAL` generated column over `JSON_VALUE(doc, '$.path')` and index that column.
- IF a `JSON` column stores data that has a fixed, known shape (always the same keys) THEN **SUGGESTION** : a fixed shape is better modelled as real typed columns ; reserve JSON for genuinely variable or sparse data.
- IF `JSON` is used purely for MySQL interoperability THEN **NOTE** : that is valid ; MariaDB JSON output is standards-compliant text.
