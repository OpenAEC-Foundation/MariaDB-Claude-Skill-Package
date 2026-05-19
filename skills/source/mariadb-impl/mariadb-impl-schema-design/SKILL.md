---
name: mariadb-impl-schema-design
description: >
  Use when designing a new MariaDB schema, choosing primary-key type, structuring multi-tenant tables, picking ROW_FORMAT or CHARSET, or applying Frappe/ERPNext table-naming conventions.
  Prevents the common mistake of UUID-text PK on InnoDB (insert fragmentation), schema-per-tenant scaling without quota, utf8 instead of utf8mb4 (3-byte trap), or composite-key in wrong column-order.
  Covers PK type choice (BIGINT AUTO_INCREMENT vs ULID/UUIDv7 vs UUID-string), multi-tenant patterns (schema-per-tenant vs row-tenant-id), ROW_FORMAT=DYNAMIC default, CHARSET utf8mb4_uca1400 default 10.6+, foreign-key constraints, generated columns, Frappe naming convention (tab<DoctypeName>, parent/parentfield/parenttype/idx columns), required InnoDB ROW_FORMAT=DYNAMIC + large_prefix for utf8mb4 indexes.
  Keywords: schema design, primary key, BIGINT AUTO_INCREMENT, UUID, ULID, UUIDv7, multi-tenant, schema per tenant, row tenant id, ROW_FORMAT, DYNAMIC, COMPRESSED, utf8mb4, utf8mb4_uca1400, foreign key, generated column, virtual column, tab DoctypeName, Frappe, ERPNext, parent parentfield, large_prefix, innodb_large_prefix, multi tenancy in mariadb, how do I model a tenant, my index is too long, 1071 specified key was too long, emoji breaks insert, latin1 lock-in, char36 uuid, JSON longtext, getting started with schema design
license: MIT
compatibility: "Designed for Claude Code. Requires MariaDB 10.6-LTS, 10.11-LTS, 11.x, 12.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# MariaDB Schema Design

Decisions made at `CREATE TABLE` time outlive any application layer. This skill covers primary-key type choice, multi-tenant data layout, ROW_FORMAT, CHARSET / collation, foreign-key constraints, generated columns, and the Frappe / ERPNext companion conventions. Read this before the first DDL of a new project.

## Quick Reference

- ALWAYS default to `BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY` on single-writer InnoDB tables : monotonic, B-tree friendly, 8 bytes per row.
- ALWAYS use `BINARY(16)` (not `CHAR(36)`) when a UUID is required, with `UUID_TO_BIN(uuid, 1)` to apply the swap-flag for time-low/high ordering ; CHAR(36) bloats every secondary index by ~9x.
- ALWAYS prefer ULID or UUIDv7 over UUIDv4 for distributed-write designs : time-prefixed sortable keys preserve B-tree locality.
- ALWAYS declare `ENGINE=InnoDB ROW_FORMAT=DYNAMIC DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_uca1400_ai_ci` per-table on 10.6-LTS and 10.11-LTS : stock binaries still default to `latin1` and emoji silently fail.
- ALWAYS set `innodb_default_row_format=dynamic` and `innodb_large_prefix=1` at server level when running Frappe / ERPNext or any utf8mb4 schema with `VARCHAR(255)` indexes ; the 767-byte legacy prefix limit blocks utf8mb4 indexes (255 * 4 = 1020 bytes) on COMPACT row format.
- NEVER use `utf8` as the charset declaration : it is a 3-byte alias for `utf8mb3` and cannot store characters above U+FFFF (emoji, many CJK, mathematical symbols).
- NEVER pick `CHAR(36)` UUID-text PK on a write-heavy InnoDB table : random insert positions in the clustered index fragment pages and force page-splits on every batch.
- NEVER apply schema-per-tenant beyond a few hundred tenants : `mysql` metadata, open-file-handles, and per-DB connection overhead degrade non-linearly past ~1000 schemas.
- NEVER design a multi-tenant row layout without `INDEX(tenant_id, ...)` on every common query path : the leftmost-prefix rule means single-column indexes ignoring `tenant_id` cannot serve the planner.
- NEVER apply `ON DELETE CASCADE` on a huge child table : a single parent DELETE acquires X-locks on every matching child row and may run for hours.
- MariaDB JSON is a LONGTEXT alias, not native binary (per D-010). Use `CHECK (JSON_VALID(col))` for structure, and functional indexes on VIRTUAL columns for index access (`ADD COLUMN x AS (JSON_VALUE(doc,'$.k')) VIRTUAL, ADD KEY ix(x)`).
- Frappe / ERPNext tables follow `tab<DoctypeName>` naming (literal `tab` prefix, with embedded spaces allowed inside backticks, for example `` `tabSales Invoice` ``). Child tables include `parent`, `parentfield`, `parenttype`, `idx` columns.

