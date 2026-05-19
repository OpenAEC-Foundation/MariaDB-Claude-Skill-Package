---
name: mariadb-impl-query-optimization
description: >
  Use when a query is slow, when reading EXPLAIN output, when deciding on index hints, when tuning optimizer_switch flags, when using optimizer_trace, or when comparing MariaDB and MySQL optimizer behavior.
  Prevents the common mistake of trusting type=ALL queries in production, blindly applying USE INDEX without checking selectivity, leaving outdated statistics, or copying MySQL 8 optimizer assumptions to MariaDB.
  Covers EXPLAIN reading column-by-column, EXPLAIN FORMAT=JSON, ANALYZE FORMAT=JSON (actual execution stats), index hints USE/FORCE/IGNORE INDEX, optimizer_switch flags (MariaDB-specific), optimizer_trace, ANALYZE TABLE for statistics, persistent vs in-memory statistics.
  Keywords: EXPLAIN, EXPLAIN FORMAT JSON, ANALYZE FORMAT JSON, query plan, optimizer, optimizer_switch, optimizer_trace, USE INDEX, FORCE INDEX, IGNORE INDEX, type ALL, type ref, type range, key_len, rows, filtered, Using filesort, Using temporary, Using index, ANALYZE TABLE, persistent statistics, why is my query slow, slow query, query is slow, query takes forever, full table scan, missing index, wrong index, stale statistics, how do I read EXPLAIN, how do I optimize a query, what does type=ALL mean, getting started with query optimization
license: MIT
compatibility: "Designed for Claude Code. Requires MariaDB 10.6-LTS, 10.11-LTS, 11.x, 12.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# MariaDB Query Optimization

How to read `EXPLAIN`, validate with `ANALYZE`, steer the optimizer via `optimizer_switch` and index hints, capture decisions with `optimizer_trace`, and keep statistics fresh with `ANALYZE TABLE`. Use this skill any time a query is slow, before adding an index, and before introducing a hint.

## Quick Reference

- `EXPLAIN` returns the plan ; `ANALYZE FORMAT=JSON` runs the query and shows actual rows next to estimates.
- `type=ALL` means full table scan : unacceptable on any table over a few thousand rows. Boundary "fine vs fix now" sits at `type=range`. `type=index` is a full index scan, not a range scan.
- `optimizer_switch` flags are MariaDB-specific. NEVER copy MySQL 8 optimizer assumptions ; flag names and defaults differ (e.g. `rowid_filter`, `condition_pushdown_for_subquery`, `split_materialized`).
- `optimizer_trace` (10.4.3+) is the deep-dive tool : `SET optimizer_trace='enabled=on'` then read `INFORMATION_SCHEMA.OPTIMIZER_TRACE`.
- ALWAYS run `ANALYZE TABLE` after bulk inserts or schema changes. Stale statistics drive bad plans.
- ALWAYS use `ANALYZE SELECT`, NEVER `ANALYZE UPDATE` or `ANALYZE DELETE` for diagnostics : the last two execute the mutation.
- `FORCE INDEX` overrides the cost model. Use it for verification, NOT as a long-term production fix : it locks you to a plan that breaks when statistics or data shape change.
- Index hints accept `FOR JOIN`, `FOR ORDER BY`, `FOR GROUP BY` modifiers. Default scope is `FOR JOIN` (affects WHERE).

## Decision Tree : "My query is slow"

