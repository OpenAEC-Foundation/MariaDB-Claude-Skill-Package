# Examples : Worked Query Optimizations

Eight end-to-end runs of the 7-step procedure. Each shows the user-supplied query, the EXPLAIN signal, the classification, the proposed fix, and the structured output. All SQL is annotated for MariaDB 10.6+.

## Example 1 : type=ALL query, missing index

**User query and plan**

```sql
-- 10.6+
EXPLAIN SELECT id, total FROM orders WHERE customer_id = 4711;
-- type: ALL | possible_keys: NULL | key: NULL | rows: 2000000 | Extra: Using where
```

**Procedure**

- Step 2 signal : `type=ALL`, `key=NULL`, `possible_keys=NULL`.
- Step 3 classification : missing index (no candidate index exists at all).
- Step 4 fix : add an index on the equality predicate column.
- Step 5 validation : `customer_id` is high-selectivity (many distinct customers, each owning a small fraction of rows). `SHOW INDEX FROM orders` shows no index on `customer_id` : not a duplicate.

```sql
-- 10.6+
CREATE INDEX ix_orders_customer ON orders (customer_id);
```

**Structured output**

```
Bottleneck      : Missing index
Root cause      : No index on customer_id, optimizer must scan all 2,000,000 rows
Proposed change : CREATE INDEX ix_orders_customer ON orders (customer_id);
Selectivity     : customer_id is high-selectivity, distinct-values close to total-rows / rows-per-customer
Expected effect : type=ALL becomes type=ref, rows drops from 2,000,000 to the per-customer count
Verification    : ANALYZE FORMAT=JSON SELECT id, total FROM orders WHERE customer_id = 4711;
```

## Example 2 : Using filesort, sort problem

**User query and plan**

```sql
-- 10.6+
EXPLAIN SELECT id, created_at FROM events
 WHERE channel = 'web' ORDER BY created_at DESC LIMIT 20;
-- type: ref | key: ix_events_channel | Extra: Using where; Using filesort
```

**Procedure**

- Step 2 signal : access is `type=ref` (acceptable) but `Using filesort` is present.
- Step 3 classification : sort problem. The index on `channel` alone cannot serve the `ORDER BY`.
- Step 4 fix : replace the single-column index with a composite ending in the sort column.
- Step 5 validation : `ix_events_channel(channel)` is a leftmost prefix of the proposed `(channel, created_at)`, so the old index becomes redundant : drop it as part of the fix.

```sql
-- 10.6+
CREATE INDEX ix_events_channel_created ON events (channel, created_at);
DROP INDEX ix_events_channel ON events;  -- now a redundant leftmost prefix
```

**Structured output**

```
Bottleneck      : Sort problem
Root cause      : ix_events_channel serves the equality filter but not the ORDER BY, forcing a filesort
Proposed change : composite index (channel, created_at); drop the now-redundant single-column index
Selectivity     : channel is the equality column, created_at provides ordered access for the sort
Expected effect : Using filesort disappears, the LIMIT 20 reads only 20 index entries
Verification    : ANALYZE FORMAT=JSON SELECT ... ORDER BY created_at DESC LIMIT 20;
```

## Example 3 : correlated subquery rewrite

**User query and plan**

```sql
-- 10.6+
EXPLAIN SELECT c.id, c.name FROM customers c
 WHERE (SELECT COUNT(*) FROM orders o WHERE o.customer_id = c.id) > 5;
-- select_type for the inner query: DEPENDENT SUBQUERY
```

**Procedure**

- Step 2 signal : `select_type=DEPENDENT SUBQUERY`. The subquery is re-executed once per customer row.
- Step 3 classification : subquery problem.
- Step 4 fix : a rewrite is the priority here, not an index. Rewrite the correlated subquery as a `JOIN` with `GROUP BY` so the optimizer can pick a single join order.

```sql
-- 10.6+ : correlated subquery rewritten as a join
SELECT c.id, c.name
  FROM customers c
  JOIN (SELECT customer_id FROM orders
         GROUP BY customer_id HAVING COUNT(*) > 5) busy
    ON busy.customer_id = c.id;
```

