---
name: mariadb-agents-query-optimizer
description: >
  Use when a user provides a slow query plus its EXPLAIN output and wants a concrete optimization, or asks "make this query faster", or wants an index recommendation validated against MariaDB optimizer behavior.
  Prevents the common mistake of suggesting an index without checking selectivity, proposing FORCE INDEX as a fix, copying MySQL 8 optimizer hints, or recommending an index that duplicates an existing one.
  Covers a deterministic query-optimization procedure : read EXPLAIN, identify the bottleneck (type=ALL, filesort, temporary), propose index or rewrite, validate against composite leftmost-prefix rule, check covering-index opportunity, verify with ANALYZE FORMAT=JSON, cross-references mariadb-impl-query-optimization, mariadb-syntax-indexing, mariadb-errors-slow-queries.
  Keywords: optimize my query, make this query faster, query optimization, EXPLAIN analysis, index recommendation, why is this slow, slow query fix, covering index, index suggestion, query rewrite, optimize SQL, performance fix
license: MIT
compatibility: "Designed for Claude Code. Requires MariaDB 10.6-LTS, 10.11-LTS, 11.x, 12.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# MariaDB Query Optimizer

An orchestration skill : a deterministic 7-step procedure Claude applies whenever a user hands over a slow query and wants it made faster. This skill does NOT teach EXPLAIN syntax. It is a checklist that drives the analysis and references three companion skills for the underlying mechanics.

## When To Use This Skill

- The user pastes a SQL query and says it is slow, takes forever, or times out.
- The user pastes a query plus `EXPLAIN` output and asks for an optimization.
- The user asks "should I add an index here" or "which index do I need".
- The user proposes an index or `FORCE INDEX` and asks whether it is correct.

## Quick Reference

- Require the query plus its `EXPLAIN` (or `EXPLAIN FORMAT=JSON`) before optimizing. If the user did not provide it, produce the exact `EXPLAIN` statement and ask them to run it.
- Bottleneck classes : missing index / wrong index / sort problem / join-order problem / subquery problem. Classify into exactly one before proposing a fix.
- Fix priority order : add or fix an index > covering-index opportunity > query rewrite > `FORCE INDEX` (last resort, with explicit caveat).
- ALWAYS state the selectivity assumption behind any index recommendation. Selectivity = approximate distinct-values / total-rows. An index on a low-selectivity column will not be chosen.
- ALWAYS verify the proposed index does not duplicate an existing one. A new index whose columns are a leftmost prefix of an existing index is redundant.
- Verify the fix with `ANALYZE FORMAT=JSON` : compare `r_rows` (actual) against `rows` (estimate). A large mismatch means stale statistics, not a bad index.
- NEVER recommend `FORCE INDEX` as a permanent production fix. NEVER copy MySQL 8 optimizer hints (`/*+ ... */`). NEVER suggest an index without stating its selectivity assumption.

## The 7-Step Procedure

Apply these steps in order. Do not skip a step. Each step names the companion skill that holds the detailed mechanics.

### Step 1 : Require the query plus EXPLAIN

If the user provided both the query and its `EXPLAIN` output, continue to Step 2.

If the user provided only the query, do NOT guess the plan. Produce the exact statement and ask them to run it :

```sql
-- 10.6+ : ask the user to run this and paste the output
EXPLAIN <their query>;

-- 10.6+ : richer structured plan, preferred when the user can run it
EXPLAIN FORMAT=JSON <their query>;
```

NEVER analyze a query without a plan. The plan is the input ; an optimization without it is a guess.

### Step 2 : Read the EXPLAIN signals

Reference : `mariadb-impl-query-optimization` for full column-by-column EXPLAIN reading.

Scan the plan for these signals, in this order :

