# Examples : Query Optimization

Ten worked examples. Each shows the symptom, the diagnostic, and the fix.

## Example 1 : Reading EXPLAIN on a slow JOIN

Symptom : two-table join takes seconds, expected milliseconds.

```sql
-- 10.6+
EXPLAIN
SELECT o.id, c.name
  FROM orders o
  JOIN customers c ON c.id = o.customer_id
 WHERE o.created_at >= '2026-01-01';
```

```text
id  select_type  table  type   key             rows    Extra
 1  SIMPLE       o      ALL    NULL            1500000 Using where
 1  SIMPLE       c      eq_ref PRIMARY               1
```

Read : `type=ALL` on `orders`, no index used. Optimizer expects 1.5M rows scanned. The predicate `created_at >= '2026-01-01'` has no supporting index.

Fix :

```sql
-- 10.6+
ALTER TABLE orders ADD INDEX idx_orders_created_at (created_at);
ANALYZE TABLE orders;
```

After : `type=range`, `key=idx_orders_created_at`, `rows` drops to expected slice.

## Example 2 : Using filesort caused by ORDER BY

Symptom : query has an index on `created_at` but still does `Using filesort`.

```sql
-- 10.6+
EXPLAIN
SELECT id, customer_id
  FROM orders
 WHERE customer_id = 4711
 ORDER BY created_at DESC
 LIMIT 50;
```

```text
type=ref  key=idx_orders_customer_id  rows=8000  Extra=Using where; Using filesort
```

The index supports the WHERE but not the ORDER BY. Optimizer reads 8000 rows, then sorts.

Fix : composite index matching `WHERE + ORDER BY` :

```sql
-- 10.6+
ALTER TABLE orders ADD INDEX idx_cust_created (customer_id, created_at);
ANALYZE TABLE orders;
```

`type=ref`, no `Using filesort` ; optimizer walks the index in the right order and stops after 50 rows.

## Example 3 : ANALYZE FORMAT=JSON to compare estimate vs actual

Symptom : EXPLAIN says 50000 rows, query runs much slower than expected.

```sql
-- 10.6+ (feature 10.1+)
ANALYZE FORMAT=JSON
  SELECT *
    FROM orders
   WHERE customer_id = 4711
     AND status = 'pending';
```

Output (excerpt) :

```json
{
  "table": {
    "table_name": "orders",
    "access_type": "ref",
    "key": "idx_orders_customer_id",
    "rows": 50000,
    "r_rows": 2,
    "filtered": 100,
    "r_filtered": 0.004,
    "r_total_time_ms": 124.7
  }
}
```

Read : `rows=50000` estimate, `r_rows=2` actual. Massive mismatch : statistics or histograms are stale.

Fix :

```sql
-- 10.6+
ANALYZE TABLE orders PERSISTENT FOR ALL;
```

## Example 4 : Comparing FORCE INDEX vs IGNORE INDEX

Symptom : optimizer picks `idx_status` (low selectivity), bypasses `idx_cust_created`.

```sql
-- 10.6+
-- Diagnose : force the good index
EXPLAIN SELECT * FROM orders
        FORCE INDEX (idx_cust_created)
        WHERE customer_id = 4711 AND status = 'pending';
-- type=ref, rows=12  : confirms the good index is faster
```

Production fix is NOT `FORCE INDEX`. Prefer `IGNORE INDEX` to exclude the bad path :

```sql
-- 10.6+
SELECT * FROM orders
 IGNORE INDEX (idx_status)
 WHERE customer_id = 4711 AND status = 'pending';
```

Why : `IGNORE INDEX` leaves the rest of the cost model active, so when a new better index is added later, the plan adapts.

## Example 5 : optimizer_trace to understand a rejected index

Symptom : `idx_cust_created` exists but optimizer picks `idx_status`. EXPLAIN says nothing useful.

```sql
-- 10.4.3+
SET optimizer_trace = 'enabled=on';

SELECT id FROM orders
 WHERE customer_id = 4711 AND status = 'pending';

SELECT TRACE
  FROM INFORMATION_SCHEMA.OPTIMIZER_TRACE\G

SET optimizer_trace = 'enabled=off';
```

Reading the trace : look under `considered_access_paths` for `idx_cust_created`. The `cost` and `rows` fields show the optimizer's calculation. If `rows` is much higher than reality, statistics are wrong. If `cost` is high despite low `rows`, the cost model is mis-tuned for this case ; consider `optimizer_use_condition_selectivity=4` or adding a histogram on `status`.

