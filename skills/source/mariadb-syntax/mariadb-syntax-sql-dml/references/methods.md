# MariaDB DML : Complete Grammar Reference

All syntax verified 2026-05-19 against mariadb.com/kb/en/.

## INSERT

### Full grammar

```
INSERT [LOW_PRIORITY | DELAYED | HIGH_PRIORITY] [IGNORE]
    [INTO] tbl_name [PARTITION (partition_list)]
    [(col,...)]
    {VALUES | VALUE} ({expr | DEFAULT},...),(...),...
    [ ON DUPLICATE KEY UPDATE
      col=expr
        [, col=expr] ... ]
    [RETURNING select_expr [, select_expr ...]]
```

Alternate SET form :

```
INSERT [LOW_PRIORITY | DELAYED | HIGH_PRIORITY] [IGNORE]
    [INTO] tbl_name [PARTITION (partition_list)]
    SET col={expr | DEFAULT}, ...
    [ ON DUPLICATE KEY UPDATE
      col=expr [, col=expr] ... ]
    [RETURNING select_expr [, select_expr ...]]
```

Alternate SELECT form :

```
INSERT [LOW_PRIORITY | HIGH_PRIORITY] [IGNORE]
    [INTO] tbl_name [PARTITION (partition_list)]
    [(col,...)]
    SELECT ...
    [ ON DUPLICATE KEY UPDATE
      col=expr [, col=expr] ... ]
```

### Modifier semantics

| Modifier | Effect | Applies to engines |
|----------|--------|--------------------|
| `LOW_PRIORITY` | Delay until no client is reading the table | Table-level locking : MyISAM, MEMORY, MERGE. Ignored for InnoDB. |
| `HIGH_PRIORITY` | Same priority as SELECTs | Table-level locking engines only. |
| `DELAYED` | Asynchronous queue ; client returns immediately | MyISAM, MEMORY only ; deprecated in modern InnoDB-only deployments. IGNORED when combined with ON DUPLICATE KEY UPDATE. |
| `IGNORE` | Convert errors to warnings ; see anti-patterns.md | All engines. NEVER combined with ON DUPLICATE KEY UPDATE (IGNORE is silently dropped per KB). |

### AUTO_INCREMENT and LAST_INSERT_ID

- A new row INSERT generates a new auto_increment value. `LAST_INSERT_ID()` returns that value within the same session, even after subsequent statements that do not generate ids.
- Multi-row INSERT : `LAST_INSERT_ID()` returns the FIRST generated id only. The rest are contiguous when `auto_increment_increment = 1`.
- `INSERT ... ON DUPLICATE KEY UPDATE` on the UPDATE branch : LAST_INSERT_ID() returns 0 unless you write `id = LAST_INSERT_ID(id)` in the SET clause, which makes it return the existing row's id.
- AUTO_INCREMENT burn : on the UPDATE branch the auto_increment counter still advances. This produces gaps and is by design (avoiding the burn requires a SELECT-then-INSERT pattern outside the statement).

### RETURNING (INSERT)

- Added in MariaDB 10.5.0.
- Available for `INSERT ... VALUES`, `INSERT ... SET`, `INSERT ... SELECT`, `REPLACE INTO` (all forms).
- Restrictions : no aggregate functions, no multi-row or multi-column subqueries in the RETURNING list. Virtual columns, stored functions, and SQL expressions are allowed.
- Engine support : works on InnoDB, MyISAM, Aria. Storage-engine-specific gotchas should be verified per engine.

## UPDATE

### Single-table grammar

```
UPDATE [LOW_PRIORITY] [IGNORE] table_reference
    SET col1={expr1 | DEFAULT} [, col2={expr2 | DEFAULT}] ...
    [WHERE where_condition]
    [ORDER BY ...]
    [LIMIT row_count]
```

### Multi-table grammar

```
UPDATE [LOW_PRIORITY] [IGNORE] table_references
    SET col1={expr1 | DEFAULT} [, col2={expr2 | DEFAULT}] ...
    [WHERE where_condition]
```

ORDER BY and LIMIT are NOT allowed in multi-table UPDATE in pre-10.3 versions. From 10.3+ they ARE permitted, but row update order is still undefined across the joined rowset.

### Modifier semantics

| Modifier | Effect |
|----------|--------|
| `LOW_PRIORITY` | Delay until no client reads the table (table-level locking engines only). |
| `IGNORE` | Suppresses errors : duplicate-key conflicts skip the row, type-conversion errors substitute the closest valid value. NEVER use IGNORE on production-critical updates. |

### RETURNING (UPDATE)

- **NOT supported in MariaDB 10.6-LTS, 10.11-LTS, 11.x, or 12.x.**
- Available only from MariaDB 13.0 onwards (KB-verified).
- Syntax (13.0+) : `RETURNING OLD_VALUE(col) AS old, col AS new` ; OLD_VALUE() returns the pre-update value.
- For LTS-targeted skill development, the absence of UPDATE RETURNING is the rule. Wrap pre-image SELECT + UPDATE + post-image SELECT in a transaction if needed.

### ROW_COUNT()

- Returns the number of rows that were actually changed.
- If a row's new values equal its existing values, the row is NOT counted as changed (default behavior).
- With CLIENT_FOUND_ROWS protocol flag set, ROW_COUNT() returns the number of rows matched by the WHERE clause regardless of change.

## DELETE

### Single-table grammar