**Structured output**

```
Bottleneck      : Subquery problem
Root cause      : DEPENDENT SUBQUERY re-runs COUNT(*) once per customer row
Proposed change : rewrite the correlated subquery as a JOIN against a grouped derived table
Selectivity     : n/a (rewrite) ; an index on orders(customer_id) further speeds the GROUP BY
Expected effect : DEPENDENT SUBQUERY disappears, the orders aggregation runs once
Verification    : ANALYZE FORMAT=JSON SELECT ... JOIN (SELECT ... GROUP BY ...) ...;
```

## Example 4 : Using temporary, eliminate the temp table

**User query and plan**

```sql
-- 10.6+
EXPLAIN SELECT status, COUNT(*) FROM tickets
 WHERE project_id = 12 GROUP BY status;
-- type: ref | key: ix_tickets_project | Extra: Using where; Using temporary
```

**Procedure**

- Step 2 signal : access is acceptable, but `Using temporary` shows the `GROUP BY` needs a temp table.
- Step 3 classification : sort problem (GROUP BY variant : the optimizer cannot read groups in order from the index).
- Step 4 fix : extend the index so the `GROUP BY` column follows the equality predicate, letting the engine read pre-grouped.

```sql
-- 10.6+
CREATE INDEX ix_tickets_project_status ON tickets (project_id, status);
DROP INDEX ix_tickets_project ON tickets;  -- redundant leftmost prefix
```

**Structured output**

```
Bottleneck      : Sort problem (GROUP BY without ordered index access)
Root cause      : ix_tickets_project covers the filter but not status, so GROUP BY builds a temp table
Proposed change : composite index (project_id, status); drop the redundant single-column index
Selectivity     : project_id is the equality filter, status trails to provide grouped order
Expected effect : Using temporary disappears, groups are read directly from the index
Verification    : ANALYZE FORMAT=JSON SELECT status, COUNT(*) ... GROUP BY status;
```

## Example 5 : covering-index recommendation

**User query and plan**

```sql
-- 10.6+
EXPLAIN SELECT order_id, amount FROM payments WHERE method = 'card';
-- type: ref | key: ix_payments_method | rows: 80000 | Extra: Using where
```

**Procedure**

- Step 2 signal : `type=ref` is acceptable, but `rows=80000` means 80,000 row fetches just to read two columns.
- Step 3 classification : wrong index (correct access, but no covering benefit).
- Step 4 fix : covering-index opportunity. The query touches only `method`, `order_id`, `amount`. Extend the index to cover all three.
- Step 5 validation : `ix_payments_method(method)` is a leftmost prefix of `(method, order_id, amount)` : replace it, do not add alongside.

```sql
-- 10.6+
CREATE INDEX ix_payments_method_cover ON payments (method, order_id, amount);
DROP INDEX ix_payments_method ON payments;  -- redundant leftmost prefix
```

**Structured output**

```
Bottleneck      : Wrong index (no covering benefit)
Root cause      : ix_payments_method finds rows but every row is still fetched for order_id and amount
Proposed change : covering index (method, order_id, amount); drop the redundant prefix index
Selectivity     : method drives access ; order_id and amount are appended only to make the index covering
Expected effect : Extra shows Using index, the 80,000 row fetches are eliminated
Verification    : ANALYZE FORMAT=JSON SELECT order_id, amount FROM payments WHERE method = 'card';
```

## Example 6 : duplicate-index avoidance

**User proposal**

The user asks : "My query filters on `user_id`, should I add `CREATE INDEX ix_u ON sessions (user_id)`?"

**Procedure**

- Step 5 duplicate check runs first because the user proposed the index directly.

```sql
-- 10.6+
SHOW INDEX FROM sessions;
-- existing: ix_sessions_user_created on (user_id, created_at)
```

- The existing index `(user_id, created_at)` already has `user_id` as its leftmost prefix. By the leftmost-prefix rule it already serves a filter on `user_id` alone.
- The proposed `ix_u (user_id)` is redundant : it would only slow writes and waste disk.

