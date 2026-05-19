# Methods : Triggers, Events, Views

Complete syntax reference for MariaDB triggers, events, and views. All grammars verified 2026-05-19 against the MariaDB Knowledge Base.

---

## 1. Triggers

### 1.1 CREATE TRIGGER

```sql
CREATE [OR REPLACE]
    [DEFINER = { user | CURRENT_USER | role }]
    [IF NOT EXISTS]
    TRIGGER trigger_name
    trigger_time trigger_event
    ON tbl_name
    FOR EACH ROW
    [{ FOLLOWS | PRECEDES } other_trigger_name]
    trigger_body
```

- `trigger_time` : `BEFORE` or `AFTER`.
- `trigger_event` : `INSERT`, `UPDATE`, or `DELETE`. **Only one event per trigger.** Combine via separate triggers if you need both.
- `FOR EACH ROW` is mandatory and the only supported granularity. MariaDB does NOT implement statement-level triggers (`FOR EACH STATEMENT` is parsed but ignored on the server).
- `FOLLOWS` / `PRECEDES` order is observable in `INFORMATION_SCHEMA.TRIGGERS.ACTION_ORDER` (10.2.3+).
- `OR REPLACE` and `IF NOT EXISTS` are mutually exclusive ; use one or the other.
- `DEFINER` defaults to `CURRENT_USER` at creation time. The trigger runs with the definer's privileges, NOT the caller's.

### 1.2 Pseudo-records

| Trigger time | Trigger event | NEW available | OLD available | Writable in body |
|--------------|---------------|---------------|---------------|------------------|
| BEFORE       | INSERT        | yes (writable) | no            | `SET NEW.col = expr` allowed |
| AFTER        | INSERT        | yes (read)    | no            | none |
| BEFORE       | UPDATE        | yes (writable) | yes (read)   | `SET NEW.col = expr` allowed |
| AFTER        | UPDATE        | yes (read)    | yes (read)    | none |
| BEFORE       | DELETE        | no            | yes (read)    | none |
| AFTER        | DELETE        | no            | yes (read)    | none |

### 1.3 Forbidden statements inside trigger body

Per KB `trigger-limitations/` :

- `COMMIT`, `ROLLBACK`, `SAVEPOINT`, `RELEASE SAVEPOINT`, `START TRANSACTION` : all raise `Explicit or implicit commit is not allowed`.
- `RETURN` : invalid in trigger context. Use `LEAVE label_name` to exit early.
- A bare `SELECT` that returns a result set to the client. Use `SELECT ... INTO var` for variable assignment or `INSERT ... SELECT` for side-effects.
- DDL on the table the trigger is defined on.
- Triggers on tables in `mysql`, `information_schema`, or `performance_schema`.

### 1.4 DROP TRIGGER

```sql
DROP TRIGGER [IF EXISTS] [schema_name.]trigger_name
```

### 1.5 SHOW TRIGGERS

```sql
SHOW TRIGGERS [FROM schema_name] [LIKE 'pattern' | WHERE expr]
```

Columns : `Trigger`, `Event`, `Table`, `Statement`, `Timing`, `Created`, `sql_mode`, `Definer`, `character_set_client`, `collation_connection`, `Database Collation`.

### 1.6 INFORMATION_SCHEMA.TRIGGERS

Authoritative metadata source. Key columns :

| Column                 | Meaning |
|------------------------|---------|
| TRIGGER_SCHEMA         | schema name |
| TRIGGER_NAME           | trigger name |
| EVENT_MANIPULATION     | INSERT / UPDATE / DELETE |
| EVENT_OBJECT_TABLE     | target table |
| ACTION_ORDER           | numeric firing order within (table, timing, event) tuple |
| ACTION_STATEMENT       | trigger body |
| ACTION_TIMING          | BEFORE / AFTER |
| DEFINER                | definer account |
| SQL_MODE               | sql_mode at CREATE time |

---

## 2. Events

### 2.1 CREATE EVENT

```sql
CREATE [OR REPLACE]
    [DEFINER = { user | CURRENT_USER | role }]
    [IF NOT EXISTS]
    EVENT event_name
    ON SCHEDULE schedule
    [ON COMPLETION [NOT] PRESERVE]
    [ENABLE | DISABLE | DISABLE ON SLAVE]
    [COMMENT 'string']
    DO event_body
```

### 2.2 schedule grammar

