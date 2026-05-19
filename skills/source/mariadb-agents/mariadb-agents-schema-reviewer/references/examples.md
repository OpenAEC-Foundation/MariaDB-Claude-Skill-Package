# examples : worked schema reviews

Each example shows a supplied schema, the review applied across the 10 dimensions, the findings table, and the verdict. These are the canonical demonstrations of how to run this skill.

---

## Example 1 : a MyISAM transactional schema (full report)

Supplied DDL :

```sql
-- 10.6+
CREATE TABLE orders (
  id INT NOT NULL,
  customer_id INT,
  total FLOAT,
  status VARCHAR(255),
  created_at DATETIME
) ENGINE=MyISAM CHARSET=utf8;

CREATE TABLE order_lines (
  order_id INT,
  sku VARCHAR(255),
  qty INT,
  price FLOAT
) ENGINE=MyISAM CHARSET=utf8;
```

Review :

```
## Schema Review : supplied DDL (orders, order_lines)

| Dimension | Severity | Location | Problem | Fix | Skill ref |
|-----------|----------|----------|---------|-----|-----------|
| Storage engine | BLOCKER | orders | ENGINE=MyISAM on a transactional table : no transactions, table-level locking, no crash recovery | ALTER TABLE orders ENGINE=InnoDB | mariadb-core-storage-engines |
| Storage engine | BLOCKER | order_lines | ENGINE=MyISAM on a transactional table | ALTER TABLE order_lines ENGINE=InnoDB | mariadb-core-storage-engines |
| Data types | BLOCKER | orders.total | FLOAT for a money column drifts on sums and comparisons | DECIMAL(13, 2) | mariadb-syntax-sql-ddl |
| Data types | BLOCKER | order_lines.price | FLOAT for a money column | DECIMAL(13, 2) | mariadb-syntax-sql-ddl |
| Primary key | BLOCKER | order_lines | No PRIMARY KEY ; InnoDB synthesises an unindexable hidden key | Add a surrogate id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY | mariadb-impl-schema-design |
| Primary key | WARNING | orders.id | id INT NOT NULL is not declared PRIMARY KEY | Add PRIMARY KEY (id), prefer BIGINT UNSIGNED AUTO_INCREMENT | mariadb-impl-schema-design |
| Charset | WARNING | orders, order_lines | CHARSET=utf8 (utf8mb3) cannot store emoji or 4-byte characters | DEFAULT CHARSET=utf8mb4 | mariadb-errors-encoding-and-collation |
| Constraints | WARNING | order_lines.order_id | References orders with no FOREIGN KEY | FOREIGN KEY (order_id) REFERENCES orders(id) | mariadb-syntax-check-constraints |
| Indexing | WARNING | orders.customer_id | Foreign-key-shaped column with no index | INDEX (customer_id) | mariadb-syntax-indexing |
| Data types | SUGGESTION | orders.status | VARCHAR(255) for a finite status set | Size to real max or add CHECK (status IN (...)) | mariadb-syntax-sql-ddl |

**Verdict** : FAIL (5 blockers, 4 warnings, 1 suggestion). Re-review after blockers are fixed.
```

---

## Example 2 : a multi-tenant schema missing the tenant index

Supplied DDL :

```sql
-- 10.11+
CREATE TABLE invoices (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  tenant_id BIGINT UNSIGNED NOT NULL,
  number VARCHAR(40) NOT NULL,
  amount DECIMAL(13,2) NOT NULL,
  issued_on DATE,
  INDEX ix_number (number),
  INDEX ix_issued (issued_on)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

Review :

```
## Schema Review : supplied DDL (invoices)

| Dimension | Severity | Location | Problem | Fix | Skill ref |
|-----------|----------|----------|---------|-----|-----------|
| Multi-tenant | WARNING | invoices | tenant_id detected, but no index leads with tenant_id ; every tenant-scoped query scans the table | INDEX ix_tenant_issued (tenant_id, issued_on) ; reorder ix_number to (tenant_id, number) | mariadb-impl-schema-design |
| Multi-tenant | SUGGESTION | invoices | PRIMARY KEY does not lead with tenant_id ; tenant rows are not co-located on disk | Consider PRIMARY KEY (tenant_id, id) | mariadb-impl-schema-design |
| Constraints | SUGGESTION | invoices.amount | Money column with no non-negative guard | CHECK (amount >= 0) | mariadb-syntax-check-constraints |

