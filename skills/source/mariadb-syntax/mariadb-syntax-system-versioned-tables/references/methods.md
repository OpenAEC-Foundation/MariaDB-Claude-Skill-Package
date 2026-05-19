# Methods Reference : System-Versioned Tables and Application-Time Periods

Complete grammar, system variables, privileges, and version matrix. Verified against `mariadb.com/kb/en/system-versioned-tables/`, `mariadb.com/kb/en/application-time-periods/`, and `mariadb.com/kb/en/bitemporal-tables/` on 2026-05-19.

## 1. SYSTEM VERSIONING : table-option grammar

### 1.1 CREATE TABLE syntax

```
CREATE TABLE name (
    column_definition_list
    [, PERIOD FOR SYSTEM_TIME (start_col, end_col) ]
) WITH SYSTEM VERSIONING
  [ partition_clause ];
```

`column_definition_list` for the period columns (optional ; auto-generated if omitted) :

```
start_col {TIMESTAMP(6) | BIGINT UNSIGNED} GENERATED ALWAYS AS ROW START [INVISIBLE],
end_col   {TIMESTAMP(6) | BIGINT UNSIGNED} GENERATED ALWAYS AS ROW END   [INVISIBLE]
```

Rules :

- Default precision when omitted : `BIGINT UNSIGNED` (transaction-id).
- Both period columns MUST share the same data type.
- `INVISIBLE` excludes the column from `SELECT *` but it remains addressable by name.
- The two columns MUST be paired in a single `PERIOD FOR SYSTEM_TIME (...)` declaration.

### 1.2 ALTER TABLE operations (10.3+)

```sql
ALTER TABLE t ADD  SYSTEM VERSIONING;
ALTER TABLE t DROP SYSTEM VERSIONING;
```

Adding explicit columns to an already-versioned table :

```sql
ALTER TABLE t
  ADD COLUMN row_start TIMESTAMP(6) GENERATED ALWAYS AS ROW START,
  ADD COLUMN row_end   TIMESTAMP(6) GENERATED ALWAYS AS ROW END,
  ADD PERIOD FOR SYSTEM_TIME (row_start, row_end),
  ADD SYSTEM VERSIONING;
```

Conversion from implicit to explicit row_start/row_end is in-place from 10.7 onwards in many cases ; full no-rebuild support landed in 11.7. Older versions perform a full table rebuild.

### 1.3 Column-level exclusion

```sql
CREATE TABLE t (
  x INT,
  y INT WITHOUT SYSTEM VERSIONING
) WITH SYSTEM VERSIONING;

-- equivalent
CREATE TABLE t (
  x INT WITH SYSTEM VERSIONING,
  y INT
);
```

`WITHOUT SYSTEM VERSIONING` columns do NOT participate in row-history (UPDATE on them does not produce a history row if every changed column is excluded).

## 2. FOR SYSTEM_TIME : query-time grammar

```
FOR SYSTEM_TIME AS OF { TIMESTAMP expr | TRANSACTION expr | expr }
FOR SYSTEM_TIME BETWEEN start_expr AND end_expr
FOR SYSTEM_TIME FROM start_expr TO end_expr
FOR SYSTEM_TIME ALL
```

Placement : immediately after the table reference, before joins and `WHERE` :

```sql
SELECT t.id, j.label
  FROM t FOR SYSTEM_TIME AS OF TIMESTAMP '2026-01-01'
  JOIN  j ON j.id = t.j_id
  WHERE t.status = 'OPEN';
```

Inclusivity :

| Form | Inclusivity |
|------|-------------|
| `AS OF expr` | row valid at `expr` |
| `BETWEEN a AND b` | inclusive both ends |
| `FROM a TO b` | inclusive start, exclusive end |
| `ALL` | every version recorded |

`AS OF TRANSACTION n` ONLY works on transaction-precision tables.

Session-level implicit AS-OF :

```sql
SET SESSION system_versioning_asof = '2026-01-01 12:00:00';
SELECT * FROM t;                       -- implicit AS OF
SET SESSION system_versioning_asof = DEFAULT;
```

Does NOT apply to `INSERT ... SELECT` or `REPLACE ... SELECT`.

## 3. PARTITION BY SYSTEM_TIME grammar