## Decision Trees

### Primary-key type selection

```
Single MariaDB writer, no sharding planned, no external id requirement ?
   YES : BIGINT UNSIGNED AUTO_INCREMENT (default ; 8 bytes ; monotonic)
   NO  : continue

External id leaked to clients or used as URL slug ?
   YES : add a secondary UUID column, keep BIGINT PK internal
   NO  : continue

Distributed-write (Galera multi-master, app-side ID generation, offline-first sync) ?
   YES, time-ordered ID acceptable : BINARY(16) ULID or UUIDv7
   YES, fully random UUIDv4 required : BINARY(16) UUID with UUID_TO_BIN(uuid, 1) swap-flag
   NO  : BIGINT AUTO_INCREMENT
```

NEVER use `CHAR(36)`. 36 ASCII bytes are 36 in latin1 but 144 bytes in utf8mb4 secondary indexes. On a 100M-row table this means ~14 GB of index vs ~2 GB for BINARY(16).

### Multi-tenant layout selection

```
Tenants < 50, tenants need full SQL-level isolation, no cross-tenant queries ?
   YES : schema-per-tenant (one DATABASE per tenant ; Frappe model)
   NO  : continue

Tenants 50-1000, cross-tenant analytics needed, soft-isolation acceptable ?
   YES : row-level tenant_id with INDEX(tenant_id, ...) on every query path
   NO  : continue

Tenants > 1000 ?
   row-level tenant_id is the only viable pattern at this scale ;
   schema-per-tenant exhausts mysql metadata and open-file-handles
```

Per the Frappe install pattern : `bench new-site` creates one MariaDB DATABASE per Frappe site. This is schema-per-tenant ; it works up to ~hundreds of sites per server, then connection-pool and metadata costs dominate.

### ROW_FORMAT selection

```
Default OLTP table with mixed columns ?
   YES : ROW_FORMAT=DYNAMIC (default since 10.2 ; off-page LONG VARCHAR/BLOB)

Archive table, write-once-read-rarely, disk constrained ?
   YES : ROW_FORMAT=COMPRESSED with KEY_BLOCK_SIZE=8 (CPU cost on every read)

Pre-10.2 legacy table being upgraded ?
   ALTER to ROW_FORMAT=DYNAMIC ; rebuild required, plan a maintenance window

INSTANT ADD/DROP COLUMN needed often ?
   ROW_FORMAT=DYNAMIC (COMPRESSED rejects INSTANT operations)
```

`innodb_default_row_format=dynamic` is the modern server default. DYNAMIC supports a 3072-byte index prefix per the KB ; COMPACT / REDUNDANT cap at 767 bytes and break utf8mb4 indexes on VARCHAR(255).

### CHARSET and collation selection

```
New MariaDB 10.6-LTS or 10.11-LTS install on stock binary ?
   Server default is still latin1 ; declare DEFAULT CHARSET=utf8mb4 per table.
   Add character-set-server=utf8mb4 to my.cnf.

MariaDB 11.6+ ?
   Server default migrated to utf8mb4 ; per-table declaration still recommended
   for portability.

Need accent-insensitive case-insensitive collation, Unicode 14.0 (10.6+) ?
   utf8mb4_uca1400_ai_ci (default on 11.6+)

Need accent-sensitive case-insensitive ?
   utf8mb4_uca1400_as_ci

Need binary-comparison sort (case + accent sensitive, fastest, locale-agnostic) ?
   utf8mb4_bin
```

