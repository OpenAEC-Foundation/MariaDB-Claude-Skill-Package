# Methods Reference : MariaDB Stored Procedures and Stored Functions

Complete grammar, characteristic matrix, SQLSTATE catalogue, and system variables. All grammar verified against MariaDB KB on 2026-05-19.

---

## 1. CREATE PROCEDURE grammar

### MariaDB 10.6 through 11.7

```
CREATE [OR REPLACE]
  [DEFINER = { user | CURRENT_USER | role | CURRENT_ROLE }]
  PROCEDURE [IF NOT EXISTS] sp_name ([ proc_parameter [, ...] ])
  [ characteristic ... ]
  routine_body

proc_parameter:
  [ IN | OUT | INOUT ] param_name type

characteristic:
    LANGUAGE SQL
  | [NOT] DETERMINISTIC
  | { CONTAINS SQL | NO SQL | READS SQL DATA | MODIFIES SQL DATA }
  | SQL SECURITY { DEFINER | INVOKER }
  | COMMENT 'string'

routine_body:
  Valid SQL procedure body, typically BEGIN ... END
```

### MariaDB 11.8+ additions (Oracle-mode extensions)

```
proc_parameter:
    [ OUT | INOUT | IN OUT ] param_name type
  | [ IN ] param_name type [ DEFAULT value_or_expression ]
```

- `IN OUT` is a synonym of `INOUT` enabling Oracle-style syntax.
- `DEFAULT value_or_expression` is allowed ONLY on `IN` parameters (not `OUT` / `INOUT`).
- Backward-incompatible : 10.6-LTS through 11.7 reject `DEFAULT` and `IN OUT` as syntax errors.

### Call syntax

```sql
CALL sp_name ([ argument [, ...] ]);
-- OUT and INOUT parameters require a user variable as the argument :
CALL p(123, @out_value);
SELECT @out_value;
```

### Privilege requirements

- `CREATE ROUTINE` to create.
- The creator is automatically granted `ALTER ROUTINE` and `EXECUTE` on the new routine.
- `EXECUTE` on the routine to call it.
- With `SQL SECURITY DEFINER` (the default), the routine body runs with the DEFINER's privileges.
- With `SQL SECURITY INVOKER`, privilege checks defer to the caller.

---

## 2. CREATE FUNCTION grammar

### MariaDB 10.6 through 11.7

```
CREATE [OR REPLACE]
  [DEFINER = { user | CURRENT_USER | role | CURRENT_ROLE }]
  [AGGREGATE] FUNCTION [IF NOT EXISTS] func_name ([ func_parameter [, ...] ])
  RETURNS type
  [ characteristic ... ]
  func_body

func_parameter (10.6 through 10.7):
  param_name type

func_parameter (10.8+, OUT/INOUT only callable from SET):
    [ OUT | INOUT ] param_name type
  | [ IN ] param_name type
```

### MariaDB 11.8+

```
func_parameter:
    [ OUT | INOUT | IN OUT ] param_name type
  | [ IN ] param_name type [ DEFAULT value_or_expression ]
```

### RETURNS and RETURN

- `RETURNS type` is MANDATORY for every function.
- Body MUST execute `RETURN expr ;` to produce a value.
- Multiple `RETURN` statements are allowed (one per branch) ; the first reached terminates the function.
- Type coercion : with `SQL_MODE` including `STRICT_ALL_TABLES` or `STRICT_TRANS_TABLES`, a type mismatch on the returned value raises error 1366. Otherwise, values are coerced.

### Function call syntax

```sql
-- Standard (IN-only parameters) :
SELECT func_name(arg1, arg2);
SET @x = func_name(arg1, arg2);
WHERE col = func_name(arg);

-- OUT / INOUT parameters (10.8+) : SET-only, CANNOT be used in SELECT :
SET @out = NULL;
SET @ret = func_with_out(@in, @out);
-- SELECT func_with_out(@in, @out) ;   -- ERROR 4186
```

### AGGREGATE FUNCTION

`CREATE AGGREGATE FUNCTION ...` defines a custom aggregate (used with `GROUP BY`). The body uses a `FETCH GROUP NEXT ROW ;` cursor-like iteration pattern. Out of scope for this skill ; covered separately if needed.

---

## 3. Characteristic clause matrix