```
PARTITION BY SYSTEM_TIME
  [ INTERVAL n unit [STARTS 'ts'] | LIMIT n ]
  [ AUTO [PARTITIONS p] ]
  [ ( PARTITION pname HISTORY [, ...] , PARTITION cname CURRENT ) ]
```

Constraints :

- Exactly one CURRENT partition.
- At least one HISTORY partition.
- Transaction-precision tables CANNOT be partitioned with `BY SYSTEM_TIME`.
- `INTERVAL ... AUTO` added in 10.9.1.
- `LIMIT ... AUTO` added in 10.9.1.
- Default partition layout (10.5+) : omit the `(PARTITION ...)` list and MariaDB creates default HISTORY + CURRENT.

Toggle AUTO on an existing table :

```sql
-- enable
ALTER TABLE t PARTITION BY SYSTEM_TIME INTERVAL 1 HOUR AUTO;
-- disable
ALTER TABLE t PARTITION BY SYSTEM_TIME INTERVAL 1 HOUR;
```

Subpartitioning :

```sql
CREATE TABLE t (x INT) WITH SYSTEM VERSIONING
  PARTITION BY SYSTEM_TIME
    SUBPARTITION BY KEY (x) SUBPARTITIONS 4
  ( PARTITION ph HISTORY, PARTITION pc CURRENT );
```

## 4. DELETE HISTORY grammar

```
DELETE HISTORY FROM t
  [ BEFORE SYSTEM_TIME { 'ts' | TRANSACTION n } ];
```

Privilege : `DELETE HISTORY` (granted separately from `DELETE`).

```sql
GRANT DELETE HISTORY ON db.* TO 'audit_admin'@'%';
```

`TRUNCATE TABLE` on a system-versioned table raises error 4137 ; use `DELETE HISTORY` or drop partitions.

## 5. PERIOD FOR : application-time grammar

### 5.1 PERIOD declaration

```
PERIOD FOR period_name (start_col, end_col)
```

Rules :

- Columns MUST be `DATE`, `DATETIME`, or `TIMESTAMP` of equal precision.
- Columns are implicitly `NOT NULL`.
- An invisible CHECK enforces `start_col < end_col`.
- `TIME` and `YEAR` are NOT supported.
- The period_name CANNOT be `SYSTEM_TIME` (reserved for system-versioning).

### 5.2 ALTER TABLE operations

```sql
ALTER TABLE t ADD  PERIOD [IF NOT EXISTS] FOR period_name (start_col, end_col);
ALTER TABLE t DROP PERIOD [IF EXISTS]     FOR period_name;
```

### 5.3 WITHOUT OVERLAPS constraint (10.5.3+)

```sql
ALTER TABLE prices ADD PRIMARY KEY (product_id, validity WITHOUT OVERLAPS);
```

Prevents two rows from sharing a (product_id) + overlapping (validity) tuple.

### 5.4 FOR PORTION OF DML

DELETE :

```sql
DELETE FROM t
  FOR PORTION OF period_name FROM start_expr TO end_expr;
```

UPDATE :

```sql
UPDATE t
  FOR PORTION OF period_name FROM start_expr TO end_expr
  SET col = expr [, ...];
```

Restrictions :

- `FROM` and `TO` expressions MUST be constants.
- Multi-table DELETE/UPDATE NOT supported under `FOR PORTION OF`.
- UPDATE cannot SET the two period columns.
- UPDATE cannot reference period values in the SET expression.
- `FOR PORTION OF system_time` raises a syntax error on bitemporal tables.

Effect of DELETE FOR PORTION OF on each affected row :

| Overlap | Result |
|---------|--------|
| Range covers row | row removed |
| Range partially overlaps | row shrunk to the non-overlap remainder |
| Range strictly inside row | row split into two rows |

## 6. Bitemporal grammar (10.5+)

Combine in one CREATE TABLE :

```sql
CREATE TABLE t (
  x        INT,
  d_start  DATE,
  d_end    DATE,
  rs       TIMESTAMP(6) GENERATED ALWAYS AS ROW START INVISIBLE,
  re       TIMESTAMP(6) GENERATED ALWAYS AS ROW END   INVISIBLE,
  PERIOD FOR application_time (d_start, d_end),
  PERIOD FOR SYSTEM_TIME       (rs, re)
) WITH SYSTEM VERSIONING;
```