1. **Get the plan** : run `EXPLAIN <query>`. Read the `type` column first.
2. **If `type=ALL` on a large table** : the optimizer is doing a full table scan. Go to step 4.
3. **If `type=index`** : full index scan (not as bad as ALL, but not a range scan). Check whether a more selective composite index would help.
4. **Check `Extra` for red flags** : `Using filesort` and `Using temporary` indicate the optimizer cannot satisfy ORDER BY or GROUP BY from an index. Composite index that matches `filter + sort` columns usually fixes it.
5. **Verify the estimate** : run `ANALYZE FORMAT=JSON <query>`. Compare `r_rows` (actual) vs `rows` (estimate). Large mismatch means stale statistics : run `ANALYZE TABLE`.
6. **If estimates are correct but the plan is still bad** : check `optimizer_switch` defaults. Some MariaDB flags are OFF by default (`mrr`, `mrr_cost_based`, `index_merge_sort_intersection`). Toggle for the session and re-EXPLAIN.
7. **If the optimizer rejects a clearly better index** : enable `optimizer_trace`, read the trace, find the cost calculation that rejected the index.
8. **Last resort** : use `IGNORE INDEX (bad_idx)` to exclude the bad path. `FORCE INDEX` only for one-off verification, not for shipped code.

## Reading the EXPLAIN Output

| Column | What it means | What to watch |
|--------|---------------|---------------|
| `id` | Statement / subquery sequence | Higher id = deeper subquery ; same id = joined together |
| `select_type` | Statement role | `SIMPLE`, `PRIMARY`, `SUBQUERY`, `DEPENDENT SUBQUERY` (BAD : re-executed per outer row), `DERIVED`, `MATERIALIZED`, `UNION`, `LATERAL DERIVED` |
| `table` | Physical or derived table | Aliases shown as `<derivedN>` for inline subqueries |
| `type` | Access method | Best to worst : `system`, `const`, `eq_ref`, `ref`, `ref_or_null`, `index_subquery`, `unique_subquery`, `index_merge`, `range`, `index`, `ALL`. Fix anything worse than `range`. |
| `possible_keys` | Indexes the optimizer considered | NULL = no usable index found |
| `key` | Index actually chosen | NULL = full table scan |
| `key_len` | Bytes of the index used | Short value on a composite key means trailing columns are not used |
| `ref` | What is compared against the index | `const` good, column name = join |
| `rows` | Estimated rows the optimizer expects to read | Estimate ; verify against `r_rows` from ANALYZE |
| `Extra` | Notes on plan execution | Watch for `Using filesort`, `Using temporary`, `Using join buffer`, `Range checked for each record` |

```sql
-- 10.6+
EXPLAIN SELECT o.id, c.name
   FROM orders o
   JOIN customers c ON c.id = o.customer_id
  WHERE o.created_at >= '2026-01-01'
  ORDER BY o.created_at DESC
  LIMIT 50;

-- type=ALL on orders + Using filesort + Using temporary = composite index
-- on (created_at DESC, customer_id) almost always fixes both.
```

## EXPLAIN FORMAT=JSON and ANALYZE FORMAT=JSON

`EXPLAIN FORMAT=JSON` returns a structured tree with per-node `r_filtered`, `read_cost`, `eval_cost`, and access strategy.

`ANALYZE FORMAT=JSON` (10.1+) actually runs the query and adds `r_rows`, `r_filtered`, `r_total_time_ms`, `r_loops`. This is the gold standard for "is my optimizer estimate correct" checks.

```sql
-- 10.6+
ANALYZE FORMAT=JSON
  SELECT *
    FROM orders
   WHERE customer_id = 4711
     AND created_at >= '2026-01-01';
```

If `rows: 5000` but `r_rows: 12` : your statistics are wrong. Run `ANALYZE TABLE orders`. If `r_rows` matches `rows` but the plan is still bad : the cost model is choosing wrong, go to `optimizer_trace`.

NEVER substitute `UPDATE` or `DELETE` for `SELECT` in `ANALYZE`. `ANALYZE UPDATE` and `ANALYZE DELETE` execute the mutation.

## Index Hints

Three keywords, three meanings :

