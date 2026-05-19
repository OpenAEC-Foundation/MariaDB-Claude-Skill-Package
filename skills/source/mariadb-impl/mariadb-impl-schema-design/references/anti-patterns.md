# MariaDB Schema Design : Anti-Patterns

Nine patterns from real production schemas, with symptom, root cause, and verified fix. Every claim verified against `mariadb.com/kb/en/` on 2026-05-19.

## AP-01 : `CHAR(36)` UUID-text primary key on InnoDB

### Anti-pattern

```sql
CREATE TABLE event (
  id        CHAR(36)   NOT NULL,                            -- '01ee-...'
  body      JSON       NOT NULL,
  PRIMARY KEY (id),
  KEY ix_created (created_at)
) ENGINE=InnoDB
  DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_uca1400_ai_ci;
```

### Why it fails

- 36 ASCII bytes become 144 bytes in every secondary index because `utf8mb4` reserves 4 bytes per char.
- UUIDv4 values are random : every INSERT lands in a random clustered-index page, forcing page splits and page-cache misses.
- On a 100M-row table with five secondary indexes the size delta vs `BINARY(16)` is roughly +60 GB.
- Range scans by `id` are meaningless because the values are unsorted.

### Symptoms

- INSERT throughput collapses non-linearly past ~10M rows.
- `SHOW ENGINE INNODB STATUS\G` shows persistent `Buffer pool hit rate` below 99%.
- Backup size grows much faster than row count.

### Fix

```sql
-- Switch to BINARY(16) with swap-flag (UUIDv1) or use UUIDv7 / ULID
CREATE TABLE event (
  id   BINARY(16) NOT NULL,                                  -- UUID_TO_BIN(uuid, 1)
  body JSON       NOT NULL CHECK (JSON_VALID(body)),
  created_at TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  PRIMARY KEY (id),
  KEY ix_created (created_at)
) ENGINE=InnoDB ROW_FORMAT=DYNAMIC
  DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_uca1400_ai_ci;
```

If id leaks to clients, `SELECT BIN_TO_UUID(id, 1) AS uuid FROM event;` returns the textual form. Migration from CHAR(36) requires a rewrite ; use `pt-online-schema-change` for zero downtime on large tables.

## AP-02 : `utf8` (a.k.a. `utf8mb3`) charset on user-input text

### Anti-pattern

```sql
CREATE TABLE comment (
  id   INT AUTO_INCREMENT PRIMARY KEY,
  body TEXT
) ENGINE=InnoDB DEFAULT CHARSET=utf8;                        -- 3-byte alias
```

### Why it fails

- `utf8` in MariaDB is a deprecated alias for `utf8mb3`, the 3-byte BMP-only subset of UTF-8.
- Any code point above U+FFFF (emoji, mathematical symbols, many CJK extensions) is rejected or truncated.
- With strict `sql_mode`, INSERT fails with error 1366 `Incorrect string value`. Without strict mode, the value is silently truncated at the first non-BMP byte.

### Symptoms

- Users report "my emoji disappears" or "save fails when I paste this character".
- Errors `Incorrect string value: '\xF0\x9F\x98\x80...' for column 'body'` for any 4-byte UTF-8 sequence.

### Fix

```sql
ALTER TABLE comment
  CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_uca1400_ai_ci,
  ALGORITHM=COPY, LOCK=SHARED;
```

Also set server-level `character-set-server=utf8mb4` and `collation-server=utf8mb4_uca1400_ai_ci` in `my.cnf` so new tables inherit the correct charset.

## AP-03 : `ROW_FORMAT=COMPACT` with utf8mb4 indexes (error 1071)

### Anti-pattern

```sql
CREATE TABLE doc (
  id    INT AUTO_INCREMENT PRIMARY KEY,
  title VARCHAR(255) NOT NULL,
  KEY ix_title (title)
) ENGINE=InnoDB ROW_FORMAT=COMPACT
  DEFAULT CHARSET=utf8mb4;
```

### Why it fails

- COMPACT row format limits index prefixes to 767 bytes.
- `VARCHAR(255)` in utf8mb4 occupies up to 255 * 4 = 1020 bytes per row in an index.
- The server rejects the index creation with error 1071 `Specified key was too long ; max key length is 767 bytes`.

### Symptoms

- `bench new-site` (Frappe / ERPNext) fails immediately with error 1071.
- ORM-generated migrations fail on the first VARCHAR(255) unique key.

### Fix

```sql
-- Per-table : declare DYNAMIC explicitly
... ROW_FORMAT=DYNAMIC ...

-- Server-wide : set in my.cnf and restart
[mysqld]
innodb_default_row_format = dynamic
innodb_large_prefix       = 1
```

