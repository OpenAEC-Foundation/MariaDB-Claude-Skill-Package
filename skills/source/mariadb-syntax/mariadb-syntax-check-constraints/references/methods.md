# CHECK Constraints : Complete Reference

Comprehensive grammar, semantics, version notes, and interaction tables for MariaDB CHECK constraints. All entries verified against official MariaDB KB on 2026-05-19.

## 1. Grammar (BNF-style)

### Column-level CHECK

```
column_definition ::=
    column_name data_type
    [NOT NULL | NULL]
    [DEFAULT default_value]
    [AUTO_INCREMENT]
    [UNIQUE [KEY] | [PRIMARY] KEY]
    [COMMENT 'string']
    [CHECK (check_expression)]
    [other_column_options...]
```

`check_expression` MUST evaluate to a boolean (TRUE / FALSE / NULL) and MUST be deterministic. Inline column-level CHECK has an auto-generated constraint name of the form `CONSTRAINT_<N>` unless wrapped with `CONSTRAINT name CHECK (...)`.

### Table-level CHECK

```
table_constraint ::=
    [CONSTRAINT [constraint_name]]
    CHECK (check_expression)
```

Used inside `CREATE TABLE` after the last column definition, or via `ALTER TABLE ... ADD CONSTRAINT`.

### CREATE TABLE syntax (CHECK position)

```sql
CREATE TABLE table_name (
    column_definition [, column_definition]...
    [, table_constraint]...
) [table_options];
```

### ALTER TABLE ADD CONSTRAINT

Per MariaDB KB `mariadb.com/kb/en/alter-table/` :

```sql
ALTER TABLE table_name
    ADD CONSTRAINT [constraint_name] CHECK (check_expression);
```

### ALTER TABLE DROP CONSTRAINT

```sql
ALTER TABLE table_name
    DROP CONSTRAINT constraint_name;
```

Auto-generated names follow pattern `CONSTRAINT_1`, `CONSTRAINT_2`, ... per table, allocated in declaration order. Look up the actual name via `information_schema.CHECK_CONSTRAINTS`.

## 2. NULL Semantics : Three-Valued Logic (3VL)

CHECK predicates use SQL three-valued logic. A predicate evaluates to one of `TRUE`, `FALSE`, `NULL` (unknown). A CHECK constraint **fails only on FALSE**.

| Predicate | Result | CHECK outcome |
|---|---|---|
| `5 > 0` | TRUE | passes |
| `-1 > 0` | FALSE | FAILS (row rejected) |
| `NULL > 0` | NULL | passes |
| `NULL IS NULL` | TRUE | passes |
| `NULL = NULL` | NULL | passes (this is why `IS NULL` exists) |
| `JSON_VALID('{"a":1}')` | 1 (TRUE) | passes |
| `JSON_VALID('not json')` | 0 (FALSE) | FAILS |
| `JSON_VALID(NULL)` | NULL | passes |
| `1 IN (1,2,NULL)` | TRUE | passes |
| `3 IN (1,2,NULL)` | NULL (because of NULL in list) | passes |
| `3 NOT IN (1,2,NULL)` | NULL | passes (a common surprise) |

### Practical implication

A column `balance INT CHECK (balance >= 0)` allows:

- `0`, `1`, `100`, `2147483647` (TRUE, passes)
- `NULL` (NULL predicate, passes)

It rejects `-1`, `-100`. ALWAYS combine with `NOT NULL` if you want a strictly-required positive value:

```sql
balance INT NOT NULL CHECK (balance >= 0)
```

### NOT IN trap

```sql
status_code INT CHECK (status_code NOT IN (400, 401, NULL))
-- Any value of status_code evaluates to NULL (because NULL is in the IN-list).
-- The CHECK always passes. This is almost never what you want.
```

Correct form :

```sql
status_code INT NOT NULL CHECK (status_code NOT IN (400, 401))
```

## 3. Allowed Expressions

DETERMINISTIC operations only :

- Comparison operators : `=`, `<>`, `<`, `<=`, `>`, `>=`
- Boolean operators : `AND`, `OR`, `NOT`
- Arithmetic : `+`, `-`, `*`, `/`, `%`, `DIV`, `MOD`
- String functions : `LENGTH`, `CHAR_LENGTH`, `LEFT`, `RIGHT`, `SUBSTRING`, `UPPER`, `LOWER`, `TRIM`, `CONCAT`, `REGEXP`/`RLIKE`, `LIKE`, `INSTR`
- JSON functions (deterministic only) : `JSON_VALID`, `JSON_EXTRACT`, `JSON_TYPE`, `JSON_LENGTH`, `JSON_CONTAINS`, `JSON_KEYS`
- Type-cast : `CAST(x AS type)`, `CONVERT(x, type)`
- Conditional : `CASE`, `IF`, `IFNULL`, `COALESCE`, `NULLIF`
- Math : `ABS`, `CEILING`, `FLOOR`, `ROUND`, `MOD`, `POWER`, `SQRT`
- Set membership : `IN`, `BETWEEN`, `IS NULL`, `IS NOT NULL`
- Bit operators : `&`, `|`, `^`, `~`, `<<`, `>>`