| Clause | Procedure | Function | Default | Effect |
|--------|-----------|----------|---------|--------|
| `LANGUAGE SQL` | yes | yes | `SQL` | Informational. MariaDB supports only SQL. |
| `[NOT] DETERMINISTIC` | yes (informative) | yes (load-bearing) | `NOT DETERMINISTIC` | Function : optimizer may cache / replicate under STATEMENT binlog. Procedure : informative only. |
| `CONTAINS SQL` | yes | yes | yes (default) | Body contains SQL that neither reads nor writes data (e.g. `SET`, `DO`). |
| `NO SQL` | yes | yes | no | Asserts no SQL inside. Currently informative ; MariaDB has no non-SQL routine language. |
| `READS SQL DATA` | yes | yes | no | Body contains `SELECT` but no DML / DDL. |
| `MODIFIES SQL DATA` | yes | yes | no | Body contains `INSERT` / `UPDATE` / `DELETE` / `REPLACE` / DDL. |
| `SQL SECURITY DEFINER` | yes | yes | yes (default) | Body runs with DEFINER privileges. |
| `SQL SECURITY INVOKER` | yes | yes | no | Body runs with caller privileges. |
| `COMMENT 'string'` | yes | yes | empty | Free-text description visible in `SHOW CREATE`. |

**Important** : When `log_bin_trust_function_creators = OFF` (the secure default), `CREATE FUNCTION` is REFUSED unless one of (`DETERMINISTIC`, `NO SQL`, `READS SQL DATA`) is set, OR the user holds the `SUPER` / `BINLOG ADMIN` privilege. This prevents non-deterministic side-effects from being silently replicated.

---

## 4. Local DECLARE statements (inside BEGIN ... END)

Declaration order in a routine body is STRICT :

1. Local variables : `DECLARE var_name type [DEFAULT expr] ;`
2. Named conditions : `DECLARE cond_name CONDITION FOR { SQLSTATE 'xxxxx' | mariadb_error_code } ;`
3. Cursors : `DECLARE cursor_name CURSOR FOR <select_statement> ;`
4. Handlers : `DECLARE { CONTINUE | EXIT } HANDLER FOR <condition_list> <statement> ;`

Reversing the order is a syntax error.

### Local variable scoping

- Local variables are visible only within the `BEGIN ... END` block where declared (and nested blocks).
- Local variable names are distinguished from session variables : `var_name` vs `@var_name`.
- Default initialization is `NULL` if no `DEFAULT` clause.

---

## 5. Control flow statements

### IF

```sql
IF condition THEN
  statements;
[ ELSEIF condition THEN
  statements; ]
[ ELSE
  statements; ]
END IF;
```

### CASE (two forms)

```sql
-- Simple CASE
CASE expr
  WHEN value1 THEN statements;
  WHEN value2 THEN statements;
  [ ELSE statements; ]
END CASE;

-- Searched CASE
CASE
  WHEN condition1 THEN statements;
  WHEN condition2 THEN statements;
  [ ELSE statements; ]
END CASE;
```

### LOOP (unconditional, needs LEAVE)

```sql
[ label: ] LOOP
  statements;
  IF exit_condition THEN LEAVE label; END IF;
END LOOP [ label ];
```

### WHILE (test-before)

```sql
[ label: ] WHILE condition DO
  statements;
END WHILE [ label ];
```

### REPEAT (test-after)

```sql
[ label: ] REPEAT
  statements;
UNTIL condition END REPEAT [ label ];
```

### ITERATE and LEAVE

- `LEAVE label ;` exits the labelled block. Works with `LOOP`, `WHILE`, `REPEAT`, `BEGIN ... END`.
- `ITERATE label ;` restarts the labelled loop from the top. Valid only with `LOOP`, `WHILE`, `REPEAT`.

---

## 6. Cursor lifecycle

```sql
-- Declaration order : variable, then handler-flag, then cursor, then handler.
DECLARE v_id   INT;
DECLARE done   INT DEFAULT 0;
DECLARE my_cur CURSOR FOR SELECT id FROM t WHERE ... ;
DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = 1;

OPEN my_cur;
read_loop: LOOP
  FETCH my_cur INTO v_id;
  IF done = 1 THEN LEAVE read_loop; END IF;
  -- process row
END LOOP;
CLOSE my_cur;
```

