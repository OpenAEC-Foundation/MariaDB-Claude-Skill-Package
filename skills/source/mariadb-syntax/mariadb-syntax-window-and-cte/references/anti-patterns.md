# Anti-Patterns : Window Functions and CTEs in MariaDB

Ten anti-patterns with concrete failure mode and corrected version. Drawn from MariaDB KB documentation and field-reported issues (LESSONS L-002).

---

## Anti-Pattern 1 : Using GROUPS as Frame Unit

**Mistake** :

```sql
-- WRONG : GROUPS is NOT supported in MariaDB
SELECT
  emp_id,
  AVG(salary) OVER (ORDER BY hire_date GROUPS BETWEEN 2 PRECEDING AND CURRENT ROW)
FROM employees;
```

**Why it fails** : MariaDB supports `ROWS` and `RANGE` only. `GROUPS` is a PostgreSQL feature and a MySQL 8.0+ feature, not implemented in MariaDB at any version through 12.x. Error : `ERROR 1064 (42000) : You have an error in your SQL syntax ... near 'GROUPS'`.

**Fix** : choose `ROWS` (row-count window) or `RANGE` (value-distance window) based on intent. If you need peer-group semantics, manually emulate with a CTE that pre-aggregates per ORDER-BY-key.

```sql
-- CORRECT : ROWS-based window
SELECT
  emp_id,
  AVG(salary) OVER (ORDER BY hire_date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)
FROM employees;
```

KB : https://mariadb.com/kb/en/window-frames/ (L-002).

---

## Anti-Pattern 2 : LAST_VALUE With Default Frame

**Mistake** :

```sql
-- WRONG : returns the current row, NOT the partition tail
SELECT
  customer_id,
  order_date,
  LAST_VALUE(order_date) OVER (PARTITION BY customer_id ORDER BY order_date) AS latest
FROM orders;
```

**Why it fails** : default frame when `ORDER BY` is present is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`. `LAST_VALUE` returns the last row of THE FRAME, which is the current row, not the partition tail. The column `latest` ends up equal to `order_date`, completely useless.

**Fix** : always write the explicit full-partition frame.

```sql
-- CORRECT : explicit full-partition frame
SELECT
  customer_id,
  order_date,
  LAST_VALUE(order_date) OVER (
    PARTITION BY customer_id
    ORDER BY order_date
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
  ) AS latest
FROM orders;
```

Alternative : use `MAX(order_date) OVER (PARTITION BY customer_id)`. KB : https://mariadb.com/kb/en/last_value/.

---

## Anti-Pattern 3 : Recursive CTE Without Termination

**Mistake** :

```sql
-- WRONG : no terminator, no CYCLE clause
WITH RECURSIVE org_tree (emp_id, manager_id) AS (
  SELECT emp_id, manager_id FROM employees WHERE manager_id IS NULL
  UNION ALL
  SELECT e.emp_id, e.manager_id
    FROM employees e JOIN org_tree t ON e.manager_id = t.emp_id
)
SELECT * FROM org_tree;
```

**Why it fails** : if `employees` accidentally contains a cycle (A manages B, B manages C, C manages A), recursion grows without bound until `max_recursive_iterations` (default 1000) aborts with `ERROR 1969`. The query has already consumed minutes of CPU and memory by the time it errors.

**Fix** : always add a depth guard OR a `CYCLE col RESTRICT` clause (10.5.2+).

```sql
-- CORRECT : depth guard (works on all versions 10.2.2+)
WITH RECURSIVE org_tree (emp_id, manager_id, depth) AS (
  SELECT emp_id, manager_id, 0 FROM employees WHERE manager_id IS NULL
  UNION ALL
  SELECT e.emp_id, e.manager_id, t.depth + 1
    FROM employees e JOIN org_tree t ON e.manager_id = t.emp_id
   WHERE t.depth < 20
)
SELECT * FROM org_tree;

-- CORRECT (10.5.2+) : built-in cycle detection
WITH RECURSIVE org_tree (emp_id, manager_id) AS (
  SELECT emp_id, manager_id FROM employees WHERE manager_id IS NULL
  UNION ALL
  SELECT e.emp_id, e.manager_id
    FROM employees e JOIN org_tree t ON e.manager_id = t.emp_id
)
CYCLE emp_id RESTRICT
SELECT * FROM org_tree;
```

KB : https://mariadb.com/kb/en/recursive-common-table-expressions-overview/.

---

## Anti-Pattern 4 : ROW_NUMBER Without ORDER BY

**Mistake** :

```sql
-- WRONG : non-deterministic ordering
SELECT
  emp_id,
  ROW_NUMBER() OVER (PARTITION BY dept_id) AS rn
