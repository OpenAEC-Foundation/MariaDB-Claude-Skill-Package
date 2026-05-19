# MariaDB SQL DDL : Methods Reference

Complete grammar and reference for every DDL construct covered by `mariadb-syntax-sql-ddl`.

## CREATE TABLE

```sql
CREATE [OR REPLACE] [TEMPORARY] TABLE [IF NOT EXISTS] tbl_name
  (create_definition, ...)
  [table_options]
  [partition_options];

create_definition:
    col_name column_definition
  | [CONSTRAINT [symbol]] PRIMARY KEY [index_type] (index_col_name, ...) [index_option]
  | {INDEX | KEY} [index_name] [index_type] (index_col_name, ...) [index_option]
  | [CONSTRAINT [symbol]] UNIQUE [INDEX | KEY] [index_name] [index_type] (index_col_name, ...) [index_option]
  | {FULLTEXT | SPATIAL} [INDEX | KEY] [index_name] (index_col_name, ...) [index_option]
  | [CONSTRAINT [symbol]] FOREIGN KEY [index_name] (index_col_name, ...)
        REFERENCES tbl_name (index_col_name, ...)
        [ON DELETE reference_option]
        [ON UPDATE reference_option]
  | [CONSTRAINT [symbol]] CHECK (expr)
  ;

column_definition:
    data_type
      [NOT NULL | NULL]
      [DEFAULT default_value]
      [AUTO_INCREMENT]
      [UNIQUE [KEY]]
      [PRIMARY [KEY]]
      [COMMENT 'string']
      [COLLATE collation_name]
      [INVISIBLE]                                   -- 10.3+
      [GENERATED ALWAYS AS (expr) [VIRTUAL|PERSISTENT|STORED]]   -- 10.2+
      [CHECK (expr)]
  ;
```

### Table options

| Option | Meaning |
|--------|---------|
| `ENGINE=` | Storage engine. InnoDB default. Alternatives : Aria, MyISAM, MEMORY, MyRocks, S3, Spider, Archive, CSV. |
| `ROW_FORMAT=` | InnoDB : `DYNAMIC` (default, off-page long columns), `COMPACT`, `REDUNDANT`, `COMPRESSED` (zlib, no INSTANT). |
| `DEFAULT CHARSET=` / `CHARACTER SET=` | Per-table charset. Always set explicitly. |
| `COLLATE=` | Per-table collation. `utf8mb4_uca1400_ai_ci` is the modern default (11.6+). |
| `AUTO_INCREMENT=` | Initial value for auto-increment counter. |
| `COMMENT='...'` | Free-text table comment, surfaced in `INFORMATION_SCHEMA.TABLES.TABLE_COMMENT`. |
| `TRANSACTIONAL={0|1}` | Aria-only : enables transactional behaviour. |
| `PAGE_CHECKSUM={0|1}` | Aria-only : disable to save CPU on trusted media. |
| `STATS_PERSISTENT={DEFAULT|0|1}` | InnoDB : store cardinality stats persistently. |
| `STATS_SAMPLE_PAGES=N` | InnoDB : pages sampled for cardinality. Higher = more accurate, slower ANALYZE. |
| `KEY_BLOCK_SIZE=N` | InnoDB compressed only ; MyISAM/Aria key block size. |

### Reference options for FK

`reference_option` is one of `RESTRICT` (default, blocks delete), `CASCADE` (apply to child rows), `SET NULL` (clear child FK columns, requires nullable), `NO ACTION` (deferred-check synonym for RESTRICT in MariaDB), `SET DEFAULT` (parsed but not enforced by InnoDB).

### Restrictions

- Partitioned tables cannot declare FOREIGN KEY and cannot be referenced by FOREIGN KEY.
- Generated PERSISTENT columns can be FK columns but `ON UPDATE CASCADE/SET NULL` and `ON DELETE SET NULL` are forbidden when the column is generated.
- The PRIMARY KEY must contain every column used in the partitioning expression.

## ALTER TABLE

```sql
ALTER [ONLINE] [IGNORE] TABLE [IF EXISTS] tbl_name
  [WAIT n | NOWAIT]
  alter_specification [, alter_specification ...]
  [ALGORITHM = {DEFAULT | INSTANT | NOCOPY | INPLACE | COPY}]
  [LOCK      = {DEFAULT | NONE     | SHARED | EXCLUSIVE}]
  ;
```

