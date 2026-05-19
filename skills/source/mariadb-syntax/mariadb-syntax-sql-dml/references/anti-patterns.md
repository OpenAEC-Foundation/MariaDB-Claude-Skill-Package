# MariaDB DML : Anti-Patterns

10 real-world DML anti-patterns, each with the broken code, the failure mode, and the correct replacement. Every pattern is observed in production codebases or documented in the MariaDB KB.

---

## AP-1 : REPLACE INTO on a parent table with ON DELETE CASCADE

### Bad

```sql
-- Schema :
CREATE TABLE customer (id BIGINT PRIMARY KEY, name VARCHAR(120)) ENGINE=InnoDB;
CREATE TABLE invoice (
  id BIGINT PRIMARY KEY,
  customer_id BIGINT,
  CONSTRAINT fk_invoice_customer
    FOREIGN KEY (customer_id) REFERENCES customer(id) ON DELETE CASCADE
) ENGINE=InnoDB;

-- The "upsert" :
REPLACE INTO customer (id, name) VALUES (1, 'Acme v2');
```

### Why it fails

`REPLACE` is implemented as DELETE-then-INSERT (KB-verbatim conceptual model). The DELETE phase triggers `ON DELETE CASCADE` on `invoice`, wiping every invoice for customer 1. The INSERT phase then writes a fresh customer row with id 1 and no children. The application sees "the customer is updated" and the invoice history silently disappears.

### Correct

```sql
INSERT INTO customer (id, name) VALUES (1, 'Acme v2')
  ON DUPLICATE KEY UPDATE name = VALUES(name);
```

ON DUPLICATE KEY UPDATE performs an in-place UPDATE on conflict ; the DELETE never fires ; the CASCADE never fires ; the invoices survive.

---

## AP-2 : INSERT IGNORE used as "I do not care if it exists"

### Bad

```sql
INSERT IGNORE INTO customer (id, name, age)
  VALUES (1, 'Acme', 999);   -- age is TINYINT UNSIGNED (max 255)
```

### Why it fails

`INSERT IGNORE` silently converts ALL errors to warnings (KB-verbatim) : duplicate-key conflicts, NOT NULL violations with no default, out-of-range type conversions, and FK violations. The 999 is silently coerced to 255 (the closest valid TINYINT UNSIGNED value), the row is inserted, the return code is success, and the warning is buried unless the application explicitly calls SHOW WARNINGS.

### Correct

```sql
-- Option A : explicit no-op upsert on the unique key
INSERT INTO customer (id, name, age)
  VALUES (1, 'Acme', 999)
  ON DUPLICATE KEY UPDATE id = id;
-- The age=999 still triggers the out-of-range error visibly ; only the duplicate-key is silenced.

-- Option B : pre-check
SELECT 1 FROM customer WHERE id = 1;
-- If empty, then : INSERT INTO customer ...
```

Audit every existing `INSERT IGNORE` against these categories before declaring the codebase safe.

---

## AP-3 : ON DUPLICATE KEY UPDATE with multiple UNIQUE indexes

### Bad

```sql
CREATE TABLE contact (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  email VARCHAR(255) UNIQUE,
  phone VARCHAR(40) UNIQUE,
  name VARCHAR(120)
) ENGINE=InnoDB;

-- Existing rows :
INSERT INTO contact (email, phone, name) VALUES ('a@x.com', '555-1', 'Alice');
INSERT INTO contact (email, phone, name) VALUES ('b@x.com', '555-2', 'Bob');

-- The upsert :
INSERT INTO contact (email, phone, name) VALUES ('a@x.com', '555-2', 'Alice-2')
  ON DUPLICATE KEY UPDATE name = VALUES(name);
```

### Why it fails

KB-verbatim : "If more than one unique index is matched, only the first is updated." Both Alice's email AND Bob's phone match the new row. Only one of them (the first-matched, in undefined order) is updated. The other stays as-is. The application sees "1 row affected" or "2 rows affected" and trusts it.

### Correct

Restructure the schema so the intent is captured by ONE composite UNIQUE key, or write explicit branching :

```sql
-- Schema : single composite UNIQUE
ALTER TABLE contact DROP INDEX phone, DROP INDEX email,
  ADD UNIQUE KEY ux_contact (email, phone);

-- OR : explicit transactional branching
BEGIN;
  SELECT id FROM contact WHERE email = 'a@x.com' FOR UPDATE;
  -- branch in app code : UPDATE or INSERT
COMMIT;
```

---

## AP-4 : UPDATE without WHERE

### Bad

```sql
UPDATE customer SET tier = 'gold';
```

### Why it fails

This updates every row. In production the typo is usually `UPDATE customer SET tier = 'gold' WHERE id = 1` with the WHERE missed by an editor accident. The damage is total ; rollback requires a backup.

