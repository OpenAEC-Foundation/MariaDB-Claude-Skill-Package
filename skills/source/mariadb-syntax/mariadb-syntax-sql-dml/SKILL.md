---
name: mariadb-syntax-sql-dml
description: >
  Use when writing INSERT, UPDATE, DELETE, REPLACE, or upsert statements, debugging "why was this row not updated", or migrating MySQL DML patterns to MariaDB.
  Prevents the INSERT IGNORE silent-corruption trap, REPLACE INTO FK cascade, ON DUPLICATE KEY auto-increment burn, multi-table UPDATE/DELETE ordering pitfalls, and the "UPDATE RETURNING does not exist in LTS" gotcha.
  Covers INSERT single-row + multi-row + INSERT SET + INSERT ... SELECT, INSERT ... ON DUPLICATE KEY UPDATE, INSERT IGNORE pitfalls, REPLACE INTO, UPDATE ... ORDER BY ... LIMIT, multi-table UPDATE/DELETE with JOIN, INSERT/DELETE RETURNING, DELETE HISTORY for system-versioned tables.
  Keywords: INSERT, UPDATE, DELETE, REPLACE, ON DUPLICATE KEY UPDATE, INSERT IGNORE, RETURNING, INSERT RETURNING, DELETE RETURNING, multi-table UPDATE, multi-table DELETE, upsert, why was my row not updated, my insert ignored the error, auto increment burn, FK cascade on REPLACE, mariadb upsert pattern, UPDATE RETURNING not supported, DELETE HISTORY, system-versioned table purge, LAST_INSERT_ID, ROW_COUNT
license: MIT
compatibility: "Designed for Claude Code. Requires MariaDB 10.6-LTS, 10.11-LTS, 11.x, 12.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# MariaDB DML : INSERT, UPDATE, DELETE, REPLACE

Deterministic SQL DML for MariaDB. Use these patterns to write correct INSERT, UPDATE, DELETE, and REPLACE statements, and to avoid the well-known footguns that look like they work and silently corrupt data.

## Quick Reference

- **ALWAYS prefer `INSERT ... ON DUPLICATE KEY UPDATE` over `REPLACE INTO`**. `REPLACE` is a DELETE-then-INSERT under the hood, which fires `ON DELETE CASCADE`, fires DELETE + INSERT triggers, and burns a fresh `AUTO_INCREMENT` value on every call (verified : KB `replace/`).
- **NEVER use `INSERT IGNORE` to mean "skip if duplicate"**. It silently coerces NOT NULL violations to empty string / zero, suppresses out-of-range conversion errors, and hides every other constraint violation as a warning. Use `INSERT ... ON DUPLICATE KEY UPDATE` with a no-op SET, or a pre-check SELECT.
- **`ON DUPLICATE KEY UPDATE` burns AUTO_INCREMENT when the UPDATE branch is taken** on tables with non-zero `auto_increment_increment`. Each conflicting INSERT consumes a sequence value even though no row is inserted ; expect gaps. `LAST_INSERT_ID()` returns the new auto-increment value only when the INSERT branch was taken ; on the UPDATE branch it returns the existing row's id only if you write `id = LAST_INSERT_ID(id)` in the SET list (idiom from KB).
- **`ON DUPLICATE KEY UPDATE` with multiple UNIQUE indexes is UNDEFINED**. KB verbatim : "If more than one unique index is matched, only the first is updated. It is not recommended to use this statement on tables with more than one unique index."
- **`UPDATE RETURNING` does NOT exist in 10.6-LTS, 10.11-LTS, 11.x, or 12.x**. RETURNING is supported on `INSERT` (since 10.5), `DELETE` (since 10.0), and `REPLACE` (since 10.5.0). `UPDATE RETURNING` is only available from MariaDB 13.0. Verified : KB `update/` documents RETURNING as 13.0-only. Do NOT assume MySQL or PostgreSQL parity.
- **Multi-table UPDATE / DELETE row order is undefined**. Do not write logic that depends on which row is updated first. If order matters, use a single-table statement with `ORDER BY ... LIMIT` and repeat in a loop, or wrap a deterministic subquery in a transaction.
- **`DELETE HISTORY` purges row-history on system-versioned tables ; regular `DELETE` creates history**. They are NOT interchangeable.
- **`RETURNING` returns affected-row data in a single round-trip**, replacing the older `INSERT ; SELECT LAST_INSERT_ID()` pattern. Use it whenever the inserted / deleted row data is needed downstream.
- **`UPDATE` without `WHERE` updates every row**. Use `--safe-updates` mode in interactive sessions, or always include a primary-key predicate.

## Decision Trees

### Tree 1 : "I want an upsert"