DYNAMIC permits index prefixes up to 3072 bytes on a 16K page (per the InnoDB DYNAMIC row format KB), comfortably accommodating utf8mb4 VARCHAR(255) indexes.

## AP-04 : Multi-tenant row design without `tenant_id` in the leftmost index column

### Anti-pattern

```sql
CREATE TABLE invoice (
  id          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  tenant_id   BIGINT UNSIGNED NOT NULL,
  customer_id BIGINT UNSIGNED NOT NULL,
  KEY ix_customer (customer_id),                             -- missing tenant_id
  KEY ix_status   (status)                                   -- missing tenant_id
);
```

### Why it fails

- The leftmost-prefix rule means `KEY (customer_id)` cannot answer `WHERE tenant_id = ? AND customer_id = ?` selectively.
- The optimiser falls back to a full-table scan filtered by tenant_id, or picks the customer_id index and scans within it ignoring tenant boundaries.
- Result : queries get slower per-tenant as the GLOBAL row count grows, instead of scaling with per-tenant row count.

### Symptoms

- Tenant A complains the app is "much slower since tenant B onboarded last month".
- `EXPLAIN` shows `rows = <approximately total table size>` for tenant-scoped queries.

### Fix

```sql
-- Every common index leads with tenant_id
KEY ix_tenant_customer (tenant_id, customer_id),
KEY ix_tenant_status   (tenant_id, status, created_at),
UNIQUE KEY uq_tenant_number (tenant_id, number)              -- tenant-scoped uniqueness
```

Verify with `EXPLAIN` : rows should equal the per-tenant slice, not the table total.

## AP-05 : `ON DELETE CASCADE` on a huge child table

### Anti-pattern

```sql
CREATE TABLE order_doc (
  id          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  customer_id BIGINT UNSIGNED NOT NULL,
  CONSTRAINT fk_order_customer FOREIGN KEY (customer_id)
    REFERENCES customer(id) ON DELETE CASCADE                 -- millions of rows per customer
);
```

### Why it fails

- `DELETE FROM customer WHERE id = 42` acquires X-locks on every order row owned by customer 42.
- On a customer with millions of orders the delete runs for minutes to hours, blocks all writes referencing those rows, and saturates the undo log.
- A cascading delete cannot be paused or chunked from SQL ; the entire chain runs in a single transaction.

### Symptoms

- Single-row DELETE on the parent table takes hours and blocks unrelated workloads.
- Undo-log size grows without bound during the cascade ; `SHOW ENGINE INNODB STATUS` shows the cascade as an ever-growing trx.

### Fix

```sql
-- Use RESTRICT and clean up children explicitly in batches
CONSTRAINT fk_order_customer FOREIGN KEY (customer_id)
  REFERENCES customer(id) ON DELETE RESTRICT
```

Application-side deletion :
```sql
-- Chunked delete with explicit batching
DELETE FROM order_doc WHERE customer_id = 42 ORDER BY id LIMIT 5000;
-- Repeat until no rows affected, then
DELETE FROM customer WHERE id = 42;
```

## AP-06 : Schema-per-tenant beyond ~1000 databases

### Anti-pattern

```sql
-- One DATABASE per tenant ; design works for first ~100 tenants
CREATE DATABASE tenant_0001;
CREATE DATABASE tenant_0002;
-- ...
CREATE DATABASE tenant_9999;
```

### Why it fails

- `mysql` system tables (`tables`, `columns`, `statistics`) grow linearly with (databases * tables * columns). At 10000 tenants * 50 tables * 20 columns = 10M rows in `information_schema.columns`.
- `--open-files-limit` must cover every InnoDB `.ibd` file with `innodb_file_per_table=ON`. 10000 tenants * 50 tables = 500000 file descriptors.
- Application connection pools require one pool per tenant : memory and pool-warmup overhead scale linearly.
- Tools like `mysqldump --all-databases` traverse every schema serially ; backup time grows linearly.

### Symptoms

- `information_schema` queries (used by ORMs at startup) take seconds instead of milliseconds.
- `open_files_limit` warnings in error log ; server refuses to open new tables.
- Connection pool exhaustion under normal traffic.

### Fix

Convert to a row-level multi-tenant design (see Example 4 in `examples.md`) before crossing ~500 tenants. Migrate by dumping each tenant DB, adding a `tenant_id` column during load, and merging into a shared schema.

Hybrid intermediate : keep large premium tenants in their own DB, pool small free-tier tenants into a shared schema.

## AP-07 : Generated PERSISTENT column for a value used only at read time

### Anti-pattern

