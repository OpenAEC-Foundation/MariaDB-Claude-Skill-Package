# MariaDB Schema Design : Methods and Reference Tables

Complete reference matrices for every decision dimension in a new MariaDB schema. All claims verified against `mariadb.com/kb/en/` on 2026-05-19.

## 1. Primary-key type matrix

| PK type | Storage | Sort property | Use when | Avoid when |
|---|---|---|---|---|
| `BIGINT UNSIGNED AUTO_INCREMENT` | 8 bytes | monotonic | single writer, no external requirement | distributed writes, app-side ID generation |
| `INT UNSIGNED AUTO_INCREMENT` | 4 bytes | monotonic | small tables (< 4.2B rows ever) | any chance of exceeding 2^32 rows |
| `BINARY(16)` UUIDv4 (with swap) | 16 bytes | pseudo-time-sortable with `UUID_TO_BIN(uuid, 1)` | distributed writes, opaque external id | strict ordering required |
| `BINARY(16)` UUIDv7 / ULID | 16 bytes | time-sortable (millis precision) | distributed writes + time ordering matters | uniform random required |
| `CHAR(36)` UUID-text | 36 bytes (144 in utf8mb4 secondary index) | text-sort, random under v4 | NEVER recommended on InnoDB | always |
| Composite (multi-column) | sum of column bytes | by leftmost column | natural keys with clear hierarchy | when surrogate is simpler |
| `VARCHAR(140)` slug (Frappe `name`) | variable | text-sort | Frappe doctype compatibility | non-Frappe schemas |

### UUID storage breakdown

| Encoding | Bytes per row in clustered index | Bytes per row in every secondary index (utf8mb4) |
|---|---|---|
| `BIGINT` | 8 | 8 |
| `BINARY(16)` UUID | 16 | 16 |
| `CHAR(36)` UUID text | 36 (latin1) / 144 (utf8mb4) | 36 (latin1) / 144 (utf8mb4) |

On a 100M-row table with five secondary indexes : `BIGINT` = ~4 GB indexes total ; `BINARY(16)` = ~8 GB ; `CHAR(36)` in utf8mb4 = ~72 GB. The math is unforgiving.

### `UUID_TO_BIN` swap-flag

```sql
-- 10.6+ : second argument 1 swaps time-low and time-high fields
-- of a UUIDv1, making the first 6 bytes monotonic for B-tree locality
SET @uuid = UUID();                              -- 11ee-... textual
INSERT INTO t (id) VALUES (UUID_TO_BIN(@uuid, 1));
-- Read back :
SELECT BIN_TO_UUID(id, 1) FROM t WHERE id = UUID_TO_BIN(@uuid, 1);
```

The swap flag is the difference between random-insert behaviour (page splits everywhere) and clustered-tail-insert behaviour (locality preserved).

## 2. Multi-tenant pattern matrix

| Pattern | Tenant count ceiling | Isolation | Cross-tenant queries | Backup granularity | Best for |
|---|---|---|---|---|---|
| schema-per-tenant | ~hundreds | SQL-level (separate DB) | hard (UNION ALL) | per-DB `mariadb-dump` | Frappe, regulated isolation |
| row-level `tenant_id` | unbounded | logical (every query filters) | trivial (WHERE tenant_id IN ...) | per-row export | SaaS at scale |
| Hybrid (small tenants in shared schema, big tenants in own schema) | tens to hundreds | mixed | requires routing layer | mixed | rare ; high engineering cost |

### Schema-per-tenant scaling pain points

- `mysql.tables` and `mysql.columns` system tables grow linearly with (tenants * tables-per-tenant). Past ~1000 tenants the server's information_schema queries slow visibly.
- `--open-files-limit` must accommodate (tenants * indexed-tables * 2) file descriptors with `innodb_file_per_table=ON`.
- Each tenant DB requires its own connection pool entry in the application : 100 tenants * 10 pool size = 1000 connections per app instance.

### Row-level required indexing

```sql
-- Every common access path MUST start with tenant_id
KEY ix_tenant_customer (tenant_id, customer_id),
KEY ix_tenant_status   (tenant_id, status, created_at),
-- Composite uniqueness MUST include tenant_id
UNIQUE KEY uq_tenant_invoice_number (tenant_id, invoice_number)
```

A `UNIQUE KEY (invoice_number)` without `tenant_id` makes invoice numbers globally unique across all tenants : almost always a bug.

## 3. ROW_FORMAT comparison

| Format | Default since | INSTANT ADD/DROP COLUMN | Index prefix (16K page) | Disk footprint | CPU cost |
|---|---|---|---|---|---|
| `DYNAMIC` | 10.2 (default) | yes | 3072 bytes | baseline | baseline |
| `COMPRESSED` | (opt-in) | NO (rejected) | 3072 bytes | -30% to -70% | +20% to +200% |
| `COMPACT` | pre-10.2 (deprecated default) | partial | 767 bytes | -5% vs DYNAMIC | low |
| `REDUNDANT` | MySQL 4.x compat | no | 767 bytes | +20% vs COMPACT | low |