## 4. Forbidden Expressions

NEVER use any of the following in CHECK :

- **Sub-queries** : `(SELECT ...)` rejected at parse time.
- **References to other tables** : implicit via column resolution, rejected.
- **Non-deterministic functions** :
  - Time : `NOW()`, `CURRENT_TIMESTAMP`, `CURDATE`, `CURTIME`, `UNIX_TIMESTAMP()` (without argument), `SYSDATE()`
  - Random : `RAND()`, `UUID()`, `UUID_SHORT()`
  - Session/connection : `CONNECTION_ID()`, `USER()`, `CURRENT_USER()`, `SESSION_USER()`, `SYSTEM_USER()`, `LAST_INSERT_ID()`, `ROW_COUNT()`, `FOUND_ROWS()`
  - Server identity : `VERSION()`, `DATABASE()`, `SCHEMA()`
- **User-defined functions** declared `NOT DETERMINISTIC`.
- **Variables** : `@var`, `@@session.var`, `@@global.var`.
- **`auto_increment` columns** : explicitly forbidden per KB.
- **Stored procedure CALL** : CHECK is an expression, not a statement context.

If a forbidden expression is used, MariaDB raises a syntax error or a "constraint contains non-deterministic function" error at table-creation or ALTER time.

## 5. Constraint Naming

| Form | Name allocation | Droppable by exact name |
|---|---|---|
| `col INT CHECK (col > 0)` | auto : `CONSTRAINT_<N>` | YES (but lookup needed) |
| `CONSTRAINT my_chk CHECK (col > 0)` (table-level) | explicit : `my_chk` | YES |
| `col INT CONSTRAINT my_chk CHECK (col > 0)` (column-level) | explicit : `my_chk` | YES |

ALWAYS name CHECK constraints. Anonymous CHECKs are dropped by looking up `information_schema.CHECK_CONSTRAINTS`, which is a runtime indirection that fails in DDL replay scripts.

### Lookup query

```sql
-- 10.2.1+
SELECT CONSTRAINT_NAME, CHECK_CLAUSE
  FROM information_schema.CHECK_CONSTRAINTS
 WHERE CONSTRAINT_SCHEMA = DATABASE()
   AND TABLE_NAME = 'your_table';
```

## 6. INSERT IGNORE / UPDATE IGNORE Interaction

| Statement | CHECK violation result |
|---|---|
| `INSERT` | Hard ERROR, no row inserted, error code 4025 |
| `INSERT IGNORE` | WARNING, row NOT inserted, no error raised |
| `UPDATE` | Hard ERROR, no rows updated, error code 4025 |
| `UPDATE IGNORE` | WARNING, the violating row is skipped, other rows are updated |
| `REPLACE` | Hard ERROR, behaves like INSERT |
| `INSERT ... ON DUPLICATE KEY UPDATE` | Hard ERROR on the UPDATE branch |
| `LOAD DATA INFILE` | Hard ERROR by default |
| `LOAD DATA INFILE ... IGNORE` | WARNING per row, violating rows skipped |

Per `mariadb.com/kb/en/insert-ignore/` : "By using the IGNORE keyword all errors are converted to warnings, which will not stop inserts of additional rows." CHECK constraint failures are included in this set. The row that violated CHECK is NOT inserted.

## 7. sql_mode Interactions

CHECK enforcement is NOT controlled by `sql_mode` in MariaDB 10.2.1+. It is always on unless globally disabled via `check_constraint_checks`. The following sql_mode flags interact only indirectly :

| sql_mode flag | Effect on CHECK |
|---|---|
| `STRICT_TRANS_TABLES` | Independent : controls implicit coercion errors, not CHECK |
| `STRICT_ALL_TABLES` | Same as above |
| `NO_ZERO_DATE` | Independent : a CHECK on a DATE column does not bypass this |
| `IGNORE_SPACE` | Independent |
| `ANSI_QUOTES` | Affects identifier parsing only |

In other words : CHECK is enforced regardless of `sql_mode`. The only override is `check_constraint_checks = OFF`.

## 8. The `check_constraint_checks` Server Variable

```sql
-- session scope
SET SESSION check_constraint_checks = OFF;
-- ... load data ...
SET SESSION check_constraint_checks = ON;

-- global scope (requires SUPER / SYSTEM_VARIABLES_ADMIN)
SET GLOBAL check_constraint_checks = OFF;
```