Suffix decoder : `ai` = accent-insensitive, `as` = accent-sensitive, `ci` = case-insensitive, `cs` = case-sensitive, `_uca1400` = Unicode Collation Algorithm v14.0 (Unicode 14.0 ordering).

### Foreign-key action selection

```
Child row meaningless without parent ?
   ON DELETE CASCADE (small child tables only)

Child row should survive parent removal but lose linkage ?
   ON DELETE SET NULL (FK column must be NULLable)

Parent removal forbidden while children exist ?
   ON DELETE RESTRICT (default) ; application surfaces the 1451 error

Updates to the parent key column should propagate ?
   ON UPDATE CASCADE (rare ; immutable surrogate PKs avoid this)
```

`SET DEFAULT` is parsed by MariaDB but not enforced (per the KB `foreign-keys` page) ; treat it as RESTRICT in design. Foreign keys require InnoDB and are forbidden on partitioned tables.

## Patterns

### Default production-grade table

```sql
-- 10.6+ : safe per-table declaration that survives stock-binary latin1 defaults
CREATE TABLE customer (
  id           BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  external_id  BINARY(16)      NOT NULL,
  email        VARCHAR(255)    NOT NULL,
  display_name VARCHAR(255)    NOT NULL,
  status       ENUM('active','pending','suspended') NOT NULL DEFAULT 'pending',
  metadata     JSON            NULL CHECK (JSON_VALID(metadata)),
  created_at   TIMESTAMP(6)    NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  updated_at   TIMESTAMP(6)    NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
                                       ON UPDATE CURRENT_TIMESTAMP(6),
  PRIMARY KEY (id),
  UNIQUE KEY uq_external_id (external_id),
  UNIQUE KEY uq_email       (email),
  KEY ix_status_created     (status, created_at)
) ENGINE=InnoDB
  ROW_FORMAT=DYNAMIC
  DEFAULT CHARSET=utf8mb4
  COLLATE=utf8mb4_uca1400_ai_ci;
```

`external_id BINARY(16)` is the URL-safe ULID / UUIDv7 ; internal `id BIGINT` keeps the clustered index small. `UNIQUE KEY uq_email` works because DYNAMIC permits 3072-byte index prefixes ; on COMPACT this would error 1071 for utf8mb4.

### Row-level multi-tenant table

```sql
-- All queries MUST start WHERE tenant_id = ? to use the index
CREATE TABLE invoice (
  id          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  tenant_id   BIGINT UNSIGNED NOT NULL,
  customer_id BIGINT UNSIGNED NOT NULL,
  amount      DECIMAL(12,2)   NOT NULL,
  status      ENUM('draft','sent','paid','void') NOT NULL DEFAULT 'draft',
  created_at  TIMESTAMP(6)    NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  PRIMARY KEY (id),
  KEY ix_tenant_customer (tenant_id, customer_id),
  KEY ix_tenant_status   (tenant_id, status, created_at),
  CONSTRAINT fk_invoice_customer FOREIGN KEY (customer_id)
    REFERENCES customer(id) ON DELETE RESTRICT,
  CONSTRAINT fk_invoice_tenant   FOREIGN KEY (tenant_id)
    REFERENCES tenant(id)   ON DELETE RESTRICT
) ENGINE=InnoDB ROW_FORMAT=DYNAMIC DEFAULT CHARSET=utf8mb4
  COLLATE=utf8mb4_uca1400_ai_ci;
```

`tenant_id` is the leftmost column on every index because every query filters by tenant first. Without this, the optimiser scans the whole table even if the rest of the predicate is selective.

### UUIDv7 / ULID primary key