```sql
-- 10.6+
-- USE INDEX : optimizer chooses from this list, or table scan if none fit.
SELECT * FROM orders USE INDEX (idx_created_at)
 WHERE created_at >= '2026-01-01';

-- FORCE INDEX : same as USE INDEX but tells the optimizer the table scan is
-- prohibitively expensive. The hint sticks even when stats say "scan is faster".
SELECT * FROM orders FORCE INDEX (idx_created_at)
 WHERE created_at >= '2026-01-01';

-- IGNORE INDEX : exclude a specific index, keep the rest of the cost model.
-- Lowest blast-radius for production : the plan can still adapt as data changes.
SELECT * FROM orders IGNORE INDEX (idx_bad_legacy)
 WHERE created_at >= '2026-01-01';
```

Modifiers steer the hint at a specific clause :

```sql
-- 10.6+
SELECT * FROM orders
 USE INDEX FOR ORDER BY (idx_created_at_desc)
 WHERE customer_id = 4711
 ORDER BY created_at DESC;
```

Default modifier is `FOR JOIN` ; the hint applies to WHERE-clause access. ALWAYS use the smallest hint that solves the problem :

- One bad path : `IGNORE INDEX (that_one)`.
- Want a specific index considered first : `USE INDEX (...)`.
- Cost model is fundamentally wrong (stale stats, complex predicate) : `FORCE INDEX (...)` for diagnosis, then fix the root cause.

## optimizer_switch (MariaDB-Specific)

Toggle individual transformations. Common defaults (verified against KB `optimizer-switch`) :

- **ON by default** : `index_merge`, `index_merge_intersection`, `index_merge_union`, `index_merge_sort_union`, `index_condition_pushdown`, `semijoin`, `materialization`, `loosescan`, `firstmatch`, `subquery_cache`, `derived_merge`, `derived_with_keys`, `condition_pushdown_for_derived`, `condition_pushdown_for_subquery` (10.4+), `condition_pushdown_from_having` (10.4.3+), `rowid_filter` (10.4.3+), `split_materialized` (10.3.4+), `exists_to_in`, `extended_keys`.
- **OFF by default** : `mrr`, `mrr_cost_based`, `mrr_sort_keys`, `index_merge_sort_intersection`, `not_null_range_scan` (10.5+), `hash_join_cardinality` (until 11.0.3).

```sql
-- 10.6+ : disable derived_merge for one session to test materialized plan
SET SESSION optimizer_switch = 'derived_merge=off';

-- Check current state :
SELECT @@SESSION.optimizer_switch;
```

NEVER set every flag to ON globally. The defaults reflect cost-model decisions that have been right more often than wrong in practice. Tuning by flag is a session-scope diagnostic tool, not a production policy.

NEVER copy MySQL 8 optimizer hints (`/*+ NO_BNL(t1) */`, `JSON_TABLE`-specific hints) to MariaDB : flag names and defaults are different. Verify with `SELECT @@optimizer_switch`.

## optimizer_trace (10.4.3+)

When `EXPLAIN` and `ANALYZE FORMAT=JSON` are not enough, dump the optimizer's full decision log.

```sql
-- 10.6+ (feature 10.4.3+)
SET optimizer_trace = 'enabled=on';

SELECT * FROM orders
 WHERE customer_id = 4711
   AND created_at >= '2026-01-01';

SELECT QUERY,
       TRACE,
       MISSING_BYTES_BEYOND_MAX_MEM_SIZE,
       INSUFFICIENT_PRIVILEGES
  FROM INFORMATION_SCHEMA.OPTIMIZER_TRACE;

SET optimizer_trace = 'enabled=off';
```

- `optimizer_trace_max_mem_size` defaults to 1048576 bytes. If `MISSING_BYTES_BEYOND_MAX_MEM_SIZE > 0` the trace was truncated : raise the limit.
- Each connection stores the trace from the last executed statement only.
- Use it to read why the optimizer rejected a particular index (cost = X) or chose a join order.

## Keeping Statistics Fresh

The optimizer's cost model needs accurate row counts and cardinality. ALWAYS run `ANALYZE TABLE` after :

