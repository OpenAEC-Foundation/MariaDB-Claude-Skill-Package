---
name: mariadb-syntax-window-and-cte
description: >
  Use when writing window functions like ROW_NUMBER, RANK, LAG, LEAD,
  computing running totals, top-N per group, percentile bucketing,
  gap-and-island queries, tree walks with recursive CTEs, hierarchy
  traversal, or migrating PostgreSQL or MySQL window-queries to MariaDB.
  Prevents the common mistake of using the GROUPS frame which is NOT
  supported in MariaDB, forgetting to terminate a recursive CTE, calling
  LAST_VALUE with the default frame and getting the current row instead
  of the partition tail, or assuming a CTE is always materialized.
  Covers window function OVER syntax with PARTITION BY, ORDER BY, and
  ROWS or RANGE frames (NEVER GROUPS), ranking functions (ROW_NUMBER,
  RANK, DENSE_RANK, PERCENT_RANK, CUME_DIST, NTILE), value functions
  (LAG, LEAD, FIRST_VALUE, LAST_VALUE, NTH_VALUE), aggregates used as
  window functions, named windows via the WINDOW clause, non-recursive
  CTEs with WITH AS, recursive CTEs since 10.2.2, the CYCLE col_list
  RESTRICT clause since 10.5.2, and the max_recursive_iterations cap.
  Keywords: window function, OVER, PARTITION BY, ORDER BY, ROW_NUMBER,
  RANK, DENSE_RANK, PERCENT_RANK, CUME_DIST, NTILE, LAG, LEAD,
  FIRST_VALUE, LAST_VALUE, NTH_VALUE, MEDIAN, PERCENTILE_CONT,
  PERCENTILE_DISC, frame, ROWS frame, RANGE frame, GROUPS frame not
  supported, EXCLUDE not supported, CTE, common table expression,
  WITH AS, recursive CTE, WITH RECURSIVE, max_recursive_iterations,
  CYCLE RESTRICT, top N per group, running total, gap and island,
  tree walk, hierarchy, organizational chart, how do I rank rows,
  how do I number rows in a group, my recursive query hangs,
  my LAST_VALUE returns the wrong row, GROUPS frame syntax error,
  unknown frame unit GROUPS
license: MIT
compatibility: "Designed for Claude Code. Requires MariaDB 10.6-LTS, 10.11-LTS, 11.x, 12.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# mariadb-syntax-window-and-cte

Window functions and Common Table Expressions in MariaDB 10.6 LTS, 10.11 LTS, 11.x, and 12.x. All claims verified against `mariadb.com/kb/en/window-functions-overview/`, `window-frames/`, `with/`, and `recursive-common-table-expressions-overview/` on 2026-05-19.

## Quick Reference