```
schedule :
    AT timestamp [+ INTERVAL interval] ...
  | EVERY interval
    [STARTS timestamp [+ INTERVAL interval] ...]
    [ENDS   timestamp [+ INTERVAL interval] ...]

interval :
    quantity {YEAR | QUARTER | MONTH | DAY | HOUR | MINUTE |
              WEEK | SECOND | YEAR_MONTH | DAY_HOUR | DAY_MINUTE |
              DAY_SECOND | HOUR_MINUTE | HOUR_SECOND | MINUTE_SECOND}
```

- `AT timestamp` : one-shot. Combined with `ON COMPLETION NOT PRESERVE` (default), the event auto-drops after firing.
- `EVERY interval` : recurring. `STARTS` defaults to `CURRENT_TIMESTAMP`. `ENDS` is optional.

### 2.3 ON COMPLETION

- `NOT PRESERVE` (default) : drop the event after its last execution (one-shot AT) or after the ENDS time (recurring with ENDS).
- `PRESERVE` : keep the event row in `INFORMATION_SCHEMA.EVENTS` after completion. Use this for one-shot events you want to inspect post-hoc.

### 2.4 State

- `ENABLE` (default) : scheduler considers it for firing.
- `DISABLE` : scheduler skips it. Body is preserved.
- `DISABLE ON SLAVE` : when this server runs as a replica with `read_only` and the binlog of the primary replicates the event's side-effects, this prevents double-execution.

### 2.5 ALTER EVENT and DROP EVENT

```sql
ALTER [DEFINER = { user | CURRENT_USER | role }]
      EVENT event_name
      [ON SCHEDULE schedule]
      [ON COMPLETION [NOT] PRESERVE]
      [RENAME TO new_event_name]
      [ENABLE | DISABLE | DISABLE ON SLAVE]
      [COMMENT 'string']
      [DO event_body]

DROP EVENT [IF EXISTS] event_name
```

### 2.6 event_scheduler system variable

```sql
SHOW VARIABLES LIKE 'event_scheduler';
SET GLOBAL event_scheduler = ON;
```

Values :

| Value     | Behaviour |
|-----------|-----------|
| OFF       | Default on many distros. Scheduler thread does NOT run. `SET GLOBAL event_scheduler = ON` activates it without restart. |
| ON        | Scheduler thread is active. Events fire per their schedule. |
| DISABLED  | Scheduler thread is disabled and cannot be turned on at runtime. The server was started with `--event-scheduler=DISABLED`. A restart with `event_scheduler=ON` in `[mysqld]` is required. |

Persist `event_scheduler = ON` in the `[mysqld]` section of `my.cnf` for it to survive restart. `SET GLOBAL` alone does not survive restart.

### 2.7 INFORMATION_SCHEMA.EVENTS

Key columns for monitoring :

| Column              | Meaning |
|---------------------|---------|
| EVENT_NAME          | event name |
| EVENT_DEFINITION    | DO body |
| EVENT_TYPE          | ONE TIME / RECURRING |
| EXECUTE_AT          | for one-shot |
| INTERVAL_VALUE      | for recurring |
| INTERVAL_FIELD      | unit : DAY, HOUR, etc. |
| STATUS              | ENABLED / DISABLED / SLAVESIDE_DISABLED |
| ON_COMPLETION       | PRESERVE / NOT PRESERVE |
| LAST_EXECUTED       | timestamp of last successful run, NULL if never |
| STARTS              | recurring window start |
| ENDS                | recurring window end |
| DEFINER             | account |

### 2.8 SHOW EVENTS

```sql
SHOW EVENTS [FROM schema_name] [LIKE 'pattern' | WHERE expr]
```

---

## 3. Views

### 3.1 CREATE VIEW

```sql
CREATE [OR REPLACE]
    [ALGORITHM = { UNDEFINED | MERGE | TEMPTABLE }]
    [DEFINER = { user | CURRENT_USER | role }]
    [SQL SECURITY { DEFINER | INVOKER }]
    VIEW view_name [(column_list)]
    AS select_statement
    [WITH [CASCADED | LOCAL] CHECK OPTION]
```

### 3.2 ALGORITHM

| Value     | Behaviour |
|-----------|-----------|
| UNDEFINED | Default. Optimizer chooses MERGE when the view is mergeable, TEMPTABLE otherwise. |
| MERGE     | View body is inlined into the outer query at parse time. Preserves index pushdown. Required for the view to be updatable. |
| TEMPTABLE | View result is materialized into a per-query temp table at every reference. Breaks index pushdown. The view is NEVER updatable. |