- Bulk inserts of >10% of table size.
- `ALTER TABLE` that rebuilds.
- Restore from backup.
- Large `DELETE` (statistics see fewer rows than the data file).

```sql
-- 10.6+ : engine-internal stats (default, fast)
ANALYZE TABLE orders;

-- 10.6+ : engine-independent persistent stats (richer, slower)
ANALYZE TABLE orders PERSISTENT FOR ALL;
```

`use_stat_tables` controls engine-independent statistics :

- `NEVER` : engine-internal only (default in older versions).
- `COMPLEMENTARY` : use engine-internal if PERSISTENT is missing.
- `PREFERABLY` : use PERSISTENT if collected.

```sql
-- 10.6+ : prefer richer EITS when collected
SET SESSION use_stat_tables = 'PREFERABLY';
```

Histograms (controlled by `histogram_type`, `histogram_size`) further improve selectivity estimates for non-indexed columns. Set `histogram_size=0` to disable, default is engine-dependent.

## Reading the Signal : EXPLAIN Red Flags

| Signal | Meaning | Typical fix |
|--------|---------|-------------|
| `type=ALL` | Full table scan | Add an index that matches the WHERE predicate |
| `type=index` | Full index scan | Add a more selective composite index, or rewrite to use range |
| `Using filesort` | External sort step | Add composite index matching `WHERE + ORDER BY` |
| `Using temporary` | Intermediate temp table for GROUP BY / DISTINCT / UNION | Add covering index or rewrite to avoid materialization |
| `Using join buffer (Block Nested Loop)` | Join without index access | Add index on join column |
| `Range checked for each record` | No usable index decided up-front | Often dynamic predicate ; consider rewriting |
| `DEPENDENT SUBQUERY` | Subquery re-executed per outer row | Rewrite as JOIN or use semijoin transformation |
| `Using index` (in Extra) | Covering index, no row read | GOOD : keep |
| `Using index condition` | ICP active, predicate pushed to engine | GOOD : keep |

## Quality Rules

- ALWAYS run `EXPLAIN` before tuning a query. ALWAYS verify with `ANALYZE FORMAT=JSON` before adding a hint.
- ALWAYS run `ANALYZE TABLE` after bulk data movement, before judging plans.
- NEVER ship `FORCE INDEX` in production code as the primary fix. Use it to diagnose, then fix the root cause (index, stats, query).
- NEVER set `optimizer_switch` flags globally without measuring. Session-scope is the diagnostic tool ; global change is the production decision.
- NEVER assume MySQL 8 optimizer behaviour applies. Flag names and defaults differ ; verify with `@@optimizer_switch`.
- NEVER run `ANALYZE UPDATE` or `ANALYZE DELETE` to "see what it would do" : they execute the mutation.

## Reference Links

- `references/methods.md` : EXPLAIN column reference, optimizer_switch full flag list with version + default, optimizer_trace API, ANALYZE TABLE syntax, statistics variables.
- `references/examples.md` : ten worked examples covering EXPLAIN reading, FORCE INDEX comparison, optimizer_trace dump, ANALYZE FORMAT=JSON estimate-vs-actual, statistics refresh after bulk insert, derived_with_keys flag impact, ICP + covering index.
- `references/anti-patterns.md` : eight real anti-patterns with code and corrected alternatives.

## Sources

- KB `EXPLAIN` : https://mariadb.com/kb/en/explain/
- KB `optimizer-switch` : https://mariadb.com/kb/en/optimizer-switch/
- KB `optimizer-trace-overview` (modern path) : https://mariadb.com/docs/server/ha-and-performance/optimization-and-tuning/query-optimizer/optimizer-trace/optimizer-trace-overview
- KB `ANALYZE` statement : https://mariadb.com/kb/en/analyze-statement/
- KB `ANALYZE TABLE` : https://mariadb.com/kb/en/analyze-table/
- KB index hints : https://mariadb.com/kb/en/index-hints-how-to-force-query-plans/
