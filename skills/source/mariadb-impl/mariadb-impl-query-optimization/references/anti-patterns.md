# Anti-Patterns : Query Optimization

Eight real anti-patterns. Each contains the broken code, why it fails, and the correct alternative.

## 1. Trusting the `rows` column blindly

### Wrong

```sql
-- 10.6+
EXPLAIN SELECT * FROM orders WHERE customer_id = 4711;
-- rows = 50000  : "looks fine"
```

The developer reads `rows=50000`, thinks "small enough", and ships. Production users complain.

### Why it fails

`rows` is an OPTIMIZER ESTIMATE based on statistics. After bulk loads, ALTER TABLE rebuilds, or large DELETE, statistics drift. The actual row count may be 5 or 5 million.

### Correct

```sql
-- 10.6+
ANALYZE FORMAT=JSON
  SELECT * FROM orders WHERE customer_id = 4711;
-- Compare r_rows (actual) vs rows (estimate). Mismatch means stale stats.

ANALYZE TABLE orders;                       -- refresh engine stats
ANALYZE TABLE orders PERSISTENT FOR ALL;    -- richer EITS
```

Rule : ALWAYS validate the estimate via `ANALYZE FORMAT=JSON` before claiming a plan is good.

## 2. `FORCE INDEX` without re-checking after statistics change

### Wrong

```sql
-- 10.6+
SELECT * FROM orders FORCE INDEX (idx_status)
 WHERE status = 'pending'
   AND customer_id = 4711;
-- Shipped 6 months ago. Worked fine then.
```

After a bulk load doubled the `pending` rows but added a much better composite index `idx_cust_status` the year after, this query is now stuck on the worse index.

### Why it fails

`FORCE INDEX` makes the optimizer treat table scans as prohibitively expensive AND ignores any newer, better index that is not in the hint list. The hint hard-codes a plan against an outdated reality.

### Correct

```sql
-- 10.6+
-- Diagnose : prove the better index wins
EXPLAIN SELECT * FROM orders FORCE INDEX (idx_cust_status)
        WHERE status = 'pending' AND customer_id = 4711;

-- Fix at the root : drop the bad index OR exclude it by smaller-blast hint
SELECT * FROM orders IGNORE INDEX (idx_status)
 WHERE status = 'pending' AND customer_id = 4711;
```

Rule : `FORCE INDEX` is a diagnostic, not a production strategy. Prefer `IGNORE INDEX` and let the cost model adapt as data shifts.

## 3. `type=ALL` on a large table in production

### Wrong

```sql
-- 10.6+
SELECT * FROM audit_log WHERE action = 'login_failure';
-- EXPLAIN shows type=ALL, table has 200M rows.
```

Shipped because "it works on dev" (10k rows).

### Why it fails

A full table scan reads every page. On 200M rows that is gigabytes of IO and seconds-to-minutes of wall time. Concurrent queries pile up, connections exhaust, application stalls.

### Correct

```sql
-- 10.6+
-- Add an index that supports the predicate
ALTER TABLE audit_log ADD INDEX idx_audit_action (action);
ANALYZE TABLE audit_log;

-- Verify
EXPLAIN SELECT * FROM audit_log WHERE action = 'login_failure';
-- type=ref, key=idx_audit_action
```

Rule : NEVER ship a query whose `EXPLAIN type=ALL` against a table over a few thousand rows. CI-gate this with `pt-query-digest` or a pre-deploy EXPLAIN check.

## 4. Outdated statistics after bulk import

### Wrong

```sql
-- 10.6+
LOAD DATA INFILE '/data/orders-q1.csv' INTO TABLE orders;
-- 10M rows loaded. Reports immediately query orders. Optimizer uses old stats.
SELECT customer_id, SUM(total) FROM orders WHERE created_at >= '2026-01-01' GROUP BY customer_id;
-- Plan reads through Block Nested Loop, full scan.
```

### Why it fails

InnoDB's `innodb_stats_auto_recalc` triggers on 10% changes but may not run immediately. EITS is NEVER auto-refreshed. The optimizer sees the OLD row counts and picks plans that were correct for the smaller table.

### Correct

```sql
-- 10.6+
LOAD DATA INFILE '/data/orders-q1.csv' INTO TABLE orders;

ANALYZE TABLE orders;                      -- engine stats
ANALYZE TABLE orders PERSISTENT FOR ALL;   -- EITS
```

Rule : ALWAYS run `ANALYZE TABLE` after bulk inserts > 10% of table size, after restore-from-backup, and after large DELETE.

## 5. Copying MySQL 8 `USE INDEX` clauses verbatim

### Wrong

```sql
-- Copied from a MySQL 8 codebase
SELECT * FROM orders USE INDEX (idx_created_at)
        /*+ NO_BNL(orders) */                 -- MySQL 8 optimizer hint
 WHERE created_at >= '2026-01-01';
```

