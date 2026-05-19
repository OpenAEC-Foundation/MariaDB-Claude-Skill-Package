# Methods : Window Functions and CTEs in MariaDB

Complete API surface for window functions, frame grammar, and CTE syntax. All signatures verified against the MariaDB Knowledge Base on 2026-05-19.

---

## Window Function Skeleton

```
window_function([arguments]) OVER ( [window_spec] | window_name )

window_spec ::=
  [PARTITION BY expr [, expr ...]]
  [ORDER BY expr [ASC | DESC] [, expr [ASC | DESC] ...]]
  [frame_clause]

frame_clause ::=
  {ROWS | RANGE} frame_extent

frame_extent ::=
    frame_border
  | BETWEEN frame_border AND frame_border

frame_border ::=
    UNBOUNDED PRECEDING
  | expr PRECEDING
  | CURRENT ROW
  | expr FOLLOWING
  | UNBOUNDED FOLLOWING
```

NEVER use `GROUPS` as a frame unit : MariaDB does not implement it (KB `window-frames`). NEVER use `EXCLUDE CURRENT ROW`, `EXCLUDE GROUP`, `EXCLUDE TIES`, `EXCLUDE NO OTHERS` : not implemented. NEVER use `NULLS FIRST` / `NULLS LAST` inside `OVER (... ORDER BY ...)` : not supported.

Default frame when `ORDER BY` is specified without an explicit frame : `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`. Without `ORDER BY` the window is the entire partition.

KB : https://mariadb.com/kb/en/window-frames/

---

## Named Window Clause

```sql
-- 10.2+
SELECT ...
FROM ...
WHERE ...
GROUP BY ...
HAVING ...
WINDOW w AS (window_spec) [, w2 AS (window_spec) ...]
ORDER BY ...
```

Named windows are referenced in `OVER w` instead of repeating the spec.

KB : https://mariadb.com/kb/en/window-functions-overview/

---

## Ranking Functions

All ranking functions REQUIRE `ORDER BY` inside `OVER()`. Frame clauses are IGNORED by ranking functions even if you write one.

### ROW_NUMBER

```
ROW_NUMBER() OVER ( [PARTITION BY ...] ORDER BY ... )
```

- Introduced : 10.2
- Return type : BIGINT UNSIGNED
- Returns : sequential integer per row inside partition, ties broken arbitrarily, always unique
- KB : https://mariadb.com/kb/en/row_number/

### RANK

```
RANK() OVER ( [PARTITION BY ...] ORDER BY ... )
```

- Introduced : 10.2
- Return type : BIGINT UNSIGNED
- Returns : tied rows share rank, subsequent ranks SKIP (1, 1, 3, 4, 4, 6)
- KB : https://mariadb.com/kb/en/rank/

### DENSE_RANK

```
DENSE_RANK() OVER ( [PARTITION BY ...] ORDER BY ... )
```

- Introduced : 10.2
- Return type : BIGINT UNSIGNED
- Returns : tied rows share rank, subsequent ranks DO NOT skip (1, 1, 2, 3, 3, 4)
- KB : https://mariadb.com/kb/en/dense_rank/

### PERCENT_RANK

```
PERCENT_RANK() OVER ( [PARTITION BY ...] ORDER BY ... )
```

- Introduced : 10.2
- Return type : DOUBLE
- Formula : `(RANK() - 1) / (total_rows_in_partition - 1)`
- Range : [0, 1]
- KB : https://mariadb.com/kb/en/percent_rank/

### CUME_DIST

```
CUME_DIST() OVER ( [PARTITION BY ...] ORDER BY ... )
```

- Introduced : 10.2
- Return type : DOUBLE
- Formula : `rank_including_ties / total_rows_in_partition`
- Range : (0, 1]
- KB : https://mariadb.com/kb/en/cume_dist/

### NTILE

```
NTILE(n) OVER ( [PARTITION BY ...] ORDER BY ... )
```