```
Need to insert-or-update?
  Single UNIQUE key (incl. PRIMARY)?
    Yes : INSERT ... ON DUPLICATE KEY UPDATE
      - Auto_increment safe ; FK-safe ; trigger-safe.
      - Use VALUES(col) inside SET for multi-row inserts.
    Multiple UNIQUE keys?
      Yes : DO NOT use ON DUPLICATE KEY UPDATE (undefined behavior).
        Use INSERT ... ON DUPLICATE KEY UPDATE only if the unique-keys
        cannot match different existing rows in the same call. Otherwise :
          SELECT existing -> branch in app code, OR
          Restructure schema to have a single composite UNIQUE.
  No UNIQUE / PRIMARY?
    REPLACE INTO is equivalent to plain INSERT in that case (KB) ; just INSERT.
```

### Tree 2 : "Should I use REPLACE INTO?"

```
Is there ANY ON DELETE CASCADE on the table or child tables?
  Yes : NEVER use REPLACE. The cascade will wipe children.
        Use INSERT ... ON DUPLICATE KEY UPDATE.
  No, but :
    Is AUTO_INCREMENT used and you want a stable id?
      Yes : NEVER use REPLACE (it burns a new id each call).
            Use INSERT ... ON DUPLICATE KEY UPDATE.
    Are there DELETE triggers that do anything (audit, side effects)?
      Yes : NEVER use REPLACE (the DELETE trigger fires every call).
            Use INSERT ... ON DUPLICATE KEY UPDATE.
    None of the above?
      REPLACE INTO is "legal" but still uncommon. Default to INSERT ... ODKU.
```

### Tree 3 : "I want INSERT IGNORE for a duplicate-skip"

```
Do you understand that IGNORE suppresses ALL errors, not just duplicates?
  No : STOP. Read references/anti-patterns.md "INSERT IGNORE silent coercion".
  Yes : Are out-of-range, NOT NULL, charset, and FK errors safe to silently
        coerce-and-warn in your use case?
    No (the common case) : Use INSERT ... ON DUPLICATE KEY UPDATE id = id
      - One unique key match -> no-op UPDATE. Other errors surface normally.
      - 0 rows affected = duplicate ; 1 row = inserted ; 2 rows = updated.
    Yes (rare ; bulk import where dirty data is tolerated) : INSERT IGNORE,
      but always inspect SHOW WARNINGS after the batch and log the count.
```

### Tree 4 : "I need the inserted id / row data"

```
Single INSERT, single AUTO_INCREMENT column?
  Use LAST_INSERT_ID() right after INSERT.
  OR : INSERT INTO t (...) VALUES (...) RETURNING id, created_at  -- 10.5+

Multi-row INSERT, all generated ids?
  INSERT INTO t (...) VALUES (...), (...), (...)
    RETURNING id, ...                                              -- 10.5+
  - LAST_INSERT_ID() returns the FIRST generated id only.

DELETE and need to capture which rows you removed?
  DELETE FROM t WHERE ... RETURNING id, ...                        -- 10.0+

UPDATE and need new/old values?
  Pre-13.0 : RETURNING is NOT supported. Options :
    SELECT before, UPDATE, SELECT after in a transaction, OR
    Use a stored procedure with explicit OUT parameters, OR
    Wait for 13.0+ which adds UPDATE ... RETURNING OLD_VALUE(col).
```

## Patterns

### Single-row INSERT

```sql
-- 10.6+ : all LTS versions
INSERT INTO customer (name, email)
  VALUES ('Acme Corp', 'billing@acme.example');
```

### Multi-row INSERT

```sql
-- 10.6+ : single network round-trip, single transaction segment, single auto-increment burst
INSERT INTO customer (name, email) VALUES
  ('Acme Corp',   'billing@acme.example'),
  ('Globex Ltd',  'ap@globex.example'),
  ('Initech Inc', 'invoices@initech.example');
```

### INSERT SET (single-row, named columns)

```sql
-- 10.6+ : equivalent to INSERT INTO ... VALUES (...), more readable for wide tables
INSERT INTO customer
  SET name = 'Acme Corp',
      email = 'billing@acme.example',
      created_at = NOW();
```

### INSERT ... SELECT

```sql
-- 10.6+ : copy rows from one table to another (or same table with care)
INSERT INTO customer_archive (id, name, email, archived_at)
  SELECT id, name, email, NOW()
  FROM customer
  WHERE created_at < NOW() - INTERVAL 3 YEAR;
```

### Upsert : INSERT ... ON DUPLICATE KEY UPDATE

```sql
-- 10.6+ : the canonical upsert. Use VALUES(col) to refer to "what would have been inserted".
INSERT INTO inventory (sku, qty, last_seen)
  VALUES ('SKU-001', 10, NOW())
  ON DUPLICATE KEY UPDATE
    qty = qty + VALUES(qty),
    last_seen = VALUES(last_seen);
```