- ALWAYS use `ROWS` or `RANGE` for frame units. **NEVER use `GROUPS`** : MariaDB does NOT implement it at any version through 12.x (PostgreSQL and MySQL 8.0+ only).
- ALWAYS write an explicit frame for `LAST_VALUE`, `FIRST_VALUE`, `NTH_VALUE`, `SUM`, `AVG`, `COUNT`, `MIN`, `MAX` when you want partition-wide results. The default frame when `ORDER BY` is present is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` (a running window, NOT the full partition).
- ALWAYS terminate a recursive CTE with a depth guard (`WHERE depth < N`) or `CYCLE col RESTRICT` (10.5.2+). Without termination, `max_recursive_iterations` (default 1000) aborts the query with error 1969.
- ALWAYS write `ORDER BY` inside `OVER()` when using value functions (`LAG`, `LEAD`, `FIRST_VALUE`, `LAST_VALUE`, `NTH_VALUE`). They have no defined behavior without it.
- ALWAYS use `ROW_NUMBER()` (not `RANK`) when you need a unique rank per row. `RANK` skips after ties (1, 1, 3, 4) ; `DENSE_RANK` does not skip (1, 1, 2, 3) ; `ROW_NUMBER` is always unique.
- NEVER assume a non-recursive CTE is materialized. The optimizer merges it into the outer query when `optimizer_switch=derived_merge=on` (the default). Use `EXPLAIN` to confirm : `select_type=DERIVED` means materialized, inline expansion means merged. Recursive CTEs are ALWAYS materialized.
- NEVER use `EXCLUDE CURRENT ROW`, `EXCLUDE GROUP`, `EXCLUDE TIES`, `EXCLUDE NO OTHERS`, or `DISTINCT` inside an aggregate-as-window. None are implemented in MariaDB.
- NEVER use `NULLS FIRST` or `NULLS LAST` in `ORDER BY` inside `OVER()` : not supported. `NULL` sorts first under `ASC`, last under `DESC` (the default).
- ALWAYS check version : recursive CTE needs 10.2.2+, `CYCLE col RESTRICT` needs 10.5.2+, `WITHIN GROUP` for `PERCENTILE_CONT` / `PERCENTILE_DISC` / `MEDIAN` needs 10.3.3+.

## Version Matrix

| Feature | Introduced | Status in 10.6 / 10.11 / 11.x / 12.x |
|---------|------------|--------------------------------------|
| Window functions (ROW_NUMBER, RANK, etc.) | 10.2 | Supported |
| `WINDOW` named-window clause | 10.2 | Supported |
| Aggregates as window functions | 10.2 | Supported |
| Non-recursive CTE (`WITH name AS`) | 10.2.1 | Supported |
| Recursive CTE (`WITH RECURSIVE`) | 10.2.2 | Supported |
| `PERCENTILE_CONT`, `PERCENTILE_DISC`, `MEDIAN` | 10.3.3 | Supported |
| `CYCLE col_list RESTRICT` | 10.5.2 | Supported |
| `GROUPS` frame unit | NEVER | **NOT supported** |
| `EXCLUDE` clause | NEVER | **NOT supported** |
| `NULLS FIRST` / `NULLS LAST` in OVER ORDER BY | NEVER | **NOT supported** |
| `DISTINCT` inside aggregate-window | NEVER | **NOT supported** |

## Decision Tree : Which Ranking Function?

1. **Need a unique integer per row, ties broken arbitrarily?** Use `ROW_NUMBER()`.
2. **Need ties to share the same rank, with subsequent ranks skipped (1, 1, 3)?** Use `RANK()`.
3. **Need ties to share the same rank, but NO skipping (1, 1, 2)?** Use `DENSE_RANK()`.
4. **Need a percentage-position in [0, 1] interval?** Use `PERCENT_RANK()` (returns `(rank - 1) / (total - 1)`).
5. **Need a cumulative distribution in (0, 1] interval?** Use `CUME_DIST()`.
6. **Need to split a partition into N equal-sized buckets?** Use `NTILE(N)`.

## Decision Tree : Window Function vs Aggregate vs CTE

1. **Need to keep all rows AND show an aggregate alongside?** Window function : `SUM(x) OVER (PARTITION BY g)`. NEVER `GROUP BY` (collapses rows).
2. **Need to filter on a window-function result (e.g. top-3 per group)?** Wrap window in a CTE, filter in outer SELECT. Window functions are evaluated AFTER `WHERE`, so they cannot appear in `WHERE`.
3. **Need a hierarchical walk (manager / parent / category tree)?** Recursive CTE with `UNION ALL`. NEVER use `UNION` (deduplicates the recursion, killing performance).
4. **Need readability for a multi-step query?** Non-recursive CTE. Be aware the optimizer may merge it into the outer query (verify with `EXPLAIN`).

## Window Function Syntax

```sql
-- 10.2+ : core window function syntax
SELECT
  emp_id,
  dept_id,
  salary,
  ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rn,
  RANK()       OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rk,
  LAG(salary, 1, 0) OVER (PARTITION BY dept_id ORDER BY hire_date) AS prev_salary
FROM employees;
```

Three optional sub-clauses inside `OVER()` : `PARTITION BY`, `ORDER BY`, and a frame `{ROWS|RANGE} frame_clause`. Without `ORDER BY`, the window is the entire partition. With `ORDER BY` and no explicit frame, the default is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` (a running window).

## Frame Syntax (ROWS or RANGE ONLY)

```sql
-- 10.2+ : explicit frame, full partition
SUM(x) OVER (PARTITION BY g ORDER BY t
             ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)

-- 10.2+ : moving 7-row average
AVG(x) OVER (ORDER BY t ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)

-- 10.2+ : value-based window (RANGE uses ORDER BY value, not row count)
SUM(amount) OVER (ORDER BY order_date
                  RANGE BETWEEN INTERVAL 7 DAY PRECEDING AND CURRENT ROW)
```

Frame boundaries : `UNBOUNDED PRECEDING`, `n PRECEDING`, `CURRENT ROW`, `n FOLLOWING`, `UNBOUNDED FOLLOWING`. **NEVER `GROUPS BETWEEN ...`** : MariaDB rejects this with a syntax error.