- Introduced : 10.2
- Return type : BIGINT
- `n` : positive integer literal or expression resolving to one
- Returns : bucket index in `[1, n]`. If partition has fewer rows than `n`, only the first `partition_rows` buckets are populated.
- KB : https://mariadb.com/kb/en/ntile/

---

## Value Functions

All value functions REQUIRE `ORDER BY` inside `OVER()`. `LAG`, `LEAD`, `FIRST_VALUE`, `NTH_VALUE` IGNORE the frame clause. `LAST_VALUE` HONOURS the frame clause (this is the LAST_VALUE footgun).

### LAG

```
LAG(expr [, offset [, default]]) OVER ( [PARTITION BY ...] ORDER BY ... )
```

- Introduced : 10.2
- `offset` : non-negative integer, default 1
- `default` : value returned when offset goes past the partition start, default NULL
- Return type : type of `expr`
- KB : https://mariadb.com/kb/en/lag/

### LEAD

```
LEAD(expr [, offset [, default]]) OVER ( [PARTITION BY ...] ORDER BY ... )
```

- Introduced : 10.2
- Symmetric to LAG : looks `offset` rows AHEAD of the current row
- KB : https://mariadb.com/kb/en/lead/

### FIRST_VALUE

```
FIRST_VALUE(expr) OVER ( [PARTITION BY ...] ORDER BY ... [frame_clause] )
```

- Introduced : 10.2
- Returns : value of `expr` in the FIRST row of the frame (NOT partition, unless frame is `UNBOUNDED PRECEDING`)
- KB : https://mariadb.com/kb/en/first_value/

### LAST_VALUE

```
LAST_VALUE(expr) OVER ( [PARTITION BY ...] ORDER BY ... [frame_clause] )
```

- Introduced : 10.2
- Returns : value of `expr` in the LAST row of the frame
- **Footgun** : with default frame `RANGE UNBOUNDED PRECEDING TO CURRENT ROW`, `LAST_VALUE` returns the CURRENT row. To get the partition tail, ALWAYS write `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`.
- KB : https://mariadb.com/kb/en/last_value/

### NTH_VALUE

```
NTH_VALUE(expr, n) OVER ( [PARTITION BY ...] ORDER BY ... [frame_clause] )
```

- Introduced : 10.2
- `n` : positive integer, 1-based
- Returns : value of `expr` in the n-th row of the frame, or NULL if frame has fewer than `n` rows
- KB : https://mariadb.com/kb/en/nth_value/

---

## Aggregates as Window Functions

`SUM`, `AVG`, `COUNT`, `MIN`, `MAX`, `BIT_AND`, `BIT_OR`, `BIT_XOR`, `STD`, `STDDEV`, `STDDEV_POP`, `STDDEV_SAMP`, `VAR_POP`, `VAR_SAMP`, `VARIANCE`, `JSON_ARRAYAGG`, `JSON_OBJECTAGG`.

```
aggregate_function(expr) OVER ( [PARTITION BY ...] [ORDER BY ...] [frame_clause] )
```

NEVER use `DISTINCT` inside an aggregate-window in MariaDB (not supported). NEVER use `GROUP_CONCAT` as a window function : it requires `GROUP BY` semantics and is not implemented as a window-aggregate.

---

## Statistical Window Functions

### MEDIAN

```
MEDIAN(expr) OVER ( [PARTITION BY ...] )
```

- Introduced : 10.3.3
- Equivalent to `PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY expr) OVER (...)`
- KB : https://mariadb.com/kb/en/median/

### PERCENTILE_CONT

```
PERCENTILE_CONT(p) WITHIN GROUP (ORDER BY expr) OVER ( [PARTITION BY ...] )
```

- Introduced : 10.3.3
- `p` : DOUBLE in `[0, 1]` (percentile fraction)
- Continuous interpolation between values
- KB : https://mariadb.com/kb/en/percentile_cont/

### PERCENTILE_DISC

