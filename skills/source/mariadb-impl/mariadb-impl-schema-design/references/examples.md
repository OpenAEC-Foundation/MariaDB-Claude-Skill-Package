# MariaDB Schema Design : Working Examples

Twelve end-to-end CREATE TABLE patterns covering primary-key choice, multi-tenancy, ROW_FORMAT, charset, foreign keys, generated columns, JSON, and Frappe / ERPNext conventions. Every snippet runs on MariaDB 10.6-LTS, 10.11-LTS, 11.x, and 12.x unless annotated otherwise.

## Example 1 : Minimal modern table (BIGINT PK)

```sql
-- 10.6+ : the default first table in any new schema
CREATE TABLE customer (
  id           BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  email        VARCHAR(255)    NOT NULL,
  display_name VARCHAR(255)    NOT NULL,
  created_at   TIMESTAMP(6)    NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  PRIMARY KEY (id),
  UNIQUE KEY uq_email (email)
) ENGINE=InnoDB
  ROW_FORMAT=DYNAMIC
  DEFAULT CHARSET=utf8mb4
  COLLATE=utf8mb4_uca1400_ai_ci;
```

`uq_email` on `VARCHAR(255)` works because DYNAMIC permits a 3072-byte index prefix (255 chars * 4 bytes = 1020 bytes < 3072). On COMPACT it would error 1071.

## Example 2 : BINARY(16) UUID primary key with swap-flag

```sql
-- 10.6+ : distributed-writer pattern, time-low/high swapped for locality
CREATE TABLE order_doc (
  id           BINARY(16)      NOT NULL,                   -- UUID_TO_BIN(uuid, 1)
  customer_id  BIGINT UNSIGNED NOT NULL,
  total_cents  BIGINT          NOT NULL,
  placed_at    TIMESTAMP(6)    NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  PRIMARY KEY (id),
  KEY ix_customer_placed (customer_id, placed_at)
) ENGINE=InnoDB ROW_FORMAT=DYNAMIC DEFAULT CHARSET=utf8mb4
  COLLATE=utf8mb4_uca1400_ai_ci;

-- Insert with server-side UUID
INSERT INTO order_doc (id, customer_id, total_cents)
VALUES (UUID_TO_BIN(UUID(), 1), 42, 19999);

-- Read back as textual UUID
SELECT BIN_TO_UUID(id, 1) AS uuid, customer_id, total_cents
FROM order_doc;
```

## Example 3 : ULID / UUIDv7 primary key (app-generated)

```sql
-- 10.6+ : application generates a time-prefixed 128-bit identifier
CREATE TABLE event (
  id          BINARY(16)      NOT NULL,                    -- ULID or UUIDv7
  aggregate   BINARY(16)      NOT NULL,
  event_type  VARCHAR(64)     NOT NULL,
  payload     JSON            NOT NULL CHECK (JSON_VALID(payload)),
  occurred_at TIMESTAMP(6)    NOT NULL,
  PRIMARY KEY (id),
  KEY ix_aggregate_time (aggregate, occurred_at),
  KEY ix_type_time      (event_type, occurred_at)
) ENGINE=InnoDB ROW_FORMAT=DYNAMIC DEFAULT CHARSET=utf8mb4
  COLLATE=utf8mb4_uca1400_ai_ci;
```

App-side example (pseudo-code) :
```python
# pip install python-ulid
from ulid import ULID
event_id = ULID().bytes        # 16 raw bytes ; first 48 bits = millisecond timestamp
cursor.execute("INSERT INTO event (id, aggregate, event_type, payload, occurred_at) VALUES (?, ?, ?, ?, NOW(6))",
               (event_id, agg_bytes, 'OrderPlaced', '{"qty":3}'))
```

## Example 4 : Row-level multi-tenant table