```sql
-- 'name_lower' written to disk on every INSERT/UPDATE,
-- but the app only reads it in case-insensitive search queries
CREATE TABLE customer (
  id         BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  full_name  VARCHAR(255) NOT NULL,
  name_lower VARCHAR(255) AS (LOWER(full_name)) PERSISTENT,
  KEY ix_name_lower (name_lower)
);
```

### Why it fails

- PERSISTENT materialises the column on disk. Every row pays the storage + write-amplification cost.
- VIRTUAL would compute the value on read, costing CPU per row but zero disk.
- The KB-documented restriction "adding an index on a VIRTUAL column requires ALGORITHM=COPY" applies here, but is a one-time cost vs the per-row PERSISTENT overhead.

### Symptoms

- Table size on disk is materially larger than rows + indexes would predict.
- INSERT throughput is lower than identical schema without the generated column.

### Fix

```sql
-- VIRTUAL : computed on read, indexed via the virtual column
CREATE TABLE customer (
  id         BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  full_name  VARCHAR(255) NOT NULL,
  name_lower VARCHAR(255) AS (LOWER(full_name)) VIRTUAL,
  KEY ix_name_lower (name_lower)
);
```

When to keep PERSISTENT : (a) the column is the FK target (VIRTUAL cannot back an FK), (b) the expression is expensive AND reads dominate writes by orders of magnitude, (c) row-based replication consistency demands materialised values.

## AP-08 : Missing `innodb_large_prefix` on legacy MariaDB for utf8mb4 indexes

### Anti-pattern

```ini
# Stock my.cnf on legacy 10.x install
[mysqld]
character-set-server = utf8mb4
# innodb_large_prefix not set (defaults vary by version)
# innodb_default_row_format not set
```

```sql
CREATE TABLE customer (
  id    INT AUTO_INCREMENT PRIMARY KEY,
  email VARCHAR(255) NOT NULL,
  UNIQUE KEY uq_email (email)
);
```

### Why it fails

- Without `innodb_large_prefix=1`, the InnoDB index-prefix cap remains 767 bytes regardless of row format claims elsewhere.
- `VARCHAR(255)` in utf8mb4 occupies 1020 bytes : error 1071 on index creation.
- The Frappe install docs explicitly call this out as the most common first-install failure.

### Symptoms

- `bench new-site` fails at the first table creation with error 1071.
- Generic ORM migrations fail on `VARCHAR(255) UNIQUE` on stock 10.x with old defaults.

### Fix

```ini
[mysqld]
character-set-server          = utf8mb4
collation-server              = utf8mb4_uca1400_ai_ci    # or utf8mb4_unicode_ci for Frappe
innodb_default_row_format     = dynamic
innodb_large_prefix           = 1
innodb_file_per_table         = 1
innodb_file_format            = Barracuda
character-set-client-handshake = FALSE
```

Restart the server. Verify :
```sql
SHOW VARIABLES LIKE 'innodb_large_prefix';
SHOW VARIABLES LIKE 'innodb_default_row_format';
-- Both should be ON / dynamic.
```

## AP-09 : JSON column without `CHECK (JSON_VALID(...))`

### Anti-pattern

```sql
CREATE TABLE webhook_event (
  id      BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  payload JSON NOT NULL                                      -- no CHECK constraint
);
```

### Why it fails

- MariaDB JSON is an alias for LONGTEXT (per D-010 ; see LESSONS L-005). It is plain text, not a binary structured type.
- Without the CHECK constraint, an INSERT of arbitrary text succeeds : `INSERT INTO webhook_event (payload) VALUES ('not even json');` returns OK.
- Every downstream consumer that calls `JSON_EXTRACT`, `JSON_VALUE`, or `JSON_TABLE` then crashes on the bad row with error 4038 `The JSON binary value is invalid`.
- The bad row is invisible until a JSON function is called on it, which is often weeks or months after the bad write.

### Symptoms

- Reports of "this dashboard query started failing yesterday" with `Invalid JSON text` errors pointing at rows inserted long ago.
- A migration from MySQL native-JSON column to MariaDB JSON column silently allows bad data on the MariaDB side.

### Fix

```sql
-- Every JSON column gets the validator
CREATE TABLE webhook_event (
  id      BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  payload JSON NOT NULL CHECK (JSON_VALID(payload))
);

-- Retro-fit existing column :
ALTER TABLE webhook_event
  ADD CONSTRAINT chk_payload_valid CHECK (JSON_VALID(payload));
-- Note : adding a CHECK does NOT validate existing rows by default.
-- Validate first with :
SELECT id FROM webhook_event WHERE NOT JSON_VALID(payload);
-- Clean up bad rows, then add the constraint.
```

This is the single most important schema-design rule when migrating from MySQL : MariaDB JSON does NOT validate without the explicit constraint.