```sql
-- 10.6+ : time-prefixed sortable PK for distributed writers
CREATE TABLE event (
  id          BINARY(16)      NOT NULL,
  aggregate   BINARY(16)      NOT NULL,
  payload     JSON            NOT NULL CHECK (JSON_VALID(payload)),
  occurred_at TIMESTAMP(6)    NOT NULL,
  PRIMARY KEY (id),
  KEY ix_aggregate_time (aggregate, occurred_at)
) ENGINE=InnoDB ROW_FORMAT=DYNAMIC DEFAULT CHARSET=utf8mb4
  COLLATE=utf8mb4_uca1400_ai_ci;

-- Application or MariaDB 11.x UUID()-based generation, stored as BINARY(16)
-- ULID example : 01H7Z1G5N4QF7XK0RJ8Y8D9C3W (encoded as 16 raw bytes)
```

Time-prefix in the first 48 bits keeps writes near the tail of the clustered index, avoiding the random-insert page-split pattern of UUIDv4. NEVER store as `CHAR(26)` (ULID) or `CHAR(36)` (UUID) when index size matters.

### Schema-per-tenant (Frappe pattern)

```sql
-- Pattern : one DATABASE per tenant ; isolation is at the DB level
CREATE DATABASE `tenant_acme`
  DEFAULT CHARSET=utf8mb4
  DEFAULT COLLATE=utf8mb4_uca1400_ai_ci;

CREATE USER 'acme_app'@'%' IDENTIFIED BY '<strong-secret>';
GRANT ALL ON `tenant_acme`.* TO 'acme_app'@'%';
FLUSH PRIVILEGES;
```

Limits :
- Stop scaling this past ~1000 databases per server : `mysql` schema tables grow, `--open-files-limit` must be raised, connection pools must keep one pool per tenant.
- Cross-tenant analytical queries require either replication into a single warehouse DB or careful `UNION ALL` across `<tenant>.<table>` references.

### Frappe / ERPNext companion table

```sql
-- v15 floor : MariaDB 10.6.6+. v16 floor : MariaDB 11.8.
-- Parent doctype : `tabUser` (literal `tab` prefix is part of every Frappe table)
CREATE TABLE `tabUser` (
  name         VARCHAR(140) NOT NULL,    -- Frappe's PK is always the slug `name`
  creation     DATETIME(6)  NULL,
  modified     DATETIME(6)  NULL,
  modified_by  VARCHAR(140) NULL,
  owner        VARCHAR(140) NULL,
  docstatus    INT          NOT NULL DEFAULT 0,
  idx          INT          NOT NULL DEFAULT 0,
  email        VARCHAR(140) NULL,
  enabled      INT          NOT NULL DEFAULT 1,
  PRIMARY KEY (name),
  KEY modified (modified)
) ENGINE=InnoDB ROW_FORMAT=DYNAMIC DEFAULT CHARSET=utf8mb4
  COLLATE=utf8mb4_unicode_ci;

-- Child table for a doctype : standard Frappe child columns are mandatory
CREATE TABLE `tabUser Role` (
  name         VARCHAR(140) NOT NULL,
  parent       VARCHAR(140) NULL,        -- FK by convention to tabUser.name
  parentfield  VARCHAR(140) NULL,        -- which docfield in the parent doctype
  parenttype   VARCHAR(140) NULL,        -- doctype name of the parent
  idx          INT          NOT NULL DEFAULT 0,
  role         VARCHAR(140) NULL,
  PRIMARY KEY (name),
  KEY parent   (parent, parentfield, parenttype)
) ENGINE=InnoDB ROW_FORMAT=DYNAMIC DEFAULT CHARSET=utf8mb4
  COLLATE=utf8mb4_unicode_ci;
```