### Correct

```sql
-- ALWAYS include a WHERE on UPDATE.
UPDATE customer SET tier = 'gold' WHERE id = 1;

-- In interactive sessions, enable --safe-updates which refuses UPDATE / DELETE
-- without a WHERE that uses an indexed column.
SET sql_safe_updates = 1;
UPDATE customer SET tier = 'gold';   -- ERROR 1175 : You are using safe update mode...
```

`sql_safe_updates` blocks the unguarded UPDATE/DELETE and is a $0 safety net for human-driven sessions. Set it in client config (`.my.cnf` `[client]` block) so every interactive session inherits it.

---

## AP-5 : Multi-table UPDATE expecting deterministic row-order

### Bad

```sql
UPDATE customer c
JOIN tier_rule r ON r.min_score <= c.score
SET c.tier = r.tier_name
WHERE c.score > 0
ORDER BY r.min_score DESC
LIMIT 1;   -- ERROR 1221 in pre-10.3 : ORDER BY + LIMIT not allowed in multi-table UPDATE
```

### Why it fails

Two distinct failures depending on version :
- Pre-10.3 : `ORDER BY` + `LIMIT` in multi-table UPDATE is a syntax error.
- 10.3+ : it parses, but row update order across the JOIN is undefined. Two tier_rule rows can match the same customer ; the "last" winning rule is non-deterministic.

### Correct

```sql
-- Compute the winning rule per customer in a derived table, then UPDATE deterministically.
UPDATE customer c
JOIN (
  SELECT c2.id AS customer_id,
         (SELECT r.tier_name
          FROM tier_rule r
          WHERE r.min_score <= c2.score
          ORDER BY r.min_score DESC
          LIMIT 1) AS winning_tier
  FROM customer c2
  WHERE c2.score > 0
) AS w ON w.customer_id = c.id
SET c.tier = w.winning_tier;
```

The derived table makes the "winning rule per customer" deterministic ; the outer UPDATE then writes one value per customer with no row-order ambiguity.

---

## AP-6 : ON DUPLICATE KEY UPDATE and LAST_INSERT_ID assumption

### Bad

```sql
INSERT INTO inventory (sku, qty, last_seen)
  VALUES ('SKU-001', 5, NOW())
  ON DUPLICATE KEY UPDATE
    qty = qty + VALUES(qty);

SELECT LAST_INSERT_ID();   -- expected to be the row's id ; actually 0 on UPDATE branch
```

### Why it fails

When ODKU takes the UPDATE branch, no new row is inserted, so `LAST_INSERT_ID()` is NOT updated. The application reads 0 (or whatever the previous LAST_INSERT_ID was) and treats it as a new id.

### Correct

```sql
INSERT INTO inventory (sku, qty, last_seen)
  VALUES ('SKU-001', 5, NOW())
  ON DUPLICATE KEY UPDATE
    id = LAST_INSERT_ID(id),     -- THE IDIOM : write the existing id into LAST_INSERT_ID
    qty = qty + VALUES(qty);

SELECT LAST_INSERT_ID();   -- now the existing row's id
```

`LAST_INSERT_ID(expr)` with an argument sets the session's last-insert-id to that expression. Adding `id = LAST_INSERT_ID(id)` to the SET clause is the canonical way to retrieve the row id on the UPDATE branch.

---

## AP-7 : ON DUPLICATE KEY UPDATE burning AUTO_INCREMENT (assumed safe)

### Bad

```sql
-- High-frequency upsert in a tight loop
INSERT INTO event_dedup (event_key, last_seen)
  VALUES (?, NOW())
  ON DUPLICATE KEY UPDATE last_seen = NOW();
-- event_dedup.id is AUTO_INCREMENT
```

After 10 million conflicts (no inserts), the next genuine new row gets id = 10_000_001. Gaps of millions are common.

### Why it fails

The AUTO_INCREMENT counter advances on every INSERT attempt regardless of whether the ODKU UPDATE branch was taken (KB-verbatim : "If there is an AUTO_INCREMENT field, a new value will be generated.") This is by-design but surprises developers who expect the counter to stay flat when only the UPDATE branch is taken.

### Correct

```sql
-- Option A : drop the AUTO_INCREMENT and use the natural key as primary.
ALTER TABLE event_dedup
  DROP PRIMARY KEY,
  DROP COLUMN id,
  ADD PRIMARY KEY (event_key);

-- Option B : SELECT-then-INSERT (more round-trips, no burn)
SELECT 1 FROM event_dedup WHERE event_key = ? FOR UPDATE;
-- If found : UPDATE last_seen ; else : INSERT.
```

Accept the gaps OR remove the AUTO_INCREMENT. Trying to "fix" the gaps with `ALTER TABLE ... AUTO_INCREMENT = N` resets only the next value, not historical gaps.

---