FROM employees;
```

**Why it fails** : without `ORDER BY` inside `OVER()`, the ordering is undefined. The same query on the same data may produce different `rn` values across runs. Reports built on this are silently wrong.

**Fix** : ALWAYS specify `ORDER BY` for ranking and value window functions. If no semantic ordering exists, use the primary key as a stable tie-breaker.

```sql
-- CORRECT
SELECT
  emp_id,
  ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY salary DESC, emp_id) AS rn
FROM employees;
```

KB : https://mariadb.com/kb/en/row_number/.

---

## Anti-Pattern 5 : NTILE on a Small Partition

**Mistake** :

```sql
-- WRONG : assuming all 10 buckets populated on every partition
SELECT
  customer_id,
  segment,
  NTILE(10) OVER (PARTITION BY segment ORDER BY lifetime_value DESC) AS decile
FROM customers;
```

**Why it fails** : if a segment has fewer than 10 customers, only the first `N` buckets exist for that segment. Downstream code that joins on `decile=10` silently returns zero rows for small segments.

**Fix** : pre-check partition size, or use `PERCENT_RANK` / `CUME_DIST` which return a continuous value :

```sql
-- CORRECT : continuous distribution
SELECT
  customer_id,
  segment,
  PERCENT_RANK() OVER (PARTITION BY segment ORDER BY lifetime_value DESC) AS pr,
  CASE
    WHEN PERCENT_RANK() OVER (PARTITION BY segment ORDER BY lifetime_value DESC) < 0.1
      THEN 'top 10%'
    ELSE 'lower 90%'
  END AS bucket
FROM customers;
```

KB : https://mariadb.com/kb/en/ntile/.

---

## Anti-Pattern 6 : Assuming CTE Is Always Materialized

**Mistake** :

```sql
-- WRONG : assuming temp-table caching across references
WITH expensive AS (
  SELECT customer_id, complex_calculation(history) AS score
  FROM customer_history
)
SELECT * FROM expensive WHERE score > 100
UNION ALL
SELECT * FROM expensive WHERE score < 10;
```

**Why it fails** : `optimizer_switch=derived_merge=on` (default) MAY merge the CTE into each outer reference, causing `complex_calculation` to run TWICE instead of once. The "I'll just CTE it once and reuse it" mental model is incorrect.

**Fix** : disable derived merge for the session, or use a temporary table.

```sql
-- CORRECT : force materialization
SET SESSION optimizer_switch = 'derived_merge=off';

WITH expensive AS (
  SELECT customer_id, complex_calculation(history) AS score
  FROM customer_history
)
SELECT * FROM expensive WHERE score > 100
UNION ALL
SELECT * FROM expensive WHERE score < 10;
```

Alternative : `CREATE TEMPORARY TABLE expensive AS SELECT ...` and query the temp table. KB : https://mariadb.com/kb/en/optimizer-switch/.

---

## Anti-Pattern 7 : CYCLE Clause Used Before 10.5.2

**Mistake** :

```sql
-- WRONG on MariaDB < 10.5.2
WITH RECURSIVE g (a, b) AS (
  SELECT a, b FROM edges WHERE a = 1
  UNION ALL
  SELECT e.a, e.b FROM edges e JOIN g ON e.a = g.b
)
CYCLE a RESTRICT
SELECT * FROM g;
```

**Why it fails** : the `CYCLE col_list RESTRICT` syntax was added in 10.5.2. On 10.5.1 or earlier (or on community-built 10.4 still in production), this produces `ERROR 1064` syntax error at `CYCLE`.

**Fix** : check version. On 10.5.2+ use `CYCLE`. On 10.5.1 or earlier, use a path-string guard.

```sql
-- CORRECT pre-10.5.2 : path-string guard
WITH RECURSIVE g (a, b, path) AS (
  SELECT a, b, CAST(CONCAT(a, '->', b) AS CHAR(2000))
    FROM edges WHERE a = 1
  UNION ALL
  SELECT e.a, e.b,
         CAST(CONCAT(g.path, '->', e.b) AS CHAR(2000))
    FROM edges e JOIN g ON e.a = g.b
   WHERE LOCATE(CONCAT('->', e.b), g.path) = 0  -- skip if b already visited
)
SELECT * FROM g;
```

KB : https://mariadb.com/kb/en/with/.

---

## Anti-Pattern 8 : Window Function in WHERE Clause

**Mistake** :

```sql
-- WRONG : window function not allowed in WHERE
SELECT product_id, revenue
FROM monthly_product_revenue
WHERE RANK() OVER (ORDER BY revenue DESC) <= 10;
```

**Why it fails** : SQL evaluation order processes `WHERE` BEFORE `SELECT`-list expressions, so window functions are not allowed in `WHERE`. Error : `ERROR 1582 (HY000) : Window function ... is not allowed in WHERE`.

**Fix** : wrap in a CTE or subquery so the rank is computed in a derived stage you can filter.

```sql
-- CORRECT : CTE wrapper
WITH ranked AS (
  SELECT product_id, revenue,
         RANK() OVER (ORDER BY revenue DESC) AS rk
  FROM monthly_product_revenue
)
SELECT product_id, revenue
FROM ranked
WHERE rk <= 10;
```

KB : https://mariadb.com/kb/en/window-functions-overview/.

---

## Anti-Pattern 9 : Using EXCLUDE or NULLS FIRST in OVER

**Mistake** :

```sql
-- WRONG : neither EXCLUDE nor NULLS FIRST is supported
SELECT
  emp_id,
  SUM(salary) OVER (
    PARTITION BY dept_id
    ORDER BY hire_date NULLS FIRST
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    EXCLUDE CURRENT ROW
  )