### Restrictions

- Cursors are READ-ONLY (no positioned `UPDATE WHERE CURRENT OF`).
- Cursors are NON-SCROLLABLE (only forward `FETCH`).
- Cursors are ASENSITIVE (the server decides whether modifications to the underlying table are visible during iteration ; do not rely either way).
- The SELECT cannot contain an `INTO` clause (the `INTO` belongs to the `FETCH`).
- Cursor query cannot be a parameter or variable (unless 10.8+ prepared-statement cursor or 12.3+ dynamic cursor).

### MariaDB 11.8+ : SYS_REFCURSOR

Oracle-mode reference cursors allow returning a cursor handle from a function or as an OUT parameter. Not covered in detail here ; see KB `sys_refcursor/` if you target 11.8+ exclusively.

---

## 7. DECLARE HANDLER full grammar

```
DECLARE handler_type HANDLER
  FOR condition_value [, condition_value] ...
  statement

handler_type:
    CONTINUE
  | EXIT
  | UNDO     -- listed in syntax but NOT implemented in MariaDB ; do not use

condition_value:
    SQLSTATE [VALUE] 'xxxxx'
  | condition_name             -- declared via DECLARE ... CONDITION FOR ...
  | SQLWARNING                 -- shorthand : class '01...'
  | NOT FOUND                  -- shorthand : class '02...'
  | SQLEXCEPTION               -- shorthand : every class except '00', '01', '02'
  | mariadb_error_code         -- e.g. 1062 = duplicate key
```

### Handler precedence (most-to-least specific)

1. Numeric error code handler (highest priority).
2. SQLSTATE handler.
3. Class shorthand (`SQLEXCEPTION`, `SQLWARNING`, `NOT FOUND`).

If multiple handlers in nested blocks match, the innermost-block handler wins.

### Scope

- Declared inside a `BEGIN ... END` block.
- Active while execution is inside that block (including nested blocks unless overridden).
- `EXIT` handler : after the handler statement runs, the enclosing block terminates as if `END` were reached.
- `CONTINUE` handler : after the handler statement runs, execution resumes at the statement AFTER the one that raised the condition.

### Re-entrancy

A handler that is currently active cannot itself catch a new occurrence of the same condition ; this prevents infinite-loop traps inside a buggy handler.

---

## 8. SIGNAL grammar and signal_information_item

```
SIGNAL { SQLSTATE [VALUE] 'xxxxx' | condition_name }
  [ SET signal_information_item = expr [, ...] ]
```

### signal_information_item (full list)

- `MESSAGE_TEXT` : the human-readable error message.
- `MYSQL_ERRNO` : the numeric error code (1 through 65535 ; MUST NOT be 0).
- `CLASS_ORIGIN`, `SUBCLASS_ORIGIN`.
- `CONSTRAINT_CATALOG`, `CONSTRAINT_SCHEMA`, `CONSTRAINT_NAME`.
- `CATALOG_NAME`, `SCHEMA_NAME`, `TABLE_NAME`, `COLUMN_NAME`.
- `CURSOR_NAME`.

### SQLSTATE class semantics

- `'00xxx'` : success. NEVER use with SIGNAL (will be rejected).
- `'01xxx'` : warning. SIGNAL with these classes raises a warning, not an error.
- `'02xxx'` : NOT FOUND. Used internally by cursor end-of-data.
- `'45000'` : the canonical class for unhandled user-defined errors. ALWAYS use `'45000'` for application errors unless you have a specific reason to use another class.
- Other `'XXxxx'` ranges : standard ANSI / ODBC reserved. Reusing them can confuse drivers.

---

## 9. RESIGNAL grammar

```
RESIGNAL [ { SQLSTATE [VALUE] 'xxxxx' | condition_name } ]
         [ SET signal_information_item = expr [, ...] ]
```

- ONLY valid inside an active handler. Outside : error 1645 "RESIGNAL when handler not active".
- With no arguments : re-raises the caught condition unchanged.
- With `SET ...` only : re-raises with overridden fields ; unspecified fields retain the original error values.
- With a new `SQLSTATE` : raises a new condition that REPLACES the caught one. Use sparingly ; loses original context.
- Unlike `SIGNAL`, `RESIGNAL` does NOT empty the diagnostics area ; it APPENDS another condition.

---