## Example 6 : Statistics refresh after bulk insert

Symptom : nightly batch loaded 5M rows, queries against the new data run with `Using join buffer (Block Nested Loop)`.

```sql
-- 10.6+
LOAD DATA INFILE '/data/orders-2026-Q1.csv'
  INTO TABLE orders;

-- InnoDB auto-recalc may not have triggered for some indexes ; force it.
ANALYZE TABLE orders;
ANALYZE TABLE orders PERSISTENT FOR ALL;   -- richer EITS

SET SESSION use_stat_tables = 'PREFERABLY';
```

After : optimizer sees correct row counts, switches from `Block Nested Loop` to indexed `ref` access.

## Example 7 : derived_with_keys flag impact

Symptom : a CTE / derived table makes the join slow.

```sql
-- 10.6+
EXPLAIN
WITH big_customers AS (
   SELECT id FROM customers WHERE annual_spend > 100000
)
SELECT o.id, o.total
  FROM orders o
  JOIN big_customers bc ON bc.id = o.customer_id
 WHERE o.created_at >= '2026-01-01';
```

If `Extra` shows `Using join buffer (Block Nested Loop)` against the derived table : the optimizer materialized it without a key.

Toggle :

```sql
-- 10.6+
SET SESSION optimizer_switch = 'derived_with_keys=on';
```

With `derived_with_keys=on` (default) the optimizer adds an auto-key on the derived result, eliminating the buffered join. Verify with `EXPLAIN` ; you should see `Using index` on the derived table.

## Example 8 : ICP (Index Condition Pushdown) verification

Symptom : query reads many rows even when EXPLAIN shows an index.

```sql
-- 10.6+
EXPLAIN
SELECT id FROM orders
 WHERE customer_id = 4711
   AND status = 'pending';
```

```text
type=ref  key=idx_cust_status  Extra=Using where
```

`Using where` (no `Using index condition`) means the predicate is evaluated AFTER row fetch. ICP can be confirmed by re-running with the flag explicitly :

```sql
-- 10.6+
SET SESSION optimizer_switch = 'index_condition_pushdown=on';
EXPLAIN SELECT id FROM orders
        WHERE customer_id = 4711 AND status = 'pending';
-- Extra should now show "Using index condition"
```

If still `Using where`, the index column order does not support pushdown. Make `idx_cust_status` include both columns in WHERE order.

## Example 9 : optimizer_switch session-scope diagnostic

Symptom : a 10.6 instance behaves differently from a 10.5 instance for the same query.

```sql
-- 10.6+
-- Inspect current value
SELECT @@SESSION.optimizer_switch;

-- Reproduce the older 10.5 default by disabling 10.6-introduced flags
SET SESSION optimizer_switch = 'rowid_filter=off,not_null_range_scan=off';

EXPLAIN <query>;
-- Compare with the original. If the plan is the same, those flags
-- are not the cause. Re-enable and continue.
```

Workflow rule : NEVER set new flags globally to "match older version" without measuring across the entire query workload.

## Example 10 : Detecting Using temporary on GROUP BY

Symptom : GROUP BY aggregation is slow.

```sql
-- 10.6+
EXPLAIN
SELECT customer_id, COUNT(*) AS order_count, SUM(total) AS total_spend
  FROM orders
 WHERE created_at >= '2026-01-01'
 GROUP BY customer_id;
```

```text
type=range  key=idx_orders_created_at  rows=200000  Extra=Using where; Using temporary; Using filesort
```

Two red flags : `Using temporary` (intermediate aggregation table) and `Using filesort` (final sort).

Fix : a covering index that supports both the filter and the GROUP BY column lets the optimizer use a loose-index scan :

```sql
-- 10.6+
ALTER TABLE orders ADD INDEX idx_cust_created_total (customer_id, created_at, total);
ANALYZE TABLE orders;
```

Re-EXPLAIN : `Extra` should now show `Using index for group-by` (loose-scan optimization), no temporary, no filesort.

## Bonus : Slow Query Log to find candidates

Symptom : "the application is slow" but no single query is identified.

```sql
-- 10.6+
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 0.5;
SET GLOBAL log_queries_not_using_indexes = ON;
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';
```

Wait one production cycle, aggregate with `pt-query-digest` :

```bash
pt-query-digest /var/log/mysql/slow.log | head -100
```

Top three queries by `Query_time_sum` are your optimization targets. Run EXPLAIN against each, then ANALYZE FORMAT=JSON to validate estimates, then fix indexes or rewrite.