### Alter specifications (most relevant)

| Specification | Notes |
|---------------|-------|
| `ADD [COLUMN] col_name column_definition [FIRST | AFTER col]` | INSTANT 10.3+ at end, 10.4+ any position. |
| `DROP [COLUMN] col_name` | INSTANT 10.4+ unless indexed (then NOCOPY). |
| `MODIFY [COLUMN] col_name column_definition` | Type-widening often INPLACE ; type-change usually COPY. |
| `CHANGE [COLUMN] old_col new_col column_definition` | RENAME COLUMN equivalent ; 10.5.3+ supports RENAME COLUMN as alias. |
| `RENAME COLUMN old_col TO new_col` | 10.5.3+ ; metadata-only. |
| `RENAME [TO|AS] new_name` | 10.5.3+ ; cross-database via qualified name. |
| `ADD [INDEX | KEY] index_name (col_list)` | INPLACE-only ; ADD INDEX cannot be INSTANT. |
| `DROP {INDEX | KEY} index_name` | INPLACE 10.5+. |
| `ALTER INDEX index_name {VISIBLE | IGNORED}` | 10.6+ : invisible-to-optimiser toggle without DROP. |
| `ADD CONSTRAINT [symbol] FOREIGN KEY ...` | INPLACE if `foreign_key_checks=1` and indexes exist. |
| `DROP FOREIGN KEY symbol` | INPLACE, fast. |
| `ADD CONSTRAINT [symbol] CHECK (expr)` | INPLACE 10.2+ ; existing rows validated. |
| `DROP CONSTRAINT symbol` | INPLACE. |
| `ENABLE KEYS` / `DISABLE KEYS` | MyISAM/Aria only. |
| `IMPORT TABLESPACE` / `DISCARD TABLESPACE` | InnoDB transportable-tablespace. |
| `CONVERT TO CHARACTER SET cs [COLLATE col]` | COPY ; rewrites every row. |

### ALGORITHM values

| Value | Behaviour | When |
|-------|-----------|------|
| `DEFAULT` | Server picks lightest viable algorithm. | Avoid in production code ; be explicit. |
| `INSTANT` | Metadata-only ; no row rewrite. | ADD COLUMN (10.3+ end, 10.4+ any position), DROP COLUMN (10.4+ if not in index), ENUM-append, default value change, RENAME COLUMN/INDEX (10.5.3+). |
| `NOCOPY` | No clustered-index rebuild but secondary indexes may rebuild. | DROP COLUMN of indexed column (rewrites only secondaries). |
| `INPLACE` | Storage engine modifies data in-place ; allows DML if `LOCK=NONE`. | ADD INDEX, DROP INDEX, ADD FK, type-widening, ADD CHECK. |
| `COPY` | Build a new table, copy rows, drop old, rename. | Fallback for incompatible changes (PK rebuild, column-type narrowing). LOCK=NONE supported from 11.2+. |

### LOCK values

| Value | Effect |
|-------|--------|
| `DEFAULT` | Server picks weakest allowed. |
| `NONE` | All concurrent DML permitted. Required for online schema change. |
| `SHARED` | Concurrent reads only. |
| `EXCLUSIVE` | No concurrent DML. |

### Error behaviour

If the chosen ALGORITHM cannot satisfy the requested operation, the server raises `ER_ALTER_OPERATION_NOT_SUPPORTED` (1845) before touching data. Same for LOCK level not satisfiable. This is the design intent : fail fast, never silently fall back.

## DROP TABLE

```sql
DROP [TEMPORARY] TABLE [IF EXISTS]
  tbl_name [, tbl_name] ...
  [RESTRICT | CASCADE];
```

- `IF EXISTS` suppresses the no-such-table warning. Useful in idempotent migrations.
- `TEMPORARY` drops only session-temporary tables ; safer when production names overlap with temps.
- `RESTRICT` / `CASCADE` are parsed for SQL standard compatibility but have no effect : InnoDB always RESTRICT-checks.
- Multi-table DROP is crash-safe (10.6+) but not fully atomic ; survivors after crash may be a subset.
- Dropping a table referenced by a FK from another table fails unless `foreign_key_checks=0`.