## AP-8 : UPDATE RETURNING assumed to exist in LTS

### Bad

```sql
-- Assumed available because PostgreSQL and MySQL 8.0 have it :
UPDATE customer
  SET tier = 'gold'
  WHERE id = 1
  RETURNING id, tier;
-- ERROR : 1064 You have an error in your SQL syntax (on 10.6 / 10.11 / 11.x / 12.x)
```

### Why it fails

`UPDATE RETURNING` does NOT exist in MariaDB 10.6-LTS, 10.11-LTS, 11.x, or 12.x. KB-verified : the feature is available only from MariaDB 13.0 onwards. INSERT RETURNING (since 10.5) and DELETE RETURNING (since 10.0) exist, but UPDATE RETURNING does not.

### Correct

```sql
-- Pattern for LTS : SELECT before + UPDATE in a transaction
BEGIN;
  SELECT id, tier FROM customer WHERE id = 1 FOR UPDATE;   -- old values
  UPDATE customer SET tier = 'gold' WHERE id = 1;
  SELECT id, tier FROM customer WHERE id = 1;              -- new values
COMMIT;
```

---

## AP-9 : DELETE FROM t WHERE id IN (SELECT id FROM t ...) without derived table

### Bad

```sql
DELETE FROM customer
WHERE id IN (
  SELECT id FROM customer WHERE created_at < NOW() - INTERVAL 10 YEAR LIMIT 100
);
-- On pre-10.3.1 : ERROR 1093 (HY000) : You can't specify target table 'customer' for update in FROM clause
```

### Why it fails

Historically MariaDB (and MySQL) refused to read from a table inside a subquery while modifying the same table. Since MariaDB 10.3.1 this was relaxed, but the subquery still cannot use `LIMIT` directly in some plan shapes ; portable code wraps the subquery in a derived table.

### Correct

```sql
DELETE FROM customer
WHERE id IN (
  SELECT id FROM (
    SELECT id FROM customer
    WHERE created_at < NOW() - INTERVAL 10 YEAR
    LIMIT 100
  ) AS to_purge
);
```

The `AS to_purge` derived table forces materialization, which (a) is portable across MariaDB versions, and (b) makes the LIMIT predictable.

---

## AP-10 : DELETE on a system-versioned table to "purge old data"

### Bad

```sql
CREATE TABLE audit_log (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  event_at DATETIME NOT NULL
) WITH SYSTEM VERSIONING;

-- "Purge" old rows :
DELETE FROM audit_log WHERE event_at < NOW() - INTERVAL 7 YEAR;
```

### Why it fails

On a system-versioned table, a regular `DELETE` does NOT remove the row from disk : it moves the row to history with `row_end = NOW()`. Disk usage keeps growing. The intent ("purge") is not satisfied. Compounding this : if the table has many such "soft-deleted" rows, `SELECT * FROM audit_log FOR SYSTEM_TIME ALL` still returns them, which surprises auditors who expected actual deletion.

### Correct

```sql
-- Step 1 : if intent is "purge old history only" :
DELETE HISTORY FROM audit_log
  BEFORE SYSTEM_TIME NOW() - INTERVAL 7 YEAR;
-- This actually drops history rows older than the cutoff. Requires DELETE HISTORY privilege.

-- Step 2 : if intent is "delete live rows but keep history shorter than 7 years" :
DELETE FROM audit_log WHERE event_at < NOW() - INTERVAL 7 YEAR;  -- soft-delete
DELETE HISTORY FROM audit_log BEFORE SYSTEM_TIME NOW() - INTERVAL 7 YEAR;  -- hard-purge

-- Caveat : MDEV-25468 documents that BEFORE SYSTEM_TIME later than active rows'
-- row_end can ALSO drop active rows. Pick a cutoff comfortably in the past.
```

The `DELETE HISTORY` privilege is granted separately (`GRANT DELETE HISTORY ON db.tbl TO 'user'@'host'`) ; regular `DELETE` privilege is not sufficient.

---

## Quick checklist before merging DML changes

1. Every UPDATE / DELETE has a WHERE clause OR is explicitly intended to touch every row (document why).
2. No `INSERT IGNORE` unless the use case is bulk dirty-data import AND `SHOW WARNINGS` is logged after the batch.
3. No `REPLACE INTO` on tables with FK CASCADE children, DELETE triggers, or AUTO_INCREMENT stability requirements.
4. ON DUPLICATE KEY UPDATE used only when the table has exactly one UNIQUE / PRIMARY key that could match.
5. No `UPDATE ... RETURNING` for LTS-targeted code (only valid from 13.0+).
6. Multi-table UPDATE / DELETE does not rely on row-order ; derived tables enforce determinism where needed.
7. System-versioned tables : DELETE for soft-delete, DELETE HISTORY for hard-purge ; never confused.