| Signal | Where | Meaning |
|--------|-------|---------|
| `type=ALL` | type column | Full table scan : no index used for access |
| `key=NULL` | key column | Optimizer found no usable index |
| `Using filesort` | Extra column | ORDER BY cannot be served from an index |
| `Using temporary` | Extra column | GROUP BY / DISTINCT / UNION needs a temp table |
| `Using join buffer` | Extra column | A join runs without index access on the joined table |
| large `rows` * low `filtered` | rows + filtered | Optimizer reads many rows, keeps few : missing selective index |
| `DEPENDENT SUBQUERY` | select_type | Subquery re-executed once per outer row |

### Step 3 : Classify the bottleneck

Map the signals to exactly one bottleneck class :

| Signals observed | Bottleneck class |
|------------------|------------------|
| `type=ALL` + `key=NULL`, no usable index exists | Missing index |
| `type=ALL` or `type=index` but a usable index exists in `possible_keys` | Wrong index (optimizer rejected it, or order is wrong) |
| `Using filesort` as the dominant cost | Sort problem |
| `Using join buffer`, or join `type=ALL` on the second table | Join-order problem |
| `DEPENDENT SUBQUERY`, or `select_type` shows a re-executed subquery | Subquery problem |

If two classes apply, fix the access-path class first (missing or wrong index), then re-run EXPLAIN before touching the rest.

### Step 4 : Propose a fix in priority order

Reference : `mariadb-syntax-indexing` for index DDL, leftmost-prefix rule, covering indexes, prefix indexes.

Walk the priority order. Stop at the first option that resolves the bottleneck.

1. **Add or fix an index.** Build a composite index whose leading columns are the equality predicates ordered by selectivity (most selective first), followed by the range predicate, followed by the `ORDER BY` columns. The leftmost-prefix rule means `INDEX(a, b, c)` serves predicates on `(a)`, `(a, b)`, `(a, b, c)` only.
2. **Covering-index opportunity.** If the query selects only a few columns, extend the index to include every column the query touches. A covering index produces `Using index` in `Extra` and skips the row fetch entirely. A prefix index can NEVER be covering.
3. **Query rewrite.** Eliminate `SELECT *` so a covering index becomes possible. Push WHERE predicates from an outer query into the subquery. Rewrite a correlated `DEPENDENT SUBQUERY` as a `JOIN` so the optimizer can pick a join order.
4. **`FORCE INDEX` (last resort).** Only when statistics are demonstrably wrong and cannot be fixed. ALWAYS attach the caveat : it overrides the cost model and locks the plan, so it breaks when data shape changes. Prefer `IGNORE INDEX (bad_one)` for the smaller blast radius.

### Step 5 : Validate the proposed index

Before recommending an index, run two checks :

- **Selectivity check.** State the assumption explicitly : "This index helps only if `customer_id` has high selectivity, roughly distinct-values / total-rows close to 1." If the column is low-selectivity (a boolean, a status with 3 values), say the optimizer will likely ignore the index and propose a different column or a composite.
- **Duplicate check.** Run `SHOW INDEX FROM <table>` (or `SHOW CREATE TABLE`). If an existing index already has the proposed columns as a leftmost prefix, the new index is redundant : recommend extending the existing index instead. Redundant indexes slow every INSERT, UPDATE, and DELETE.

### Step 6 : Verify with ANALYZE FORMAT=JSON

Reference : `mariadb-impl-query-optimization` for the full `ANALYZE` API.

After the user applies the change, have them run `ANALYZE FORMAT=JSON` and compare estimate against actual :

```sql
-- 10.6+ : runs the query, reports estimate (rows) next to actual (r_rows)
ANALYZE FORMAT=JSON <their query>;
```

- If `r_rows` is now small and close to `rows` : the fix worked.
- If `rows` and `r_rows` diverge wildly : statistics are stale. Recommend `ANALYZE TABLE <table>`, then re-check. The index is not the problem.
- ALWAYS use `ANALYZE SELECT`. NEVER tell a user to run `ANALYZE UPDATE` or `ANALYZE DELETE` to "see the plan" : both execute the mutation.

### Step 7 : Output a structured recommendation

Deliver the result in this exact shape so the user can act on it :