```
DELETE [LOW_PRIORITY] [QUICK] [IGNORE]
    FROM tbl_name [PARTITION (partition_list)]
    [FOR PORTION OF PERIOD FROM expr1 TO expr2]
    [AS alias]                                    -- 11.6+ for single-table aliases
    [WHERE where_condition]
    [ORDER BY ...]
    [LIMIT row_count]
    [RETURNING select_expr [, ...]]
```

### Multi-table grammar (two equivalent forms)

```
DELETE [LOW_PRIORITY] [QUICK] [IGNORE]
    tbl_name[.*] [, tbl_name[.*]] ...
    FROM table_references
    [WHERE where_condition]
```

```
DELETE [LOW_PRIORITY] [QUICK] [IGNORE]
    FROM tbl_name[.*] [, tbl_name[.*]] ...
    USING table_references
    [WHERE where_condition]
```

Multi-table DELETE does NOT support `ORDER BY`, `LIMIT`, or `RETURNING`.

### Modifier semantics

| Modifier | Effect |
|----------|--------|
| `LOW_PRIORITY` | Delay until no client reads the table (table-level locking engines only). |
| `QUICK` | MyISAM-only hint : do not merge index leaves after the delete. Speeds up bulk deletes ; produces fragmentation. |
| `IGNORE` | Suppress errors (FK violations during multi-table delete, etc). Use with caution. |

### RETURNING (DELETE)

- Added in MariaDB 10.0 (the first RETURNING support in MariaDB).
- Single-table DELETE only ; multi-table DELETE cannot use RETURNING.
- Returns the deleted rows' columns. Useful for capturing audit data in the same round-trip.

### FOR PORTION OF PERIOD

- Application-time period support (MariaDB-specific). Deletes only the time-slice of a row that falls within the specified period ; the remainder stays as a shortened row.
- Requires the table to be defined `WITH PERIOD FOR ...`.

### DELETE HISTORY

```
DELETE HISTORY FROM tbl_name
    [BEFORE SYSTEM_TIME { timestamp_expr | TRANSACTION trx_id }]
```

- Operates ONLY on system-versioned tables (declared `WITH SYSTEM VERSIONING`).
- Removes historical row-versions only ; current ("live") rows are untouched, with one caveat : MDEV-25468 documents that `BEFORE SYSTEM_TIME > row_end` of active records causes active rows to also be moved to history then dropped.
- Requires a separate `DELETE HISTORY` privilege ; regular `DELETE` privilege is insufficient.

## REPLACE

### Grammar

```
REPLACE [LOW_PRIORITY | DELAYED]
    [INTO] tbl_name [PARTITION (partition_list)]
    [(col,...)]
    {VALUES | VALUE} ({expr | DEFAULT},...),(...),...
    [RETURNING select_expr [, ...]]
```

Alternate forms : `REPLACE ... SET col=expr` and `REPLACE ... SELECT`.

### Semantics

- Conceptually : `BEGIN ; SELECT 1 FROM t WHERE KEY=# FOR UPDATE ; IF FOUND DELETE FROM t WHERE KEY=# ; INSERT INTO t VALUES (...) ; END ;` (KB-verbatim conceptual model).
- Conflict triggers : PRIMARY KEY or any UNIQUE index match. If the table has neither, REPLACE behaves identically to plain INSERT.
- Side effects per call when conflict matches :
  1. `ON DELETE CASCADE` fires through child tables.
  2. Both DELETE and INSERT triggers fire (DELETE first, then INSERT).
  3. AUTO_INCREMENT consumes a new value, even if the conflict matched.
- RETURNING supported since 10.5.0.

## Return-value semantics summary

| Statement | Rows affected | LAST_INSERT_ID() |
|-----------|--------------|------------------|
| INSERT (new row) | 1 | new auto_inc value |
| INSERT (multi-row, N new) | N | first generated id |
| INSERT IGNORE (skipped duplicate) | 0 | unchanged |
| INSERT ... ODKU (insert branch) | 1 | new auto_inc value |
| INSERT ... ODKU (update branch, no change) | 0 | unchanged (or row id if SET id = LAST_INSERT_ID(id)) |
| INSERT ... ODKU (update branch, with change) | 2 | unchanged (or row id with idiom) |
| REPLACE (no conflict) | 1 | new auto_inc value |
| REPLACE (conflict, replaced) | 2 | new auto_inc value (BURNED) |
| UPDATE (matched, changed) | N | unchanged |
| UPDATE (matched, no change) | 0 | unchanged |
| DELETE | N | unchanged |
| DELETE HISTORY | N | unchanged |

CLIENT_FOUND_ROWS protocol flag changes some of these counts ; default protocol applies in mariadb-client and most drivers.

## RETURNING support matrix (LTS focus)

| Statement | 10.6-LTS | 10.11-LTS | 11.x | 12.x | 13.0+ |
|-----------|----------|-----------|------|------|-------|
| INSERT RETURNING | yes (since 10.5) | yes | yes | yes | yes |
| REPLACE RETURNING | yes (since 10.5) | yes | yes | yes | yes |
| DELETE RETURNING | yes (since 10.0) | yes | yes | yes | yes |
| UPDATE RETURNING | NO | NO | NO | NO | yes |

The UPDATE RETURNING gap is the single biggest discontinuity vs PostgreSQL / SQL Server. Code that targets MariaDB LTS MUST NOT use it.