The `innodb_default_row_format` server variable controls the default. On any 10.2+ install assume DYNAMIC unless explicitly downgraded.

### Index prefix limit consequences

| Row format | utf8mb4 VARCHAR safe index width | Common breaking case |
|---|---|---|
| DYNAMIC | up to 768 chars (3072 / 4) | none in practice |
| COMPACT | 191 chars (767 / 4) | `VARCHAR(255)` index fails with error 1071 |
| REDUNDANT | 191 chars (767 / 4) | as above |

Frappe ships VARCHAR(255) indexes everywhere ; on COMPACT every `bench new-site` would fail. This is why `innodb_default_row_format=dynamic` + `innodb_large_prefix=1` are non-negotiable in the Frappe / ERPNext companion config.

## 4. Character set and collation matrix

| MariaDB version | Server default charset | Server default collation | Notes |
|---|---|---|---|
| 10.6-LTS | `latin1` (stock binary) | `latin1_swedish_ci` | Debian/Ubuntu vendor builds patch to utf8mb4 |
| 10.11-LTS | `latin1` (stock binary) | `latin1_swedish_ci` | same caveat |
| 11.0 - 11.5 | `latin1` | `latin1_swedish_ci` | |
| 11.6+ | `utf8mb4` | `utf8mb4_uca1400_ai_ci` | migration completed |
| 12.x | `utf8mb4` | `utf8mb4_uca1400_ai_ci` | unchanged |

### Recommended per-table declaration (works on all versions)

```sql
... ENGINE=InnoDB
    ROW_FORMAT=DYNAMIC
    DEFAULT CHARSET=utf8mb4
    COLLATE=utf8mb4_uca1400_ai_ci;
```

### Collation suffix decoder

| Suffix | Meaning |
|---|---|
| `ai` | accent-insensitive (e at = é at sorting) |
| `as` | accent-sensitive |
| `ci` | case-insensitive (abc = ABC) |
| `cs` | case-sensitive |
| `_bin` | byte-by-byte binary comparison ; fastest, no locale awareness |
| `_uca1400` | Unicode Collation Algorithm v14.0 (since 10.6) ; supersedes `_general_ci` and `_unicode_ci` |
| `_general_ci` | legacy, faster, incorrect sort for non-trivial Unicode (eszett, Turkish dotted i, ...) |
| `_unicode_ci` | legacy UCA v4.0 (2003 Unicode) ; closer to correct than general_ci but ancient |

### `utf8` is `utf8mb3` (3-byte) and DOES NOT store emoji

The keyword `utf8` is a deprecated alias for `utf8mb3`, which is the 3-byte BMP-only subset of UTF-8. Any character above U+FFFF (emoji, mathematical symbols, many CJK extension blocks) cannot be stored. The DB raises error 1366 `Incorrect string value` or silently truncates depending on `sql_mode`. Always use `utf8mb4` explicitly.

## 5. Foreign-key reference options

| Option | ON DELETE / ON UPDATE behaviour |
|---|---|
| `RESTRICT` (default) | parent DML rejected if any child row references it ; error 1451 / 1452 |
| `NO ACTION` | synonym for `RESTRICT` in MariaDB |
| `CASCADE` | parent DML propagates to all matching children |
| `SET NULL` | child FK column(s) set to NULL ; requires the column to be NULLable |
| `SET DEFAULT` | parsed but not enforced ; KB-documented limitation, behaves as RESTRICT |

### FK constraints summary

- InnoDB only (no FK support in Aria, MyISAM, MEMORY, ColumnStore).
- Forbidden on partitioned tables : "Partitioned tables cannot contain foreign keys, and cannot be referenced by a foreign key."
- Referenced columns must form a BTREE index ; HASH, RTREE, FULLTEXT cannot back a FK.
- TEXT and BLOB columns cannot be FK because they exceed the index-prefix limit.
- Generated PERSISTENT columns cannot be the indexed FK column ; CASCADE/SET NULL prohibited on PERSISTENT.
- FK name must be unique per database (per-table from 12.1+ per KB).

## 6. Generated columns : VIRTUAL vs PERSISTENT

| Aspect | VIRTUAL | PERSISTENT (alias STORED) |
|---|---|---|
| Storage | none (computed on read) | materialised on disk |
| Indexable | yes (10.6+ for most expressions) | yes |
| FK indexable | NO (InnoDB rejects FK on VIRTUAL index) | yes (10.6+) |
| Row-based replication | replica recomputes | replica reads from binlog |
| ALGORITHM=COPY required for indexing | yes (KB-documented) | no |
| Rebuild cost on column expression change | minimal (no data) | full table rewrite |