```sql
-- All queries filter WHERE tenant_id = ?  ; index leads with tenant_id
CREATE TABLE invoice (
  id          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  tenant_id   BIGINT UNSIGNED NOT NULL,
  number      VARCHAR(32)     NOT NULL,
  customer_id BIGINT UNSIGNED NOT NULL,
  amount      DECIMAL(12,2)   NOT NULL,
  status      ENUM('draft','sent','paid','void') NOT NULL DEFAULT 'draft',
  created_at  TIMESTAMP(6)    NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  PRIMARY KEY (id),
  UNIQUE KEY uq_tenant_number (tenant_id, number),         -- scoped per-tenant
  KEY ix_tenant_customer (tenant_id, customer_id),
  KEY ix_tenant_status   (tenant_id, status, created_at),
  CONSTRAINT fk_invoice_tenant FOREIGN KEY (tenant_id)
    REFERENCES tenant(id) ON DELETE RESTRICT
) ENGINE=InnoDB ROW_FORMAT=DYNAMIC DEFAULT CHARSET=utf8mb4
  COLLATE=utf8mb4_uca1400_ai_ci;
```

`uq_tenant_number` makes invoice numbers unique per tenant rather than globally : the common multi-tenant intent.

## Example 5 : Schema-per-tenant setup (Frappe-style)

```sql
-- One DATABASE per tenant ; SQL-level isolation
CREATE DATABASE `tenant_acme`
  DEFAULT CHARSET=utf8mb4
  DEFAULT COLLATE=utf8mb4_uca1400_ai_ci;

CREATE USER 'acme_app'@'10.%' IDENTIFIED BY '<strong-secret>';
GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, ALTER, INDEX, REFERENCES, EXECUTE
  ON `tenant_acme`.* TO 'acme_app'@'10.%';
FLUSH PRIVILEGES;

-- Then create tables inside the per-tenant DB
USE `tenant_acme`;
CREATE TABLE customer (
  id           BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  email        VARCHAR(255)    NOT NULL,
  PRIMARY KEY (id),
  UNIQUE KEY uq_email (email)
) ENGINE=InnoDB ROW_FORMAT=DYNAMIC DEFAULT CHARSET=utf8mb4
  COLLATE=utf8mb4_uca1400_ai_ci;
```

Cross-tenant analytics requires a separate aggregator (Materialize, Trino, dbt) or scripted `UNION ALL` across `<tenant>.invoice` references.

## Example 6 : Frappe tabDoctype + child table

```sql
-- Frappe v15 / ERPNext v15 require MariaDB 10.6.6+ ; v16 requires 11.8+
-- Parent doctype : table name literally starts with `tab`
CREATE TABLE `tabSales Invoice` (
  name             VARCHAR(140) NOT NULL,                  -- Frappe PK is the slug
  creation         DATETIME(6)  NULL,
  modified         DATETIME(6)  NULL,
  modified_by      VARCHAR(140) NULL,
  owner            VARCHAR(140) NULL,
  docstatus        INT          NOT NULL DEFAULT 0,
  idx              INT          NOT NULL DEFAULT 0,
  customer         VARCHAR(140) NULL,
  grand_total      DECIMAL(18,6) NOT NULL DEFAULT 0,
  posting_date     DATE         NULL,
  status           VARCHAR(30)  NULL,
  PRIMARY KEY (name),
  KEY modified     (modified),
  KEY customer     (customer),
  KEY posting_date (posting_date, status)
) ENGINE=InnoDB ROW_FORMAT=DYNAMIC DEFAULT CHARSET=utf8mb4
  COLLATE=utf8mb4_unicode_ci;

-- Child table : standard parent/parentfield/parenttype/idx columns are mandatory
CREATE TABLE `tabSales Invoice Item` (
  name           VARCHAR(140) NOT NULL,
  parent         VARCHAR(140) NULL,                        -- by convention = parent.name
  parentfield    VARCHAR(140) NULL,                        -- 'items', 'taxes', ...
  parenttype     VARCHAR(140) NULL,                        -- 'Sales Invoice'
  idx            INT          NOT NULL DEFAULT 0,
  item_code      VARCHAR(140) NULL,
  qty            DECIMAL(18,6) NOT NULL DEFAULT 0,
  rate           DECIMAL(18,6) NOT NULL DEFAULT 0,
  amount         DECIMAL(18,6) NOT NULL DEFAULT 0,
  PRIMARY KEY (name),
  KEY parent     (parent, parentfield, parenttype),
  KEY item_code  (item_code)
) ENGINE=InnoDB ROW_FORMAT=DYNAMIC DEFAULT CHARSET=utf8mb4
  COLLATE=utf8mb4_unicode_ci;
```

