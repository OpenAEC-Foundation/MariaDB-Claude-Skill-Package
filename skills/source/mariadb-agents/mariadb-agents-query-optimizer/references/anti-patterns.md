# Anti-Patterns : Query-Optimizer Procedure

Six anti-patterns that derail a query optimization. Each shows the wrong move, why it fails, and the correct alternative. These are the failure modes the orchestration procedure exists to prevent.

## Anti-Pattern 1 : Recommending FORCE INDEX as a permanent fix

**Wrong**

```sql
-- 10.6+ : "the optimizer keeps choosing the wrong index, just force it"
SELECT * FROM orders FORCE INDEX (ix_orders_created)
 WHERE customer_id = 4711 AND created_at >= '2026-01-01';
```

**Why it fails**

`FORCE INDEX` overrides the cost model and tells the optimizer the table scan is prohibitively expensive. It locks the query to one plan. When the data shape changes (the table grows, the filter distribution shifts, statistics are refreshed) the forced plan becomes the slow one, and the query has no way to adapt. It hides the real problem instead of fixing it.

**Correct**

Find the root cause. If the optimizer rejects a good index because statistics are stale, run `ANALYZE TABLE orders` and re-check. If the index column order is wrong, build the right composite. `FORCE INDEX` is only a one-off diagnostic to confirm a hypothesis, and any recommendation that includes it MUST carry that caveat. For excluding a single bad path, `IGNORE INDEX (bad_one)` has a smaller blast radius because the rest of the cost model still adapts.

## Anti-Pattern 2 : Recommending an index without a selectivity check

**Wrong**

```sql
-- 10.6+ : "the query filters on is_active, add an index on it"
CREATE INDEX ix_users_active ON users (is_active);
```

**Why it fails**

`is_active` is a boolean : two distinct values across the whole table. Selectivity is roughly 2 / total-rows, which is near zero. The optimizer estimates that the index would still match half the table, so a full scan is cheaper, and it ignores the new index. The recommendation slowed every write and consumed disk for zero query benefit.

**Correct**

Always state the selectivity assumption before recommending an index. Selectivity is approximately distinct-values / total-rows. Lead a composite with the highest-selectivity equality column. If the only filter is low-selectivity, the honest answer is that an index will not help that column alone : look for an additional predicate to form a useful composite, or accept the scan.

## Anti-Pattern 3 : Adding an index that duplicates an existing one

**Wrong**

```sql
-- 10.6+ : existing index is ix_sessions_user_created (user_id, created_at)
CREATE INDEX ix_sessions_user ON sessions (user_id);
```

**Why it fails**

By the leftmost-prefix rule, `INDEX(user_id, created_at)` already serves any query that filters on `user_id` alone. The new single-column index is redundant : it answers nothing the existing index could not, while adding write cost to every `INSERT`, `UPDATE`, and `DELETE` on `sessions` and consuming extra disk.

**Correct**

Always run `SHOW INDEX FROM <table>` or `SHOW CREATE TABLE <table>` before recommending an index. If the proposed columns form a leftmost prefix of an existing index, recommend extending the existing index instead of adding a new one. If a leading-column-only index already exists where a wider composite is needed, replace the narrow one.

## Anti-Pattern 4 : Copying MySQL 8 optimizer hints into MariaDB

**Wrong**

```sql
-- 10.6+ : "MySQL 8 fixed this with a hint, paste it in"
SELECT /*+ NO_BNL(l) JOIN_ORDER(o, l) */ o.id, l.sku
  FROM orders o JOIN order_lines l ON l.order_id = o.id;
```

**Why it fails**

The `/*+ ... */` optimizer-hint comment syntax (`NO_BNL`, `JOIN_ORDER`, `BKA`, and similar) is a MySQL 8 feature. MariaDB does not implement it : the comment is silently ignored, so the user believes the query is optimized when nothing changed. The query stays slow and the false fix masks the real bottleneck.

**Correct**

MariaDB steers the optimizer through `optimizer_switch` flags and the SQL index hints `USE INDEX`, `FORCE INDEX`, `IGNORE INDEX` (with `FOR JOIN` / `FOR ORDER BY` / `FOR GROUP BY` modifiers). For a join that fell back to Block Nested Loop, the real fix is to index the join column, not to hint. Verify available flags with `SELECT @@optimizer_switch`.

## Anti-Pattern 5 : Optimizing a query without its EXPLAIN

**Wrong**

The user pastes a slow `SELECT` and the response immediately recommends an index based on reading the `WHERE` clause, with no plan requested.

**Why it fails**

The query text does not reveal the optimizer's actual choice. Whether an index exists, whether statistics are stale, how large the table is, and which `optimizer_switch` flags are active all change the plan. An index recommended from the `WHERE` clause alone may already exist, may be ignored for low selectivity, or may not address the real bottleneck (a filesort, a join buffer, a dependent subquery). It is a guess dressed as an answer.

**Correct**

Always require the `EXPLAIN` (or `EXPLAIN FORMAT=JSON`) output before proposing anything. If the user did not provide it, hand back the exact `EXPLAIN <their query>;` statement and wait. The plan is the required input to Step 2 of the procedure ; without it the procedure cannot start.

## Anti-Pattern 6 : Ignoring the ANALYZE actual-vs-estimate mismatch

**Wrong**

The proposed index does not speed the query up. The response concludes "the index does not help, the optimizer must need a hint" and moves on to `FORCE INDEX`.

**Why it fails**

`ANALYZE FORMAT=JSON` shows `rows: 50` (estimate) against `r_rows: 900000` (actual). The optimizer chose a plan based on statistics that say the table is tiny, when it is not. The index is fine ; the cost model is working from wrong numbers. Reaching for `FORCE INDEX` here papers over stale statistics and locks in a plan that will mislead again later.

**Correct**

Always run `ANALYZE FORMAT=JSON` after the change and compare `r_rows` with `rows`. A large mismatch means stale statistics, not a bad index : recommend `ANALYZE TABLE <table>` to refresh them, then re-verify. Only when `r_rows` and `rows` agree and the plan is still wrong is the cost model genuinely at fault, and even then `optimizer_trace` is the next step, not a permanent `FORCE INDEX`. Never recommend `ANALYZE UPDATE` or `ANALYZE DELETE` to inspect a plan : both execute the mutation.