Both periods can co-exist. `FOR PORTION OF` is valid on application_time only.

## 7. System variables

| Variable | Scope | Default | Introduced | Purpose |
|----------|-------|---------|------------|---------|
| `system_versioning_alter_history` | Global/Session | `ERROR` | 10.3 | `ERROR` rejects ALTER on versioned tables ; `KEEP` allows it with history-accuracy compromised |
| `system_versioning_asof` | Global/Session | `DEFAULT` | 10.3 | Sets implicit `AS OF` for read queries in the session |
| `system_versioning_insert_history` | Global/Session | `OFF` | 10.11.0 | When `ON`, allows direct INSERT into `row_start` / `row_end` (requires `secure_timestamp` permission) |

## 8. Pseudo-columns

When period columns are implicit (`WITH SYSTEM VERSIONING` only, no explicit declaration) :

- `ROW_START` and `ROW_END` are addressable as pseudo-columns in `SELECT`.
- They are hidden from `SELECT *`.
- They cannot be referenced in `INSERT`, `UPDATE`, or `WHERE` of writes.

## 9. Privileges

| Privilege | Required for |
|-----------|--------------|
| `INSERT`, `UPDATE`, `DELETE` | Standard DML on the table |
| `DELETE HISTORY` | `DELETE HISTORY FROM t [BEFORE SYSTEM_TIME ...]` |
| `ALTER` | Adding/dropping versioning, periods, partitions |

## 10. Version matrix

| Feature | Introduced |
|---------|-----------|
| `WITH SYSTEM VERSIONING` | 10.3 |
| `PERIOD FOR SYSTEM_TIME` explicit columns | 10.3 |
| Transaction-precision (`BIGINT UNSIGNED`) | 10.3 |
| `FOR SYSTEM_TIME AS OF / BETWEEN / FROM TO / ALL` | 10.3 |
| `PARTITION BY SYSTEM_TIME` (manual) | 10.3 |
| `DELETE HISTORY` | 10.3 |
| Column-level `WITHOUT SYSTEM VERSIONING` | 10.3 |
| `system_versioning_alter_history` variable | 10.3 |
| `system_versioning_asof` variable | 10.3 |
| Application-time `PERIOD FOR` | 10.4 |
| `FOR PORTION OF` DML | 10.4 |
| Bitemporal (system-time + application-time) | 10.5 |
| `WITHOUT OVERLAPS` PK/UNIQUE | 10.5.3 |
| Default partition layout (no explicit list) | 10.5 |
| `PARTITION BY SYSTEM_TIME INTERVAL ... AUTO` | 10.9.1 |
| `PARTITION BY SYSTEM_TIME LIMIT ... AUTO` | 10.9.1 |
| `mariadb-dump --dump-history` | 10.11 |
| `system_versioning_insert_history` variable | 10.11 |
| Conversion implicit -> explicit row_start/row_end without rebuild | 11.7 |

## 11. Errors and codes

| Error | Triggered by |
|-------|--------------|
| ER_PARTITION_DEFAULT_ERROR | `PARTITION BY SYSTEM_TIME` on transaction-precision table |
| ER_VERS_NOT_VERSIONED | `FOR SYSTEM_TIME` on a non-versioned table |
| ER_VERS_NOT_ALLOWED | DDL operations forbidden on versioned tables |
| 4137 | `TRUNCATE TABLE` on a versioned table |
| ER_VERS_FIELD_WRONG_TYPE | Wrong type for ROW START / ROW END column |
| ER_VERS_NO_TRX_ID | `AS OF TRANSACTION` on a timestamp-precision table |
| ER_VERS_ALTER_NOT_ALLOWED | ALTER TABLE without `system_versioning_alter_history = KEEP` |

## 12. KB references

- https://mariadb.com/kb/en/system-versioned-tables/
- https://mariadb.com/kb/en/application-time-periods/
- https://mariadb.com/kb/en/bitemporal-tables/
- https://mariadb.com/kb/en/temporal-data-tables/
- https://mariadb.com/kb/en/create-table/
- https://mariadb.com/kb/en/alter-table/
- https://mariadb.com/kb/en/delete/
- https://mariadb.com/kb/en/update/
- https://mariadb.com/kb/en/server-system-variables/
- https://mariadb.com/kb/en/grant/