## RENAME TABLE

```sql
RENAME TABLE
  old_name1 TO new_name1
  [, old_name2 TO new_name2] ...;
```

- All renames within one statement succeed or none do : atomic per statement (InnoDB, MyRocks, MyISAM, Aria).
- Use the three-step pattern for symmetric swap : `t1 TO tmp, t2 TO t1, tmp TO t2`.
- Cross-database rename works (`db1.t TO db2.t`) but triggers (error 1435) and views (error 1450) block the move.
- Requires `DROP`, `CREATE`, `INSERT` privileges on every affected table.

## CREATE SEQUENCE (10.3+)

```sql
CREATE [OR REPLACE] [TEMPORARY] SEQUENCE [IF NOT EXISTS] seq_name
  [ AS { TINYINT | SMALLINT | MEDIUMINT | INT | BIGINT } [SIGNED|UNSIGNED] ]   -- 11.5+
  [ INCREMENT [BY|=] n ]
  [ MINVALUE [=] n | NO MINVALUE | NOMINVALUE ]
  [ MAXVALUE [=] n | NO MAXVALUE | NOMAXVALUE ]
  [ START [WITH|=] n ]
  [ CACHE [=] n | NOCACHE ]
  [ CYCLE | NOCYCLE ]
  [ table_options ]
  ;
```

Defaults : `INCREMENT BY 1`, `START WITH MINVALUE` (positive increment) or `START WITH MAXVALUE` (negative), `CACHE 1000`, `NOCYCLE`.

### Functions

| Function | Returns | Notes |
|----------|---------|-------|
| `NEXT VALUE FOR seq` | Next sequence value, advances the counter. | SQL-standard form. |
| `NEXTVAL(seq)` | Same as above. | Legacy function form, identical semantics. |
| `PREVIOUS VALUE FOR seq` | Last value generated in this session. | Returns NULL if NEXT VALUE never called in session. |
| `LASTVAL(seq)` | Same as above. | Legacy function form. |
| `SETVAL(seq, n [, is_used [, round]])` | Set next-value pointer to `n`. | If `is_used=0` the next NEXT VALUE returns `n`, else `n+INCREMENT`. |

### ALTER SEQUENCE (10.3+)

```sql
ALTER SEQUENCE [IF EXISTS] seq_name
  [ INCREMENT BY n ]
  [ MINVALUE n | NO MINVALUE ]
  [ MAXVALUE n | NO MAXVALUE ]
  [ START WITH n ]
  [ RESTART [ WITH n ] ]
  [ CACHE n | NOCACHE ]
  [ CYCLE | NOCYCLE ]
  ;
```

## DROP SEQUENCE

```sql
DROP [TEMPORARY] SEQUENCE [IF EXISTS] seq_name [, seq_name] ...;
```

## Generated Columns (10.2+)

```sql
col_name data_type [GENERATED ALWAYS] AS (expression) [VIRTUAL | PERSISTENT | STORED]
  [UNIQUE [KEY]] [COMMENT 'string']
```

- `VIRTUAL` : compute on each read, no disk storage. Default if keyword omitted.
- `PERSISTENT` and `STORED` are synonyms : evaluated once on INSERT/UPDATE, stored on disk, included in row-based replication.
- Expression restrictions : no subqueries, no stored functions, no system variables that change at runtime. Server variables (e.g. `@@time_zone`) are allowed but produce non-deterministic results if changed.
- FK : only PERSISTENT columns can be the local side of a FK. `ON UPDATE CASCADE`, `ON UPDATE SET NULL`, `ON DELETE SET NULL` are forbidden when the FK column is generated.
- ALTER restrictions : `MODIFY` or `CHANGE` of a VIRTUAL column requires `ALGORITHM=COPY`. Adding an indexed VIRTUAL column requires `ALGORITHM=COPY`. Cannot convert an existing regular column into a VIRTUAL generated column in place.
- Indexing : both VIRTUAL and PERSISTENT can be indexed. Non-deterministic functions allowed only on un-indexed VIRTUAL columns.
- Cannot be PRIMARY KEY.

## Invisible Columns (10.3+)

```sql
col_name data_type [NOT NULL] [DEFAULT default_value] INVISIBLE
```