**Structured output**

```
Bottleneck      : n/a (proposed index review)
Root cause      : ix_sessions_user_created already serves WHERE user_id = ? via the leftmost prefix
Proposed change : do NOT create ix_u ; the proposed index is redundant
Selectivity     : not applicable, no new index recommended
Expected effect : avoiding the duplicate keeps INSERT/UPDATE/DELETE fast and saves disk
Verification    : ANALYZE FORMAT=JSON SELECT ... WHERE user_id = ?  confirms the existing index is used
```

## Example 7 : join-order problem

**User query and plan**

```sql
-- 10.6+
EXPLAIN SELECT o.id, l.sku FROM orders o
  JOIN order_lines l ON l.order_id = o.id
 WHERE o.created_at >= '2026-01-01';
-- orders:      type: range | key: ix_orders_created
-- order_lines: type: ALL   | Extra: Using join buffer (Block Nested Loop)
```

**Procedure**

- Step 2 signal : the first table is fine ; `order_lines` shows `type=ALL` and `Using join buffer`.
- Step 3 classification : join-order problem. The join column `order_lines.order_id` has no index, so each outer row triggers a scan.
- Step 4 fix : index the join column on the second table.
- Step 5 validation : `order_id` is high-selectivity within `order_lines` (each order maps to a handful of lines). `SHOW INDEX FROM order_lines` confirms no index on `order_id`.

```sql
-- 10.6+
CREATE INDEX ix_order_lines_order ON order_lines (order_id);
```

**Structured output**

```
Bottleneck      : Join-order problem
Root cause      : order_lines.order_id has no index, so the join falls back to Block Nested Loop
Proposed change : CREATE INDEX ix_order_lines_order ON order_lines (order_id);
Selectivity     : order_id is high-selectivity inside order_lines, few lines per order
Expected effect : order_lines access becomes type=ref, the join buffer disappears
Verification    : ANALYZE FORMAT=JSON SELECT ... JOIN order_lines ...;
```

## Example 8 : full review of a slow JOIN

**User query and plan**

```sql
-- 10.6+
EXPLAIN SELECT * FROM invoices i
  JOIN customers c ON c.id = i.customer_id
 WHERE i.status = 'open' AND i.due_date < '2026-06-01'
 ORDER BY i.due_date;
-- invoices:  type: ALL | possible_keys: NULL | Extra: Using where; Using filesort
-- customers: type: eq_ref | key: PRIMARY
```

**Procedure**

- Step 2 signals : `invoices` is `type=ALL` with `key=NULL` AND `Using filesort`. `customers` is fine (`eq_ref` on the primary key).
- Step 3 classification : two classes on `invoices` : missing index plus a sort problem. Resolution order : fix the access path first.
- Step 4 fix : a composite index serving the equality predicate, the range predicate, and the sort. Order : equality (`status`) first, then the column used for both the range filter and the `ORDER BY` (`due_date`).
- Step 4 rewrite : `SELECT *` blocks any covering benefit and pulls wide rows ; recommend selecting only the needed columns.
- Step 5 validation : `status` has low selectivity (a few states) but is the only equality predicate, so it still belongs first ; `due_date` provides both the range and the order. `SHOW INDEX FROM invoices` confirms no existing index : not a duplicate.

```sql
-- 10.6+
CREATE INDEX ix_invoices_status_due ON invoices (status, due_date);
-- and rewrite SELECT * to the columns actually needed, for example:
-- SELECT i.id, i.due_date, i.amount, c.name FROM invoices i JOIN customers c ...
```

**Structured output**

```
Bottleneck      : Missing index plus sort problem on invoices
Root cause      : no index on invoices, full scan plus a filesort for ORDER BY due_date
Proposed change : composite index (status, due_date); replace SELECT * with explicit columns
Selectivity     : status is low-selectivity but the only equality term ; due_date serves range + sort
Expected effect : invoices access becomes type=range, Using filesort disappears
Verification    : ANALYZE FORMAT=JSON SELECT ... ORDER BY i.due_date; compare r_rows vs rows
```