## 10. Relevant SQLSTATE codes (procedure context)

| SQLSTATE | Numeric | Meaning | Common cause |
|----------|---------|---------|--------------|
| `'02000'` | 1329 | NO_DATA / NOT FOUND | FETCH past last row, SELECT INTO no rows |
| `'21000'` | 1242 | Cardinality violation | Scalar subquery returns >1 row |
| `'22001'` | 1406 | Data too long | VARCHAR / CHAR too short for value |
| `'22003'` | 1264 | Numeric out of range | INT overflow with STRICT mode |
| `'22007'` | 1411 | Invalid datetime | Cannot parse '2025-13-01' |
| `'22012'` | 1365 | Division by zero | Under ERROR_FOR_DIVISION_BY_ZERO SQL_MODE |
| `'23000'` | 1062 / 1452 / 1451 | Integrity constraint violation | Duplicate key, FK constraint failure |
| `'40001'` | 1213 | Deadlock | InnoDB transaction rollback after deadlock detection |
| `'42000'` | 1064 | Syntax error or access violation | Bad SQL inside dynamic EXECUTE |
| `'42S02'` | 1146 | Table not found | DROP / SELECT a missing table |
| `'42S22'` | 1054 | Column not found | Misspelled column |
| `'45000'` | (varies) | Unhandled user-defined exception | What you SIGNAL for application errors |
| `'HY000'` | (varies) | General error | Catch-all class for many runtime errors |

Verified against KB `mariadb-error-codes/` cross-referenced with `sql-state-errors/`.

---

## 11. System variables relevant to routines

| Variable | Default | Range | Effect |
|----------|---------|-------|--------|
| `max_sp_recursion_depth` | 0 | 0..255 | Maximum stored-procedure recursion. 0 forbids recursion. Set per-session before calling a recursive procedure. |
| `log_bin_trust_function_creators` | OFF | ON/OFF | When OFF (default), `CREATE FUNCTION` requires a deterministic-or-readonly characteristic OR SUPER. When ON, any user with `CREATE ROUTINE` can create non-deterministic functions. |
| `completion_type` | NO_CHAIN | NO_CHAIN / CHAIN / RELEASE | Behaviour of `COMMIT` and `ROLLBACK` inside procedures : CHAIN starts a new transaction, RELEASE disconnects. |
| `automatic_sp_privileges` | ON | ON/OFF | When ON, the creator is automatically granted EXECUTE and ALTER ROUTINE on the new routine. |
| `stored_program_cache` | 256 | 16..524288 | Number of stored programs the server caches per session before reparsing from disk. |

---

## 12. SHOW and INFORMATION_SCHEMA introspection

```sql
-- List routines in a database
SHOW PROCEDURE STATUS WHERE Db = 'mydb';
SHOW FUNCTION STATUS  WHERE Db = 'mydb';

-- View the body
SHOW CREATE PROCEDURE mydb.archive_orders_before;
SHOW CREATE FUNCTION  mydb.net_price;

-- Programmatic introspection
SELECT ROUTINE_NAME, ROUTINE_TYPE, IS_DETERMINISTIC,
       SQL_DATA_ACCESS, SECURITY_TYPE, DEFINER, CREATED, LAST_ALTERED
FROM   INFORMATION_SCHEMA.ROUTINES
WHERE  ROUTINE_SCHEMA = 'mydb';

-- Parameters
SELECT *
FROM   INFORMATION_SCHEMA.PARAMETERS
WHERE  SPECIFIC_SCHEMA = 'mydb' AND SPECIFIC_NAME = 'archive_orders_before';
```

---

## 13. DROP and ALTER

```sql
-- Drop
DROP PROCEDURE [IF EXISTS] sp_name;
DROP FUNCTION  [IF EXISTS] func_name;

-- ALTER : only changes characteristics, NOT the body.
-- To change the body, DROP + CREATE (use CREATE OR REPLACE).
ALTER PROCEDURE sp_name
  [ characteristic ... ];
ALTER FUNCTION func_name
  [ characteristic ... ];

-- To redefine the body in one statement (atomic if same connection) :
CREATE OR REPLACE PROCEDURE sp_name (...) ... BEGIN ... END;
```

`CREATE OR REPLACE` preserves grants on the routine ; a plain `DROP + CREATE` does not.