Note : Frappe historically uses `utf8mb4_unicode_ci`. Switching to `utf8mb4_uca1400_ai_ci` requires testing against framework search/sort code paths.

## Example 7 : JSON column with virtual-column functional index

```sql
-- MariaDB JSON is LONGTEXT (per D-010) : structure via CHECK,
-- index via virtual column over JSON_VALUE().
CREATE TABLE document (
  id         BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  tenant_id  BIGINT UNSIGNED NOT NULL,
  body       JSON            NOT NULL CHECK (JSON_VALID(body)),
  doc_type   VARCHAR(64)     AS (JSON_VALUE(body, '$.type')) VIRTUAL,
  author_id  BIGINT UNSIGNED AS (JSON_VALUE(body, '$.author.id')) VIRTUAL,
  PRIMARY KEY (id),
  KEY ix_tenant_type   (tenant_id, doc_type),
  KEY ix_tenant_author (tenant_id, author_id)
) ENGINE=InnoDB ROW_FORMAT=DYNAMIC DEFAULT CHARSET=utf8mb4
  COLLATE=utf8mb4_uca1400_ai_ci;
```

Adding an index on a VIRTUAL column requires `ALGORITHM=COPY` per the generated-columns KB ; budget for a table rewrite when adding new functional indexes later.

## Example 8 : Foreign-key cascade design

```sql
-- Small bounded child : CASCADE acceptable. Huge child : RESTRICT only.
CREATE TABLE invoice (
  id          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  customer_id BIGINT UNSIGNED NOT NULL,
  total_cents BIGINT          NOT NULL,
  PRIMARY KEY (id),
  KEY ix_customer (customer_id),
  CONSTRAINT fk_invoice_customer FOREIGN KEY (customer_id)
    REFERENCES customer(id) ON DELETE RESTRICT
) ENGINE=InnoDB ROW_FORMAT=DYNAMIC DEFAULT CHARSET=utf8mb4
  COLLATE=utf8mb4_uca1400_ai_ci;

CREATE TABLE invoice_line (
  id          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  invoice_id  BIGINT UNSIGNED NOT NULL,
  description VARCHAR(255)    NOT NULL,
  amount_cents BIGINT         NOT NULL,
  PRIMARY KEY (id),
  KEY ix_invoice (invoice_id),
  CONSTRAINT fk_invoice_line_invoice FOREIGN KEY (invoice_id)
    REFERENCES invoice(id) ON DELETE CASCADE                -- small child : OK
) ENGINE=InnoDB ROW_FORMAT=DYNAMIC DEFAULT CHARSET=utf8mb4
  COLLATE=utf8mb4_uca1400_ai_ci;
```

If `invoice_line` had millions of rows per invoice, switch to RESTRICT and clean up children explicitly. A CASCADE on a large child holds X-locks for the duration of the delete and blocks unrelated writes.

## Example 9 : COMPRESSED archive table

```sql
-- 10.6+ : write-once log table, prioritise disk over CPU
CREATE TABLE audit_log_2024 (
  id          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  occurred_at TIMESTAMP(6)    NOT NULL,
  actor_id    BIGINT UNSIGNED NOT NULL,
  action      VARCHAR(64)     NOT NULL,
  payload     JSON            NOT NULL CHECK (JSON_VALID(payload)),
  PRIMARY KEY (id),
  KEY ix_time_actor (occurred_at, actor_id),
  KEY ix_action     (action, occurred_at)
) ENGINE=InnoDB
  ROW_FORMAT=COMPRESSED KEY_BLOCK_SIZE=8
  DEFAULT CHARSET=utf8mb4
  COLLATE=utf8mb4_uca1400_ai_ci;
```