```
PERCENTILE_DISC(p) WITHIN GROUP (ORDER BY expr) OVER ( [PARTITION BY ...] )
```

- Introduced : 10.3.3
- `p` : DOUBLE in `[0, 1]`
- Returns actual existing value (no interpolation)
- KB : https://mariadb.com/kb/en/percentile_disc/

---

## CTE Grammar (Non-Recursive)

```
WITH cte_name [(column_list)] AS (subquery)
  [, cte_name [(column_list)] AS (subquery) ...]
{SELECT | INSERT | UPDATE | DELETE | REPLACE} ...
```

- Introduced : 10.2.1
- `column_list` : optional, renames columns produced by the subquery
- Multiple CTEs in one `WITH` separated by commas
- CTE may be referenced multiple times in the outer statement (single evaluation per reference, NOT memoized across refs unless materialized)
- Materialization : optimizer decides via `optimizer_switch=derived_merge=on|off` (default `on` since 10.2)
- KB : https://mariadb.com/kb/en/with/

---

## Recursive CTE Grammar

```
WITH RECURSIVE cte_name [(column_list)] AS (
  anchor_query
  UNION ALL
  recursive_query  -- must reference cte_name exactly once
)
[CYCLE column_list RESTRICT]  -- 10.5.2+
{SELECT | INSERT | UPDATE | DELETE | REPLACE} ...
```

- Introduced : 10.2.2
- `anchor_query` : must NOT reference `cte_name`
- `recursive_query` : MUST reference `cte_name` exactly once, with NO aggregate, GROUP BY, HAVING, DISTINCT, or window function on the CTE reference itself
- Connector : ALWAYS `UNION ALL`. `UNION` (deduplicating) is allowed but slow ; never use it for performance-sensitive recursion
- `CYCLE col RESTRICT` (10.5.2+) : terminate a branch when listed columns repeat
- Recursion cap : `max_recursive_iterations` (session variable, default 1000)
- Always materialized
- KB : https://mariadb.com/kb/en/recursive-common-table-expressions-overview/

---

## max_recursive_iterations System Variable

| Attribute | Value |
|-----------|-------|
| Scope | SESSION, GLOBAL |
| Type | INT UNSIGNED |
| Default | 1000 |
| Min | 0 |
| Max | 4294967295 |
| Dynamic | Yes |

```sql
SET SESSION max_recursive_iterations = 5000;
SET GLOBAL  max_recursive_iterations = 5000;
SHOW VARIABLES LIKE 'max_recursive_iterations';
```

Exceeding the cap raises an error and aborts the recursive CTE. KB : https://mariadb.com/kb/en/server-system-variables/

---

## CTE Materialization Control

```sql
-- inspect
SHOW VARIABLES LIKE 'optimizer_switch';

-- force merge of non-recursive CTE into outer query
SET SESSION optimizer_switch = 'derived_merge=on';

-- force materialization (e.g. for self-referenced CTE used many times)
SET SESSION optimizer_switch = 'derived_merge=off';
```

Verify with `EXPLAIN` :

- `select_type=DERIVED` : CTE materialized into a temp table
- `select_type=SUBQUERY` or inline expansion : CTE merged into outer query

Recursive CTEs are ALWAYS materialized regardless of `derived_merge`. KB : https://mariadb.com/kb/en/optimizer-switch/

---

## Statement Coverage

CTEs work in front of any DML or SELECT :

- `WITH cte AS (...) SELECT ...`
- `WITH cte AS (...) INSERT INTO t SELECT ... FROM cte`
- `WITH cte AS (...) UPDATE t JOIN cte ON ... SET ...`
- `WITH cte AS (...) DELETE t FROM t JOIN cte ON ...`
- `WITH cte AS (...) REPLACE INTO t SELECT ... FROM cte`

Window functions work in `SELECT` and `ORDER BY` clauses only ; NEVER in `WHERE`, `GROUP BY`, or `HAVING` (the SQL evaluation order forbids it). To filter on a window-function result, wrap the query in a CTE.
