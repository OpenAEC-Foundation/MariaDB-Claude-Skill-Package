# Examples : Window Functions and CTEs in MariaDB

Twelve working examples covering the canonical use-cases. All examples target MariaDB 10.6+ unless annotated otherwise.

---

## Example 1 : Top-N Per Group (Top-3 highest-paid per department)

```sql
-- 10.2.1+
WITH ranked AS (
  SELECT
    emp_id,
    dept_id,
    salary,
    ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rn
  FROM employees
)
SELECT emp_id, dept_id, salary
FROM ranked
WHERE rn <= 3
ORDER BY dept_id, rn;
```

Why `ROW_NUMBER` not `RANK` : ties on salary would produce 4+ rows per dept with `RANK`. Use `RANK` when the business wants "all rows tied at top-3".

---

## Example 2 : Running Total (Cumulative Sum by Date)

```sql
-- 10.2+
SELECT
  order_date,
  daily_total,
  SUM(daily_total) OVER (ORDER BY order_date) AS running_total
FROM daily_revenue
ORDER BY order_date;
```

Default frame `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` gives a running total automatically when `ORDER BY` is present. NO explicit frame needed.

---

## Example 3 : LAG and LEAD for Period-over-Period Comparison

```sql
-- 10.2+
SELECT
  month,
  revenue,
  LAG(revenue, 1)  OVER (ORDER BY month) AS prev_month,
  LEAD(revenue, 1) OVER (ORDER BY month) AS next_month,
  revenue - LAG(revenue, 1, 0) OVER (ORDER BY month) AS mom_delta
FROM monthly_revenue
ORDER BY month;
```

The `0` argument to `LAG` is the default returned for the first row (no prior row). Without it, the first row's `mom_delta` would be NULL.

---

## Example 4 : Percentile Bucketing with NTILE

```sql
-- 10.2+
SELECT
  customer_id,
  lifetime_value,
  NTILE(4) OVER (ORDER BY lifetime_value DESC) AS quartile
FROM customers;
```

Quartile `1` is the top 25%, `4` is the bottom 25%. NEVER call `NTILE` on a partition with fewer rows than `n` and expect every bucket populated : empty buckets are simply absent.

---

## Example 5 : Named Window for DRY Spec

```sql
-- 10.2+
SELECT
  emp_id,
  dept_id,
  salary,
  RANK()       OVER w AS dept_rank,
  AVG(salary)  OVER w AS dept_avg,
  COUNT(*)     OVER w AS dept_count
FROM employees
WINDOW w AS (PARTITION BY dept_id ORDER BY salary DESC);
```

Writing `OVER (PARTITION BY dept_id ORDER BY salary DESC)` three times invites typos. The `WINDOW` clause keeps the spec single-source.

---

## Example 6 : LAST_VALUE Done Right (Partition-tail Lookup)

```sql
-- 10.2+
SELECT
  order_id,
  customer_id,
  order_date,
  amount,
  LAST_VALUE(order_date) OVER (
    PARTITION BY customer_id
    ORDER BY order_date
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
  ) AS most_recent_order_in_partition
FROM orders;
```

WITHOUT the explicit `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`, `LAST_VALUE` would return the CURRENT row's `order_date` (because default frame `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`). This is the most common LAST_VALUE bug.

---

## Example 7 : Non-Recursive CTE for Readability

```sql
-- 10.2.1+
WITH
  active_customers AS (
    SELECT id FROM customers WHERE status = 'active'
  ),
  recent_orders AS (
    SELECT customer_id, total
    FROM orders
    WHERE order_date >= CURRENT_DATE - INTERVAL 30 DAY
  )
SELECT c.id, COALESCE(SUM(o.total), 0) AS spent_30d
FROM active_customers c
LEFT JOIN recent_orders o ON o.customer_id = c.id
GROUP BY c.id;
```

Two named building blocks beat a 5-level-deep subquery for review and debugging. The optimizer will merge or materialize each CTE based on `optimizer_switch`.

---

## Example 8 : Recursive CTE for Organizational Hierarchy

```sql
-- 10.2.2+
WITH RECURSIVE org_tree (emp_id, manager_id, full_name, depth, path) AS (
  SELECT emp_id, manager_id, full_name, 0,
         CAST(emp_id AS CHAR(1000))
    FROM employees
   WHERE manager_id IS NULL   -- anchor : CEO and top-level managers
  UNION ALL
  SELECT e.emp_id, e.manager_id, e.full_name,
         t.depth + 1,
         CONCAT(t.path, ',', e.emp_id)
    FROM employees e
    JOIN org_tree t ON e.manager_id = t.emp_id
   WHERE t.depth < 20         -- depth guard : terminator
)
SELECT emp_id, manager_id, full_name, depth, path
FROM org_tree
ORDER BY path;
```