Required server settings before `bench new-site` :
```ini
[mysqld]
character-set-server         = utf8mb4
collation-server             = utf8mb4_unicode_ci
innodb_default_row_format    = dynamic
innodb_large_prefix          = 1
innodb_file_per_table        = 1
character-set-client-handshake = FALSE
```
Without `innodb_default_row_format=dynamic` the framework's `VARCHAR(255)` indexes hit the 767-byte prefix limit and `bench new-site` fails with error 1071.

### JSON column with functional index

```sql
-- 10.6+ : MariaDB JSON is LONGTEXT (per D-010) ; validate + virtual + index
CREATE TABLE document (
  id         BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  tenant_id  BIGINT UNSIGNED NOT NULL,
  body       JSON            NOT NULL CHECK (JSON_VALID(body)),
  doc_type   VARCHAR(64)     AS (JSON_VALUE(body, '$.type')) VIRTUAL,
  PRIMARY KEY (id),
  KEY ix_tenant_type (tenant_id, doc_type)
) ENGINE=InnoDB ROW_FORMAT=DYNAMIC DEFAULT CHARSET=utf8mb4
  COLLATE=utf8mb4_uca1400_ai_ci;
```

Indexing JSON requires the virtual-column detour : MariaDB cannot index expressions directly. VIRTUAL re-evaluates on read but the index is materialised. PERSISTENT would store the value on disk too ; only choose PERSISTENT when reads are heavy and recomputation cost matters.

### Archive table with COMPRESSED row format

```sql
-- 10.6+ : trade CPU for disk on a write-once log table
CREATE TABLE audit_log_2024 (
  id          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  occurred_at TIMESTAMP(6)    NOT NULL,
  actor_id    BIGINT UNSIGNED NOT NULL,
  payload     JSON            NOT NULL CHECK (JSON_VALID(payload)),
  PRIMARY KEY (id),
  KEY ix_time_actor (occurred_at, actor_id)
) ENGINE=InnoDB
  ROW_FORMAT=COMPRESSED KEY_BLOCK_SIZE=8
  DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_uca1400_ai_ci;
```

`KEY_BLOCK_SIZE=8` halves the default 16K page. Trade-off : every read decompresses ; INSTANT ALTER is rejected ; CPU usage rises. Use only when the table is queried far less often than it is written.

## Cross-References

- `mariadb-syntax-sql-ddl` for `ALTER TABLE` algorithms, sequences, INSTANT ADD/DROP COLUMN, atomic DDL.
- `mariadb-syntax-indexing` for leftmost-prefix rule, prefix indexes, descending indexes, ignored indexes.
- `mariadb-syntax-json` for JSON_VALUE / JSON_QUERY / JSON_TABLE and LONGTEXT-alias details.
- `mariadb-syntax-check-constraints` for CHECK semantics (10.2+ enforced, parsed-but-ignored before).
- `mariadb-core-storage-engines` for InnoDB vs Aria vs other engines and their row-format defaults.
- `mariadb-impl-migration-mysql-to-mariadb` for MySQL native-JSON to MariaDB LONGTEXT-JSON conversion.
- `mariadb-errors-encoding-and-collation` for "Incorrect string value" diagnosis and recovery.

## Reference Files

- `references/methods.md` : decision tables for PK type, ROW_FORMAT trade-offs, charset/collation matrix per version, Frappe naming patterns, FK action semantics, generated-column flavours.
- `references/examples.md` : 10+ end-to-end CREATE TABLE patterns covering every scenario discussed here.
- `references/anti-patterns.md` : 9 anti-patterns from real schemas with cause, symptom, and fix.

## Sources

Verified against : `mariadb.com/kb/en/data-types/`, `mariadb.com/kb/en/character-sets-and-collations/`, `mariadb.com/kb/en/foreign-keys/`, `mariadb.com/kb/en/innodb-row-formats/`, `mariadb.com/kb/en/innodb-dynamic-row-format/`, `mariadb.com/kb/en/innodb-compressed-row-format/`, `mariadb.com/kb/en/generated-columns/`, `mariadb.com/kb/en/json-data-type/`, `docs.frappe.io/framework/v15/user/en/installation`. Last verified : 2026-05-19.