## Named Windows

```sql
-- 10.2+ : reuse window spec via WINDOW clause
SELECT
  emp_id,
  dept_id,
  RANK()      OVER w AS dept_rank,
  AVG(salary) OVER w AS dept_avg
FROM employees
WINDOW w AS (PARTITION BY dept_id ORDER BY salary DESC);
```

The `WINDOW` clause sits between `HAVING` and `ORDER BY` in the SELECT statement. Use it whenever two or more `OVER()` references would share the same spec.

## CTE Syntax

```sql
-- 10.2.1+ : non-recursive CTE
WITH recent_orders AS (
  SELECT order_id, customer_id, total
  FROM orders
  WHERE order_date >= CURRENT_DATE - INTERVAL 30 DAY
)
SELECT customer_id, SUM(total) AS spent_30d
FROM recent_orders
GROUP BY customer_id;
```

## Recursive CTE Syntax

```sql
-- 10.2.2+ : recursive CTE with depth guard
WITH RECURSIVE org_tree (emp_id, manager_id, depth, path) AS (
  SELECT emp_id, manager_id, 0, CAST(emp_id AS CHAR(1000))
    FROM employees WHERE manager_id IS NULL  -- anchor : root rows
  UNION ALL
  SELECT e.emp_id, e.manager_id, t.depth + 1,
         CONCAT(t.path, ',', e.emp_id)
    FROM employees e JOIN org_tree t ON e.manager_id = t.emp_id
   WHERE t.depth < 20                        -- terminator
)
SELECT * FROM org_tree;
```

Required structure : anchor `UNION ALL` recursive part. The recursive part must reference the CTE itself exactly once. ALWAYS use `UNION ALL`, never `UNION` (deduplication kills performance and changes semantics).

## CYCLE Clause (10.5.2+)

```sql
-- 10.5.2+ : built-in cycle detection
WITH RECURSIVE g (a, b) AS (
  SELECT a, b FROM edges WHERE a = 1
  UNION
  SELECT e.a, e.b FROM edges e JOIN g ON e.a = g.b
)
CYCLE a RESTRICT
SELECT * FROM g;
```

The `CYCLE` clause lists columns whose value-tuple, if repeated, terminates the recursion for that branch. `RESTRICT` is currently the only supported action.

## max_recursive_iterations

```sql
-- per-session override
SET SESSION max_recursive_iterations = 5000;  -- default 1000

-- check current value
SHOW SESSION VARIABLES LIKE 'max_recursive_iterations';
```

The recursive CTE aborts with error `1969 (ER_STATEMENT_TIMEOUT)` once iterations exceed the variable. ALWAYS prefer an explicit depth guard or `CYCLE col RESTRICT` ; the variable is a safety net, not a design tool.

## Reference Links

- `references/methods.md` : complete API signatures for window functions and CTE grammar
- `references/examples.md` : 10+ working examples (top-N, running total, LAG/LEAD, percentile bucketing, named windows, recursive CTE, CYCLE)
- `references/anti-patterns.md` : 8+ documented anti-patterns and corrections

## Official Sources

- Window functions overview : https://mariadb.com/kb/en/window-functions-overview/
- Window frames : https://mariadb.com/kb/en/window-frames/
- WITH (CTE) : https://mariadb.com/kb/en/with/
- Recursive CTE : https://mariadb.com/kb/en/recursive-common-table-expressions-overview/
- LAG : https://mariadb.com/kb/en/lag/
- LEAD : https://mariadb.com/kb/en/lead/
- ROW_NUMBER : https://mariadb.com/kb/en/row_number/
- RANK : https://mariadb.com/kb/en/rank/
- DENSE_RANK : https://mariadb.com/kb/en/dense_rank/
- NTILE : https://mariadb.com/kb/en/ntile/
- PERCENT_RANK : https://mariadb.com/kb/en/percent_rank/
- CUME_DIST : https://mariadb.com/kb/en/cume_dist/
- FIRST_VALUE : https://mariadb.com/kb/en/first_value/
- LAST_VALUE : https://mariadb.com/kb/en/last_value/
- NTH_VALUE : https://mariadb.com/kb/en/nth_value/
- PERCENTILE_CONT : https://mariadb.com/kb/en/percentile_cont/
- PERCENTILE_DISC : https://mariadb.com/kb/en/percentile_disc/
- MEDIAN : https://mariadb.com/kb/en/median/