The `depth < 20` guard prevents runaway recursion if the data accidentally cycles. ALWAYS include a guard OR use `CYCLE col RESTRICT` (10.5.2+) ; never rely solely on `max_recursive_iterations`.

---

## Example 9 : CYCLE Clause for Built-in Cycle Detection (10.5.2+)

```sql
-- 10.5.2+
WITH RECURSIVE reachable (src, dst) AS (
  SELECT src, dst FROM edges WHERE src = 1   -- anchor : start node
  UNION ALL
  SELECT e.src, e.dst
    FROM edges e
    JOIN reachable r ON e.src = r.dst
)
CYCLE dst RESTRICT
SELECT DISTINCT dst AS reachable_from_node_1
FROM reachable;
```

Without `CYCLE dst RESTRICT`, a graph with cycles would recurse forever until `max_recursive_iterations` aborted. The CYCLE clause is the MariaDB-native way to handle cycles.

---

## Example 10 : Gap-and-Island Reduction

```sql
-- 10.2.1+
WITH numbered AS (
  SELECT
    event_date,
    user_id,
    ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY event_date) AS rn,
    DATEDIFF(event_date, '2026-01-01') -
      ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY event_date) AS grp
  FROM daily_login_events
)
SELECT
  user_id,
  MIN(event_date) AS streak_start,
  MAX(event_date) AS streak_end,
  COUNT(*)        AS streak_days
FROM numbered
GROUP BY user_id, grp
ORDER BY user_id, streak_start;
```

Classic gap-and-island : the difference between a date-based counter and a row-counter is constant within a consecutive streak.

---

## Example 11 : Moving Average with ROWS Frame

```sql
-- 10.2+
SELECT
  trade_date,
  close_price,
  AVG(close_price) OVER (
    ORDER BY trade_date
    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
  ) AS sma_7,
  AVG(close_price) OVER (
    ORDER BY trade_date
    ROWS BETWEEN 29 PRECEDING AND CURRENT ROW
  ) AS sma_30
FROM stock_prices
WHERE ticker = 'OPENAEC'
ORDER BY trade_date;
```

`ROWS` counts rows ; `RANGE` counts ORDER-BY-value distance. For a 7-trading-day moving average, ROWS is correct. For a 7-calendar-day moving average over a date column, use `RANGE BETWEEN INTERVAL 6 DAY PRECEDING AND CURRENT ROW`.

---

## Example 12 : Filtering by Window-Function Result via CTE

```sql
-- 10.2.1+
WITH product_rank AS (
  SELECT
    product_id,
    revenue,
    RANK() OVER (ORDER BY revenue DESC) AS rk
  FROM monthly_product_revenue
  WHERE month = '2026-04'
)
SELECT product_id, revenue
FROM product_rank
WHERE rk <= 10
ORDER BY rk;
```

Window functions are evaluated AFTER `WHERE` in the SQL evaluation order, so they cannot appear in `WHERE`. The CTE shifts the rank computation into a derived stage you CAN filter.

---

## Example 13 : Recursive CTE for Date Series Generation

```sql
-- 10.2.2+
WITH RECURSIVE date_series (d) AS (
  SELECT DATE '2026-01-01'
  UNION ALL
  SELECT d + INTERVAL 1 DAY FROM date_series WHERE d < DATE '2026-12-31'
)
SELECT
  d,
  COALESCE(SUM(o.total), 0) AS daily_revenue
FROM date_series
LEFT JOIN orders o ON o.order_date = date_series.d
GROUP BY d
ORDER BY d;
```

Useful when reporting needs zero-filled days. The terminator `WHERE d < DATE '2026-12-31'` caps the recursion ; `max_recursive_iterations=1000` would only allow ~33 months otherwise.

---

## Example 14 : Aggregate Over Full Partition Alongside Detail Rows

```sql
-- 10.2+
SELECT
  order_id,
  customer_id,
  amount,
  SUM(amount) OVER (PARTITION BY customer_id) AS customer_total,
  amount / SUM(amount) OVER (PARTITION BY customer_id) AS share_of_customer
FROM orders;
```

`PARTITION BY` without `ORDER BY` gives the whole-partition sum, exactly what `GROUP BY` would produce except every detail row is retained.

---

## Verification

Every code example was verified against the cited KB page on 2026-05-19. No example uses `GROUPS` frame (not supported), `EXCLUDE` clauses (not supported), or `NULLS FIRST`/`NULLS LAST` (not supported in `OVER ORDER BY`).