### When to choose which

- VIRTUAL : default. JSON path extraction, computed enums, derived flags read occasionally.
- PERSISTENT : heavy read workload with expensive expressions, FK target columns, replication consistency demands.

## 7. Frappe / ERPNext naming convention reference

| Element | Convention | Example |
|---|---|---|
| Doctype table | `tab` + Doctype name (literal prefix) | `tabUser`, `` `tabSales Invoice` `` (backtick for spaces) |
| Primary key | column `name` VARCHAR(140) | not BIGINT, not auto-increment |
| Created timestamp | `creation` DATETIME(6) | |
| Modified timestamp | `modified` DATETIME(6) | |
| Modifier | `modified_by` VARCHAR(140) | |
| Owner | `owner` VARCHAR(140) | |
| Document state | `docstatus` INT (0=draft, 1=submitted, 2=cancelled) | |
| Sort index | `idx` INT | |
| Child-table parent link | `parent` VARCHAR(140) | references `<parenttype>.name` |
| Child-table parent field | `parentfield` VARCHAR(140) | the Frappe docfield name in the parent doctype |
| Child-table parent type | `parenttype` VARCHAR(140) | doctype of the parent (allows polymorphism) |

### Frappe version floor matrix

| Frappe major | Minimum MariaDB | Notes |
|---|---|---|
| v14 | 10.6.6 | |
| v15 | 10.6.6 | |
| v16 / develop | 11.8 | per Frappe install docs |

## 8. JSON column reference (LONGTEXT-alias caveat)

MariaDB JSON is an alias for LONGTEXT. There is no native binary storage like MySQL 5.7.8+. Consequences :

- `CHECK (JSON_VALID(col))` is the only structural validator. Without it, any string lands in a JSON column.
- Indexing requires a generated (typically VIRTUAL) column over `JSON_VALUE(col, '$.path')` plus a regular index on that generated column.
- Row-based replication of JSON columns between MySQL master and MariaDB replica is unsafe : MySQL's binary JSON format does not match MariaDB's text storage. Use statement-based, or convert columns to TEXT during migration.
- Comparison semantics differ from MySQL : MariaDB compares JSON values as their text representation (string comparison) ; MySQL compares them structurally.

### Indexable virtual-column pattern

```sql
ALTER TABLE t
  ADD COLUMN extracted_key VARCHAR(64)
    AS (JSON_VALUE(payload, '$.key')) VIRTUAL,
  ADD KEY ix_extracted_key (extracted_key),
  ALGORITHM=COPY;
```

`ALGORITHM=COPY` is required when adding an index on a VIRTUAL column (KB-documented limitation since 10.2). The table is rewritten ; plan for it on large tables.

## 9. Required server-variable settings

```ini
# /etc/mysql/mariadb.conf.d/50-modern-defaults.cnf
[mysqld]
character-set-server         = utf8mb4
collation-server             = utf8mb4_uca1400_ai_ci
innodb_default_row_format    = dynamic
innodb_large_prefix          = 1        # ensures index prefix > 767 bytes
innodb_file_per_table        = 1
innodb_file_format           = Barracuda
character-set-client-handshake = FALSE

[client]
default-character-set = utf8mb4

[mysql]
default-character-set = utf8mb4
```

Verify after restart :
```sql
SHOW VARIABLES LIKE 'character_set%';
SHOW VARIABLES LIKE 'collation%';
SHOW VARIABLES LIKE 'innodb_default_row_format';
SHOW VARIABLES LIKE 'innodb_large_prefix';
```

## 10. CREATE TABLE template skeleton

```sql
CREATE TABLE <name> (
  -- 1. Primary key (BIGINT UNSIGNED default, BINARY(16) for distributed)
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  -- 2. Multi-tenant column if applicable (always first after id)
  tenant_id BIGINT UNSIGNED NOT NULL,
  -- 3. Business columns
  ...,
  -- 4. JSON metadata if needed (with CHECK)
  metadata JSON NULL CHECK (JSON_VALID(metadata)),
  -- 5. Audit timestamps
  created_at TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  updated_at TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
                                  ON UPDATE CURRENT_TIMESTAMP(6),
  PRIMARY KEY (id),
  -- 6. Indexes lead with tenant_id when present
  KEY ix_tenant_... (tenant_id, ...),
  -- 7. Foreign keys (InnoDB only ; never on partitioned tables)
  CONSTRAINT fk_..._tenant FOREIGN KEY (tenant_id) REFERENCES tenant(id)
    ON DELETE RESTRICT
) ENGINE=InnoDB
  ROW_FORMAT=DYNAMIC
  DEFAULT CHARSET=utf8mb4
  COLLATE=utf8mb4_uca1400_ai_ci;
```