```
Bottleneck      : <one class from Step 3>
Root cause      : <why the optimizer chose this plan>
Proposed change : <the DDL or rewritten SQL>
Selectivity     : <the explicit selectivity assumption>
Expected effect : <type=ALL becomes type=ref, filesort removed, etc.>
Verification    : ANALYZE FORMAT=JSON <query>; compare r_rows vs rows
```

## Decision Tree : Picking The Fix

```
type=ALL and key=NULL ?
  Yes : is there a usable index in possible_keys ?
    No  -> Missing index. Build a composite (Step 4.1).
    Yes -> Wrong index. Check column order vs leftmost-prefix, or stale stats.
  No  : Using filesort dominates ?
    Yes -> Sort problem. Composite index ending in the ORDER BY columns.
  No  : Using join buffer or join type=ALL ?
    Yes -> Join-order problem. Index the join column on the second table.
  No  : DEPENDENT SUBQUERY ?
    Yes -> Subquery problem. Rewrite the correlated subquery as a JOIN.
```

## Cross-Skill Validation Rules

This skill is consistent with its three companions. When the analysis touches their domain, defer to them :

- `mariadb-impl-query-optimization` : EXPLAIN column reference, `EXPLAIN FORMAT=JSON`, `ANALYZE FORMAT=JSON`, `optimizer_switch`, `optimizer_trace`, `ANALYZE TABLE` for statistics. Use it for the mechanics of reading and verifying plans.
- `mariadb-syntax-indexing` : `CREATE INDEX` DDL, the leftmost-prefix rule, composite-index column ordering, covering indexes, prefix indexes, descending indexes. Use it for every index recommendation.
- `mariadb-errors-slow-queries` : slow query log configuration, finding slow queries in production, `log_slow_*` variables. Use it when the user has not yet identified which query is slow.

## Quality Rules

- ALWAYS require the query plus its `EXPLAIN` before proposing any optimization. Produce the `EXPLAIN` statement if missing.
- ALWAYS classify the bottleneck into exactly one class before proposing a fix.
- ALWAYS follow the fix priority order : index > covering index > rewrite > `FORCE INDEX`.
- ALWAYS state the explicit selectivity assumption behind an index recommendation.
- ALWAYS check for a duplicate or redundant index before recommending a new one.
- ALWAYS verify the fix with `ANALYZE FORMAT=JSON`, comparing `r_rows` against `rows`.
- NEVER recommend `FORCE INDEX` as a permanent production fix. It is a diagnostic, with an explicit caveat.
- NEVER copy MySQL 8 optimizer hints (`/*+ NO_BNL(t) */` and similar). MariaDB uses `optimizer_switch` flags and SQL index hints, not the MySQL 8 hint comment syntax.
- NEVER suggest an index without stating its selectivity assumption.
- NEVER analyze a query without its plan : the plan is the required input.

## Reference Links

- `references/methods.md` : the full 7-step procedure as deterministic rules, the EXPLAIN-signal-to-bottleneck mapping table, index-recommendation rules with the leftmost-prefix rule, covering-index detection, duplicate-index detection, and the exact cross-skill references.
- `references/examples.md` : eight worked optimizations covering a `type=ALL` query, a `Using filesort` fix, a correlated-subquery rewrite, a `Using temporary` elimination, a covering-index recommendation, duplicate-index avoidance, a join-order fix, and a full slow-JOIN review.
- `references/anti-patterns.md` : six real anti-patterns with code, why each fails, and the correct alternative.

## Sources

- KB `EXPLAIN` : https://mariadb.com/kb/en/explain/
- KB `ANALYZE` statement : https://mariadb.com/kb/en/analyze-statement/
- KB index hints : https://mariadb.com/kb/en/index-hints-how-to-force-query-plans/
- KB `optimizer-switch` : https://mariadb.com/kb/en/optimizer-switch/
- Companion skill `mariadb-impl-query-optimization`
- Companion skill `mariadb-syntax-indexing`
- Companion skill `mariadb-errors-slow-queries`