FROM employees;
```

**Why it fails** : MariaDB's window-function grammar does not implement `EXCLUDE CURRENT ROW`, `EXCLUDE GROUP`, `EXCLUDE TIES`, `EXCLUDE NO OTHERS`, or `NULLS FIRST`/`NULLS LAST` inside `OVER (... ORDER BY ...)`. Both produce syntax errors. PostgreSQL and SQL:2011 standard support them ; MariaDB does not.

**Fix** : emulate via expression arithmetic or partition-level adjustment.

```sql
-- CORRECT : subtract current row from cumulative sum
SELECT
  emp_id,
  SUM(salary) OVER (PARTITION BY dept_id ORDER BY hire_date) - salary AS sum_others
FROM employees;
```

For NULL ordering, swap with `CASE WHEN hire_date IS NULL THEN 0 ELSE 1 END ASC, hire_date ASC`. KB : https://mariadb.com/kb/en/window-frames/.

---

## Anti-Pattern 10 : UNION (Not UNION ALL) in Recursive CTE

**Mistake** :

```sql
-- WRONG : deduplicating union in recursion
WITH RECURSIVE walk (n) AS (
  SELECT 1
  UNION
  SELECT n + 1 FROM walk WHERE n < 1000
)
SELECT * FROM walk;
```

**Why it fails** : `UNION` (without `ALL`) deduplicates the result of each recursive step, which forces materialization and a deduplication scan at every iteration. Performance collapses on large recursions. Allowed by syntax, but always a mistake unless you specifically need set-semantics.

**Fix** : use `UNION ALL`. Add an explicit `DISTINCT` in the outer SELECT if dedup is needed.

```sql
-- CORRECT
WITH RECURSIVE walk (n) AS (
  SELECT 1
  UNION ALL
  SELECT n + 1 FROM walk WHERE n < 1000
)
SELECT * FROM walk;
```

KB : https://mariadb.com/kb/en/recursive-common-table-expressions-overview/.

---

## Anti-Pattern 11 : DISTINCT Inside Aggregate Window

**Mistake** :

```sql
-- WRONG : COUNT(DISTINCT ...) not allowed as window function
SELECT
  order_date,
  COUNT(DISTINCT customer_id) OVER (ORDER BY order_date
    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS uniq_customers_7d
FROM orders;
```

**Why it fails** : MariaDB does not support `DISTINCT` inside aggregate-window calls. Error : `ERROR 4019 (HY000) : DISTINCT is not allowed in window functions`.

**Fix** : pre-aggregate to one row per (date, customer) in a CTE, then run the window count over that.

```sql
-- CORRECT
WITH per_day_customer AS (
  SELECT DISTINCT order_date, customer_id FROM orders
)
SELECT
  order_date,
  COUNT(*) OVER (ORDER BY order_date
    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS uniq_customers_7d
FROM per_day_customer
GROUP BY order_date;
```

KB : https://mariadb.com/kb/en/window-functions-overview/.

---

## Verification

All anti-patterns verified against MariaDB KB on 2026-05-19. Anti-pattern 1 (no GROUPS frame) is tracked in LESSONS.md as L-002.