Return-code semantics : `1` row affected = inserted, `2` rows affected = updated (unless `CLIENT_FOUND_ROWS` flag is set, which inverts the second). `LAST_INSERT_ID()` is the new auto-increment id on insert, and on update is preserved only if you add `id = LAST_INSERT_ID(id)` to the SET clause.

### INSERT RETURNING

```sql
-- 10.5+ : fetch the generated id without a second round-trip
INSERT INTO customer (name) VALUES ('Acme')
  RETURNING id, created_at;
```

Restrictions : no aggregate functions, no multi-row / multi-column subqueries in the RETURNING list (KB).

### REPLACE INTO (only when safe ; default to ODKU)

```sql
-- 10.6+ : DELETE-then-INSERT. NEVER use on tables with FK CASCADE children, DELETE triggers, or stable-id requirements.
REPLACE INTO config_snapshot (key_name, value)
  VALUES ('schema_version', '42');
```

### Single-table UPDATE with ORDER BY + LIMIT

```sql
-- 10.6+ : safe batched update ; ORDER BY makes row selection deterministic
UPDATE outbox
  SET sent_at = NOW(), status = 'sent'
  WHERE status = 'pending'
  ORDER BY id
  LIMIT 1000;
```

### Multi-table UPDATE with JOIN

```sql
-- 10.6+ : update columns in one table based on another. ORDER BY and LIMIT NOT allowed in pre-10.3 multi-table form ; allowed in 10.6+.
UPDATE customer c
JOIN customer_score s ON s.customer_id = c.id
SET c.tier = s.tier
WHERE s.calculated_at > c.tier_updated_at;
```

Row update order across multi-table UPDATE is undefined. Do not rely on it.

### Multi-table DELETE with JOIN

```sql
-- 10.6+ : delete rows in t1 that have a match in t2
DELETE c
FROM customer c
JOIN gdpr_purge_request p ON p.customer_id = c.id
WHERE p.confirmed_at < NOW() - INTERVAL 30 DAY;
```

### DELETE with self-referential subquery

```sql
-- 10.6+ : MariaDB allows DELETE from a table referenced in a subquery since 10.3.1.
DELETE FROM customer
WHERE id IN (
  SELECT id FROM (
    SELECT id FROM customer WHERE created_at < NOW() - INTERVAL 10 YEAR LIMIT 1000
  ) AS to_purge
);
```

The inner derived table (`AS to_purge`) is required to avoid "You can't specify target table for update in FROM clause" on older minor versions.

### DELETE RETURNING

```sql
-- 10.0+ : capture what you removed in the same statement
DELETE FROM session
  WHERE expires_at < NOW()
  RETURNING id, user_id, expired_at;
```

### DELETE HISTORY on system-versioned tables

```sql
-- 10.3+ : purge row-history older than a date. Does NOT touch live (current) rows
-- except via the MDEV-25468 quirk noted in anti-patterns.md.
DELETE HISTORY FROM customer
  BEFORE SYSTEM_TIME '2024-01-01';
```

`DELETE HISTORY` requires the `DELETE HISTORY` privilege (separate from `DELETE`). Regular `DELETE` on a system-versioned table does NOT purge history ; it moves the live row to history with an updated `row_end`.

## Cross-References

- `mariadb-syntax-sql-ddl` : table creation, constraints, AUTO_INCREMENT, UNIQUE indexes that govern upsert behavior.
- `mariadb-syntax-indexing` : the UNIQUE index design that determines ODKU correctness.
- `mariadb-syntax-system-versioning` : system-versioned tables, `WITH SYSTEM VERSIONING`, `FOR SYSTEM_TIME AS OF`.
- `mariadb-errors-data-integrity` : strict_mode, sql_mode, FK constraint failures.

## Reference Files

- `references/methods.md` : full DML grammar, modifier table, return-value semantics, RETURNING per-statement support matrix.
- `references/examples.md` : 12 worked examples covering all patterns above.
- `references/anti-patterns.md` : 10 anti-patterns with code, "why this fails", and the correct replacement.

## Verification

All syntax in this skill verified 2026-05-19 via WebFetch against :
- mariadb.com/kb/en/insert/
- mariadb.com/kb/en/insert-on-duplicate-key-update/
- mariadb.com/kb/en/insert-ignore/
- mariadb.com/kb/en/replace/
- mariadb.com/kb/en/update/
- mariadb.com/kb/en/delete/
- mariadb.com/kb/en/system-versioned-tables/

Per LESSONS.md L-001, mariadb.com/kb/en/<topic>/ is the canonical path ; the `/docs/` rewrite is also valid.