### Why it fails

MariaDB does not implement MySQL 8 inline optimizer hints (`/*+ NO_BNL */`, `/*+ JOIN_FIXED_ORDER */`). The comment is parsed and ignored, so behaviour silently diverges from the source database. MariaDB's `optimizer_switch` flag names are also different : `mrr` (off by default in MariaDB) vs MySQL's `block_nested_loop` ; MariaDB has `rowid_filter`, MySQL does not.

### Correct

```sql
-- 10.6+ : use MariaDB's mechanisms
SELECT * FROM orders USE INDEX (idx_created_at)
 WHERE created_at >= '2026-01-01';

-- Tune via session optimizer_switch instead of inline hints
SET SESSION optimizer_switch = 'mrr=off,join_cache_hashed=off';
```

Rule : when migrating from MySQL 8 ALWAYS audit for `/*+ ... */` hints and `SET ... = INVISIBLE` index toggles. Translate to MariaDB equivalents (`IGNORED` index in 10.6+, `optimizer_switch` for hint behaviour).

## 6. Ignoring `Using filesort` on millions of rows

### Wrong

```sql
-- 10.6+
SELECT * FROM events
 WHERE user_id = 4711
 ORDER BY occurred_at DESC
 LIMIT 100;
-- EXPLAIN : Extra = "Using where; Using filesort"
-- Table has 50M rows, user_id selects 200000 of them.
```

The developer sees a `LIMIT 100`, assumes it will be fast, and ships.

### Why it fails

`Using filesort` means the optimizer fetches ALL 200000 matching rows, sorts them in memory or on disk, THEN applies `LIMIT 100`. The LIMIT does not save the sort. The full sort dominates the query time.

### Correct

```sql
-- 10.6+
ALTER TABLE events ADD INDEX idx_events_user_occurred (user_id, occurred_at);
ANALYZE TABLE events;

EXPLAIN SELECT * FROM events
        WHERE user_id = 4711
        ORDER BY occurred_at DESC
        LIMIT 100;
-- type=ref, no filesort ; optimizer walks the index in reverse and stops at 100.
```

Rule : ALWAYS treat `Using filesort` on a large row count as a fixable bug. Composite index matching `WHERE columns + ORDER BY column` (in that order) usually eliminates it.

## 7. View with `ALGORITHM=TEMPTABLE` inside a hot path

### Wrong

```sql
-- 10.6+
CREATE ALGORITHM=TEMPTABLE VIEW v_recent_orders AS
  SELECT * FROM orders WHERE created_at >= NOW() - INTERVAL 30 DAY;

-- Application calls this view thousands of times per minute :
SELECT id, total FROM v_recent_orders WHERE customer_id = 4711;
```

### Why it fails

`ALGORITHM=TEMPTABLE` materializes the ENTIRE view body into a temporary table on every reference. There is no index pushdown into the inner SELECT. With 5M recent orders, each call rebuilds a 5M-row temp table just to filter 12 rows.

### Correct

```sql
-- 10.6+
-- Default algorithm is UNDEFINED ; optimizer chooses MERGE when possible.
CREATE VIEW v_recent_orders AS
  SELECT * FROM orders WHERE created_at >= NOW() - INTERVAL 30 DAY;

-- Now the predicate "customer_id = 4711" is pushed into the underlying
-- table access via condition_pushdown_for_derived (default ON).
```

Rule : NEVER use `ALGORITHM=TEMPTABLE` unless the view body has non-mergeable constructs (subqueries with GROUP BY, UNION, DISTINCT) AND the materialization cost is justified. Default to `UNDEFINED` and trust the optimizer.

## 8. Treating `type=index` as "good"

### Wrong

```sql
-- 10.6+
EXPLAIN SELECT * FROM customers
        WHERE annual_spend > 50000;
-- type=index, key=idx_annual_spend, rows=400000
-- "Index is being used, good enough."
```

### Why it fails

`type=index` means FULL INDEX SCAN, not range scan. The optimizer reads every entry in `idx_annual_spend` (400000 rows), then fetches each base row to apply the predicate. It is only slightly better than `type=ALL` because the index leaf pages are smaller than the data pages.

### Correct

```sql
-- 10.6+
-- Verify with ANALYZE
ANALYZE FORMAT=JSON
  SELECT * FROM customers WHERE annual_spend > 50000;
-- r_rows confirms 400000 actual reads.

-- Add a covering index OR a composite that allows range access
ALTER TABLE customers ADD INDEX idx_spend_country (annual_spend, country);
ANALYZE TABLE customers;

EXPLAIN SELECT * FROM customers WHERE annual_spend > 50000;
-- type=range, key=idx_spend_country  : real range scan
```

Rule : the `type` ranking is best-to-worst. Anything below `range` (i.e. `index` or `ALL`) deserves attention. ALWAYS treat `type=index` as "needs investigation", NEVER as "good enough".