Mergeable constructs are anything WITHOUT : `DISTINCT`, aggregate functions, `GROUP BY`, `HAVING`, `LIMIT`, `UNION`, subqueries in the SELECT list, assignment to user variables, or refers to no tables.

### 3.3 SQL SECURITY

| Value     | Behaviour |
|-----------|-----------|
| DEFINER   | Default. View runs with the DEFINER account's privileges. Standard pattern for exposing a subset of a sensitive table. |
| INVOKER   | View runs with the calling user's privileges. The caller must have SELECT (and INSERT / UPDATE / DELETE for DML) on the base tables. |

### 3.4 WITH CHECK OPTION

Only meaningful on updatable views. Rejects an `INSERT` or `UPDATE` through the view that would produce a row outside the view's `WHERE` predicate.

- `LOCAL` : check only the immediate view.
- `CASCADED` (default) : walk every view in the chain.

### 3.5 Updatability rules

A view is updatable only when ALL of these hold :

- Exactly one base table referenced (no joins of more than one base table for INSERT, joins are allowed for UPDATE / DELETE if updating columns belong to a single base table).
- No `DISTINCT`.
- No aggregate functions (`SUM`, `COUNT`, `MIN`, `MAX`, `AVG`, etc.).
- No `GROUP BY` or `HAVING`.
- No `UNION` or `UNION ALL`.
- No subqueries in the SELECT list that reference the same base table.
- `ALGORITHM` is `MERGE` (or `UNDEFINED` and the optimizer picks MERGE).
- All NOT-NULL columns of the base table without a DEFAULT are present in the view's column list (for INSERT through the view).

### 3.6 ALTER VIEW and DROP VIEW

```sql
ALTER [ALGORITHM = { UNDEFINED | MERGE | TEMPTABLE }]
      [DEFINER = { user | CURRENT_USER | role }]
      [SQL SECURITY { DEFINER | INVOKER }]
      VIEW view_name [(column_list)]
      AS select_statement
      [WITH [CASCADED | LOCAL] CHECK OPTION]

DROP VIEW [IF EXISTS] view_name [, view_name] ... [RESTRICT | CASCADE]
```

`RESTRICT` and `CASCADE` are parsed for SQL standard compliance but have no effect in MariaDB.

### 3.7 SHOW CREATE VIEW

```sql
SHOW CREATE VIEW view_name
```

Returns the canonical `CREATE VIEW` statement including resolved DEFINER and SQL SECURITY clauses.

### 3.8 INFORMATION_SCHEMA.VIEWS

Key columns :

| Column                | Meaning |
|-----------------------|---------|
| TABLE_NAME            | view name (views appear in TABLES too with TABLE_TYPE='VIEW') |
| VIEW_DEFINITION       | the SELECT body |
| CHECK_OPTION          | NONE / LOCAL / CASCADED |
| IS_UPDATABLE          | YES / NO |
| DEFINER               | account |
| SECURITY_TYPE         | DEFINER / INVOKER |
| ALGORITHM             | UNDEFINED / MERGE / TEMPTABLE |

---

## 4. Materialized-view workaround

MariaDB does NOT support `CREATE MATERIALIZED VIEW`. The supported pattern :

```sql
CREATE OR REPLACE TABLE snapshot_name AS SELECT ... ;
CREATE INDEX ix_snapshot ON snapshot_name (...) ;

CREATE OR REPLACE EVENT ev_refresh_snapshot
  ON SCHEDULE EVERY n unit
  ON COMPLETION PRESERVE
  DISABLE ON SLAVE
DO
  BEGIN
    TRUNCATE TABLE snapshot_name;
    INSERT INTO snapshot_name SELECT ... ;
  END;
```

Trade-offs :

- Eventual consistency : the snapshot lags by up to one interval.
- Refresh cost : the full SELECT runs on every refresh ; consider delta-style refresh for large data sets.
- Concurrency : queries against the snapshot during TRUNCATE see an empty table for the refresh window. Wrap the refresh in a `RENAME TABLE` swap if zero-downtime is required.

---

## 5. CREATE OR REPLACE semantics

`CREATE OR REPLACE TRIGGER`, `EVENT`, `VIEW` are atomic : the engine drops the existing object and creates the new one in the same statement. If creation fails, the old object remains in place. Verified : KB `create-or-replace/`. Prefer this idiom over `DROP IF EXISTS` + `CREATE` for migration scripts.