- `SELECT *` skips invisible columns ; `SELECT col_name` returns them.
- `NOT NULL INVISIBLE` requires a `DEFAULT` value.
- At least one column must remain visible per table.
- `INSERT INTO t VALUES (...)` without an explicit column list does not set the invisible column ; supply a column list to set them.
- Modifying visibility is INSTANT-eligible : `ALTER TABLE t MODIFY col INT INVISIBLE, ALGORITHM=INSTANT`.

## Partitioning

```sql
PARTITION BY
    { [LINEAR] HASH (expr)
    | [LINEAR] KEY [ALGORITHM={1|2}] (column_list)
    | RANGE { (expr) | COLUMNS (column_list) }
    | LIST  { (expr) | COLUMNS (column_list) }
    | SYSTEM_TIME [INTERVAL expr time_unit] }
  [PARTITIONS n]
  [SUBPARTITION BY ... [SUBPARTITIONS n]]
  [(partition_definition, ...)]
  ;

partition_definition:
  PARTITION part_name
    [VALUES { LESS THAN ( {expr | MAXVALUE} ) | IN (value_list) | CURRENT | HISTORY }]
    [[STORAGE] ENGINE = engine_name]
    [COMMENT 'string']
    [DATA DIRECTORY = 'path']
    [INDEX DIRECTORY = 'path']
    [MAX_ROWS = n]
    [MIN_ROWS = n]
    [TABLESPACE = ts_name]
    [(subpartition_definition, ...)]
  ;
```

### Types

| Type | Use |
|------|-----|
| `RANGE (expr)` | Partition by a single integer expression. |
| `RANGE COLUMNS (col1, col2, ...)` | Multi-column range, supports dates and strings. |
| `LIST (expr)` | Discrete value list per partition. |
| `LIST COLUMNS (col1, col2, ...)` | Multi-column list, supports non-integer types. |
| `HASH (expr)` | Even distribution by hash of expression. |
| `LINEAR HASH (expr)` | Hash with linear-power-of-two partition addition (faster to add partitions). |
| `KEY (col_list)` | Hash by server-internal hash of named columns. |
| `LINEAR KEY (col_list)` | Linear variant of KEY. |
| `SYSTEM_TIME` | Auto time-period partitions on `WITH SYSTEM VERSIONING` tables ; MariaDB-only. |

### Pruning rules

- Direct comparison on the partitioning column (`WHERE created_at < '2025-01-01'`) prunes.
- Wrapping the column in a function (`WHERE YEAR(created_at) = 2024`) defeats pruning.
- `EXPLAIN PARTITIONS SELECT ...` shows accessed partitions.
- Manual override : `SELECT ... FROM tbl PARTITION (p2024) WHERE ...`.

### Restrictions

- No FOREIGN KEY in or referencing a partitioned table.
- PK and any UNIQUE index must contain every column used in the partition expression.
- Maximum 8192 partitions (incl. subpartitions) total per table.

## Atomic DDL (10.6+)

Fully atomic on server crash : `CREATE TABLE` (excluding `CREATE OR REPLACE`), single-table `DROP TABLE`, `ALTER TABLE`, `RENAME TABLE`, `CREATE VIEW`, `CREATE SEQUENCE`, `CREATE TRIGGER`, `DROP TRIGGER`.

Crash-safe but not fully atomic : multi-table `DROP TABLE`, `CREATE OR REPLACE TABLE`, `DROP DATABASE`.

Mechanism : `ddl_recovery.log` records every in-progress DDL. On restart MariaDB walks the log, completes or reverses each entry, retries up to 3 times.

Storage engines supported : InnoDB, MyRocks, MyISAM, Aria. Not yet fully supported : S3, partitioning engine for some operations.

## Sources

- `mariadb.com/kb/en/alter-table/` (verified 2026-05-19)
- `mariadb.com/kb/en/instant-add-column-for-innodb/`
- `mariadb.com/kb/en/create-sequence/`
- `mariadb.com/kb/en/generated-columns/`
- `mariadb.com/kb/en/invisible-columns/`
- `mariadb.com/kb/en/partitioning-types-overview/`
- `mariadb.com/kb/en/partition-pruning-and-selection/`
- `mariadb.com/kb/en/rename-table/`
- `mariadb.com/kb/en/atomic-ddl/`