| Value | Behavior |
|---|---|
| `ON` (default) | CHECK constraints are enforced on every INSERT/UPDATE |
| `OFF` | CHECK constraints are skipped at runtime. Existing constraint definitions remain in the schema. |

CRITICAL : re-enabling does NOT re-validate existing rows. Use only for trusted bulk loads followed by manual audit.

## 9. CHECK and Generated Columns

| Generated column type | CHECK evaluation timing |
|---|---|
| `VIRTUAL` (computed on read) | Evaluated on row write if CHECK references the generated column |
| `PERSISTENT` / `STORED` (materialized) | Evaluated on row write only |

When the underlying column of a `PERSISTENT` generated column is modified via `ALTER TABLE MODIFY COLUMN`, MariaDB recomputes the generated column for all existing rows and CHECK is re-evaluated.

## 10. CHECK on Foreign-Key Columns

CHECK and FOREIGN KEY are independent constraint types. A column can have both :

```sql
status_id INT NOT NULL,
CONSTRAINT chk_status_id_positive CHECK (status_id > 0),
FOREIGN KEY (status_id) REFERENCES statuses (id)
```

The two constraints are evaluated in declaration order (per `mariadb.com/kb/en/constraint/`). A failed CHECK aborts the row before FK lookup.

## 11. CHECK Across Storage Engines

| Engine | CHECK supported | Notes |
|---|---|---|
| InnoDB (default) | YES | Full enforcement |
| Aria | YES | Full enforcement |
| MyISAM | YES | Full enforcement |
| MEMORY | YES | Full enforcement |
| CONNECT | LIMITED | Only on local table types ; remote tables defer to the remote source |
| Spider | NO | Predicates pushed to backend, CHECK on Spider table is ignored |
| Archive | YES | Enforced on INSERT, but UPDATE is not allowed so this is moot |

## 12. Information Schema and SHOW

```sql
-- list all CHECK constraints in current database
SELECT TABLE_NAME, CONSTRAINT_NAME, CHECK_CLAUSE
  FROM information_schema.CHECK_CONSTRAINTS
 WHERE CONSTRAINT_SCHEMA = DATABASE();

-- inspect CREATE TABLE output
SHOW CREATE TABLE accounts;
```

`SHOW CREATE TABLE` emits each CHECK constraint with its (auto-generated or explicit) name.

## 13. Version Matrix

| Version | CHECK behavior |
|---|---|
| <= 10.1.x | Parsed, ignored (no enforcement) |
| 10.2.1 - 10.2.x | Enforced. First version with full CHECK semantics. |
| 10.3 - 10.4.2 | Enforced. JSON alias is LONGTEXT without auto-CHECK. |
| 10.4.3+ | JSON alias auto-applies `JSON_VALID` as a CHECK. |
| 10.6 LTS | Stable, recommended baseline. |
| 10.11 LTS | Stable, current LTS. |
| 11.x current | Stable. |
| 12.x next | Stable. |

## 14. Error Reference

| Error code | SQLSTATE | Message (truncated) | When raised |
|---|---|---|---|
| 4025 | HY000 | `CONSTRAINT '%s' failed for '%s'.'%s'` | INSERT/UPDATE violates a CHECK |
| 4026 | HY000 | `Expression for field '%s' is referring to uninitialized field '%s'` | Generated-column CHECK refs uninitialized column |
| 3818 | HY000 | `Check constraint '%s' uses non-deterministic function` | CREATE TABLE / ALTER TABLE with NOW(), etc. |
| 3814 | HY000 | `An expression of a check constraint contains disallowed function` | UUID(), CURRENT_USER, etc. |

Full anti-pattern reproductions are in `anti-patterns.md`.

## 15. Migration Notes

### From MySQL 8 -> MariaDB

- All `CHECK` clauses parse identically.
- `NOT ENFORCED` (MySQL 8 syntax) is parsed but treated as ENFORCED in MariaDB. Audit migration scripts.
- JSON columns lose native binary storage : ADD `CHECK (JSON_VALID(col))` to every JSON column to preserve the validation invariant.

### From PostgreSQL -> MariaDB

- PostgreSQL `CHECK` is also row-local. Cross-table CHECK in PostgreSQL via sub-query is not portable to MariaDB.
- PostgreSQL `NOT VALID` clause has no MariaDB equivalent ; use `check_constraint_checks = OFF` for one-time bulk load.

### From MariaDB <= 10.1 -> 10.2+

- Existing CHECK clauses that were parsed-but-ignored are NOW enforced. Audit rows that may have been inserted under the previous regime.
- Run `SELECT * FROM t WHERE NOT (check_expression)` per CHECK to find pre-existing violations before upgrade.