Limitations to remember : INSTANT ALTER is rejected, CPU per read rises, page-splits get more expensive. Acceptable for archive workloads, never for high-write OLTP.

## Example 10 : Generated PERSISTENT column for FK

```sql
-- 10.6+ : VIRTUAL columns cannot back an FK ; PERSISTENT can
CREATE TABLE shipment (
  id         BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  payload    JSON            NOT NULL CHECK (JSON_VALID(payload)),
  carrier_id BIGINT UNSIGNED
    AS (CAST(JSON_VALUE(payload, '$.carrier_id') AS UNSIGNED)) PERSISTENT,
  PRIMARY KEY (id),
  KEY ix_carrier (carrier_id),
  CONSTRAINT fk_shipment_carrier FOREIGN KEY (carrier_id)
    REFERENCES carrier(id) ON DELETE RESTRICT
) ENGINE=InnoDB ROW_FORMAT=DYNAMIC DEFAULT CHARSET=utf8mb4
  COLLATE=utf8mb4_uca1400_ai_ci;
```

PERSISTENT materialises the carrier_id on disk so the FK can reference it. KB-documented restriction : CASCADE / SET NULL actions are forbidden when the FK column is a PERSISTENT generated column.

## Example 11 : Mixed example combining everything

```sql
-- Production-grade : tenant scoping + JSON + functional index + audit timestamps + FK
CREATE TABLE ticket (
  id           BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  tenant_id    BIGINT UNSIGNED NOT NULL,
  external_id  BINARY(16)      NOT NULL,                   -- ULID for external API
  subject      VARCHAR(255)    NOT NULL,
  priority     ENUM('low','medium','high','urgent') NOT NULL DEFAULT 'medium',
  assignee_id  BIGINT UNSIGNED NULL,
  details      JSON            NOT NULL CHECK (JSON_VALID(details)),
  source_chan  VARCHAR(32)     AS (JSON_VALUE(details, '$.channel')) VIRTUAL,
  created_at   TIMESTAMP(6)    NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  updated_at   TIMESTAMP(6)    NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
                                       ON UPDATE CURRENT_TIMESTAMP(6),
  PRIMARY KEY (id),
  UNIQUE KEY uq_external_id     (external_id),
  KEY ix_tenant_priority_open  (tenant_id, priority, created_at),
  KEY ix_tenant_assignee       (tenant_id, assignee_id),
  KEY ix_tenant_channel        (tenant_id, source_chan),
  CONSTRAINT fk_ticket_tenant   FOREIGN KEY (tenant_id)   REFERENCES tenant(id)   ON DELETE RESTRICT,
  CONSTRAINT fk_ticket_assignee FOREIGN KEY (assignee_id) REFERENCES user_acct(id) ON DELETE SET NULL
) ENGINE=InnoDB ROW_FORMAT=DYNAMIC DEFAULT CHARSET=utf8mb4
  COLLATE=utf8mb4_uca1400_ai_ci;
```

`assignee_id` is NULLable so `ON DELETE SET NULL` is valid. The functional index on `source_chan` accelerates filtering tickets by their JSON-payload-derived channel without scanning every row.

## Example 12 : Charset migration of a legacy table

```sql
-- Legacy table created on stock 10.6 with latin1 default
CREATE TABLE legacy_note (
  id   INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  body TEXT
) ENGINE=InnoDB;
-- (charset inherited from server : latin1_swedish_ci)

-- 10.6+ migration to utf8mb4 :
ALTER TABLE legacy_note
  CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_uca1400_ai_ci,
  ALGORITHM=COPY, LOCK=SHARED;
```

`CONVERT TO CHARACTER SET` is a rewrite (ALGORITHM=COPY only). Plan a maintenance window for large tables, or use `pt-online-schema-change` for zero-downtime conversion. After conversion, double-check that no index now exceeds the prefix limit on the target row format.