**Verdict** : PASS WITH WARNINGS (0 blockers, 1 warning, 2 suggestions). The schema works but tenant queries will not use an index until the warning is fixed.
```

---

## Example 3 : a Frappe-style schema (tab<Doctype> pattern)

Supplied DDL :

```sql
-- 10.6+
CREATE TABLE `tabSales Invoice` (
  name VARCHAR(140) NOT NULL,
  creation DATETIME(6),
  modified DATETIME(6),
  customer VARCHAR(140),
  grand_total DECIMAL(21,9),
  status VARCHAR(140),
  PRIMARY KEY (name)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

Review :

```
## Schema Review : supplied DDL (tabSales Invoice)

| Dimension | Severity | Location | Problem | Fix | Skill ref |
|-----------|----------|----------|---------|-----|-----------|
| Naming | NOTE | tabSales Invoice | tab<Doctype> pattern : Frappe / ERPNext-generated schema. Do NOT rename ; the framework owns these identifiers | none | mariadb-impl-schema-design |
| Indexing | WARNING | tabSales Invoice.customer | customer is a Frappe Link field referencing tabCustomer, no index present | INDEX (customer) | mariadb-syntax-indexing |
| Constraints | SUGGESTION | tabSales Invoice.status | Finite status set, no CHECK ; note Frappe enforces this in the app layer | Optional CHECK ; leave to framework if app-enforced | mariadb-syntax-check-constraints |

**Verdict** : PASS WITH WARNINGS (0 blockers, 1 warning, 1 suggestion). Naming is out of scope for framework-generated schemas.
```

Reviewer note : a Frappe schema must not be graded on naming. The `tab` prefix, spaces in table names, and the `name` VARCHAR PK are all framework conventions. Grade only engine, indexing, charset, and data types.

---

## Example 4 : a JSON-heavy schema

Supplied DDL :

```sql
-- 10.6+
CREATE TABLE events (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  occurred_at DATETIME NOT NULL,
  payload JSON
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

Review :

```
## Schema Review : supplied DDL (events)

| Dimension | Severity | Location | Problem | Fix | Skill ref |
|-----------|----------|----------|---------|-----|-----------|
| JSON | WARNING | events.payload | JSON column with no CHECK (JSON_VALID(payload)) ; MariaDB JSON is a LONGTEXT alias and accepts invalid text silently | ALTER TABLE events ADD CONSTRAINT chk_payload CHECK (JSON_VALID(payload)) | mariadb-syntax-json |
| JSON | SUGGESTION | events.payload | If queried by path, no virtual-column functional index exists | Add a VIRTUAL column over JSON_VALUE(payload, '$.type') and index it | mariadb-syntax-json |

**Verdict** : PASS WITH WARNINGS (0 blockers, 1 warning, 1 suggestion). Add the JSON_VALID CHECK before shipping.
```

---

## Example 5 : a clean schema (all-pass report)

Supplied DDL :

```sql
-- 10.11+
CREATE TABLE customers (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  email VARCHAR(320) NOT NULL,
  display_name VARCHAR(120) NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT current_timestamp(),
  UNIQUE KEY uq_email (email)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_uca1400_ai_ci;

CREATE TABLE orders (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  customer_id BIGINT UNSIGNED NOT NULL,
  total DECIMAL(13,2) NOT NULL,
  status VARCHAR(20) NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT current_timestamp(),
  INDEX ix_customer (customer_id),
  CONSTRAINT fk_orders_customer FOREIGN KEY (customer_id) REFERENCES customers(id),
  CONSTRAINT chk_total CHECK (total >= 0),
  CONSTRAINT chk_status CHECK (status IN ('new','paid','shipped','cancelled'))
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_uca1400_ai_ci;
```

Review :

```
## Schema Review : supplied DDL (customers, orders)

| Dimension | Severity | Location | Problem | Fix | Skill ref |
|-----------|----------|----------|---------|-----|-----------|
| (no findings) | | | | | |

**Verdict** : PASS (0 blockers, 0 warnings, 0 suggestions). All 10 dimensions reviewed : InnoDB engine, BIGINT UNSIGNED PKs, indexed FK, utf8mb4 with explicit collation, normalized, no tenant column, consistent snake_case, CHECK and FOREIGN KEY constraints present, DECIMAL for money, no JSON columns.
```

A clean schema still produces a report. State that all 10 dimensions were run and name what passed.

---

## Example 6 : a migration DDL review

Supplied migration script :

```sql
-- migration 0042 : add loyalty tier
ALTER TABLE customers ADD COLUMN loyalty_tier VARCHAR(255);
ALTER TABLE customers ADD COLUMN points FLOAT DEFAULT 0;
ALTER TABLE customers ADD COLUMN tenant_id BIGINT UNSIGNED;
```

Review :

```
## Schema Review : migration 0042 (ALTER TABLE customers)

| Dimension | Severity | Location | Problem | Fix | Skill ref |
|-----------|----------|----------|---------|-----|-----------|
| Data types | BLOCKER | customers.points | FLOAT for a points / balance column drifts on accumulation | DECIMAL(12, 2) or BIGINT if points are integers | mariadb-syntax-sql-ddl |
| Multi-tenant | WARNING | customers.tenant_id | New tenant_id column added with no index ; existing queries become tenant-blind full scans | After the ALTER, add INDEX leading with tenant_id, and decide if existing UNIQUE keys must become (tenant_id, ...) | mariadb-impl-schema-design |
| Constraints | WARNING | customers.tenant_id | tenant_id added as NULLable with no default ; existing rows get NULL and escape tenant scoping | ADD COLUMN tenant_id BIGINT UNSIGNED NOT NULL after a backfill, or backfill then ALTER to NOT NULL | mariadb-syntax-check-constraints |
| Data types | SUGGESTION | customers.loyalty_tier | VARCHAR(255) for a finite tier set | Size down and add CHECK (loyalty_tier IN ('bronze','silver','gold')) | mariadb-syntax-sql-ddl |

**Verdict** : FAIL (1 blocker, 2 warnings, 1 suggestion). A migration that adds a tenant column without an index or NOT NULL constraint silently breaks tenant isolation.
```

For migration DDL, review the post-migration state of the table, not just the added columns. A new tenant column triggers the full multi-tenant dimension.

---

## Example 7 : an un-normalized schema with repeating groups

Supplied DDL :

```sql
-- 10.6+
CREATE TABLE contacts (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  full_name VARCHAR(120),
  phone1 VARCHAR(30),
  phone2 VARCHAR(30),
  phone3 VARCHAR(30),
  tags VARCHAR(500)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

Review :

```
## Schema Review : supplied DDL (contacts)

| Dimension | Severity | Location | Problem | Fix | Skill ref |
|-----------|----------|----------|---------|-----|-----------|
| Normalization | WARNING | contacts.phone1/2/3 | Repeating group of columns ; cannot store a fourth phone, cannot query "any contact with phone X" with an index | Extract a contact_phones child table with FK to contacts(id) | mariadb-impl-schema-design |
| Normalization | WARNING | contacts.tags | Comma-separated list in one column ; not atomic, not indexable, breaks joins | Junction table contact_tags(contact_id, tag_id) | mariadb-impl-schema-design |

**Verdict** : PASS WITH WARNINGS (0 blockers, 2 warnings). The schema runs but the repeating group and the delimited list will block indexed lookups.
```

---

## Example 8 : a UUID-text primary key schema

Supplied DDL :

```sql
-- 10.6+
CREATE TABLE sessions (
  id CHAR(36) NOT NULL PRIMARY KEY,
  user_id CHAR(36) NOT NULL,
  data JSON,
  expires_at DATETIME NOT NULL,
  INDEX ix_user (user_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

Review :

```
## Schema Review : supplied DDL (sessions)

| Dimension | Severity | Location | Problem | Fix | Skill ref |
|-----------|----------|----------|---------|-----|-----------|
| Primary key | WARNING | sessions.id | CHAR(36) UUID-text PK : up to 144 bytes in utf8mb4, copied into every secondary index, random insert order causes page splits | BINARY(16) with UUID_TO_BIN(uuid, 1) ; consider UUIDv7 for time-ordered keys | mariadb-impl-schema-design |
| Primary key | WARNING | sessions.user_id | CHAR(36) UUID-text foreign key column ; same bloat in ix_user | BINARY(16) to match the customers PK type | mariadb-impl-schema-design |
| JSON | WARNING | sessions.data | JSON column with no CHECK (JSON_VALID(data)) | ADD CONSTRAINT chk_data CHECK (JSON_VALID(data)) | mariadb-syntax-json |

**Verdict** : PASS WITH WARNINGS (0 blockers, 3 warnings). The schema is functional but the UUID-text keys triple index size and the JSON column accepts invalid text.
```
