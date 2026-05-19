# Methods : The Query-Optimizer Procedure

This file is the deterministic reference for the orchestration procedure. It expands every step of the SKILL.md into precise rules, the signal-to-bottleneck mapping, the index-recommendation rules, and the cross-skill references. It does NOT duplicate EXPLAIN syntax : that lives in `mariadb-impl-query-optimization`.

## The Procedure As Deterministic Rules

### Rule 1 : Input gate

- ALWAYS require two inputs : the SQL query text and its `EXPLAIN` (or `EXPLAIN FORMAT=JSON`) output.
- If the plan is missing, produce the exact `EXPLAIN` statement for the user's query and stop until they return the output.
- NEVER infer the plan from the query text. The optimizer's choice depends on table size, statistics, and `optimizer_switch` state that the query text does not reveal.

```sql
-- 10.6+ : the two statements to hand back when the plan is missing
EXPLAIN <user query>;
EXPLAIN FORMAT=JSON <user query>;
```

### Rule 2 : Signal extraction

Read the plan and record every signal present. Do not stop at the first one : a query can have a missing index AND a filesort.

| Column | Signal | What it indicates |
|--------|--------|-------------------|
| `type` | `ALL` | Full table scan, no index used for access |
| `type` | `index` | Full index scan, cheaper than `ALL` but still scans every index entry |
| `type` | `range`, `ref`, `eq_ref`, `const` | Acceptable access ; not a bottleneck on its own |
| `key` | `NULL` | No index chosen |
| `possible_keys` | `NULL` | The optimizer found no candidate index at all |
| `possible_keys` | non-NULL while `key` is `NULL` | A candidate exists but was rejected : wrong-index or stale-stats case |
| `rows` | large | Optimizer expects to read many rows |
| `filtered` | low (for example 1.00 to 10.00) | Most read rows are discarded : the access path is not selective |
| `Extra` | `Using filesort` | ORDER BY needs an explicit sort step |
| `Extra` | `Using temporary` | GROUP BY / DISTINCT / UNION needs an intermediate temp table |
| `Extra` | `Using join buffer (Block Nested Loop)` | A join runs without index access on the joined table |
| `Extra` | `Using index` | Covering index : GOOD, no row fetch |
| `Extra` | `Using index condition` | Index-condition pushdown active : GOOD |
| `select_type` | `DEPENDENT SUBQUERY` | Subquery re-executed once per outer row |

### Rule 3 : Bottleneck classification

Map the recorded signals to exactly one primary bottleneck class. The order below is the resolution order : if more than one applies, fix the higher entry first, then re-run EXPLAIN.

| Class | Trigger signals | Primary skill to consult |
|-------|-----------------|--------------------------|
| Missing index | `type=ALL` + `key=NULL` + `possible_keys=NULL` | `mariadb-syntax-indexing` |
| Wrong index | `type=ALL`/`index` while `possible_keys` is non-NULL ; or `key` chosen but `key_len` shows trailing composite columns unused | `mariadb-syntax-indexing` |
| Sort problem | `Using filesort` is the dominant cost and access type is otherwise acceptable | `mariadb-syntax-indexing` |
| Join-order problem | `Using join buffer`, or the second joined table shows `type=ALL` | `mariadb-impl-query-optimization` |
| Subquery problem | `select_type=DEPENDENT SUBQUERY`, or a subquery re-executed per outer row | `mariadb-impl-query-optimization` |

### Rule 4 : Fix priority

Always walk this order. Stop at the first option that resolves the classified bottleneck.

1. **Add or fix an index** (see Index-Recommendation Rules below).
2. **Covering-index opportunity** (see Covering-Index Detection below).
3. **Query rewrite** : eliminate `SELECT *`, push predicates into subqueries, rewrite correlated `DEPENDENT SUBQUERY` as a `JOIN`.
4. **`FORCE INDEX`** : last resort only, with the explicit caveat that it overrides the cost model and breaks when data shape changes. Prefer `IGNORE INDEX (bad_one)` when the goal is only to exclude one bad path.

### Rule 5 : Index validation

Before recommending any index, both checks below MUST pass.

- **Selectivity check** : state the assumption in words. Selectivity is approximately distinct-values / total-rows. Equality predicates on high-selectivity columns belong at the front of a composite. A column with few distinct values (boolean, small enum) will usually be ignored by the optimizer ; do not lead a composite with it.
- **Duplicate check** : run `SHOW INDEX FROM <table>` or `SHOW CREATE TABLE <table>`. If the proposed columns form a leftmost prefix of an existing index, the new index is redundant. Recommend extending the existing index instead.

```sql
-- 10.6+ : inspect existing indexes before recommending a new one
SHOW INDEX FROM orders;
SHOW CREATE TABLE orders\G
```

### Rule 6 : Verification

After the user applies the change, have them run `ANALYZE FORMAT=JSON` and read estimate against actual.

```sql
-- 10.6+ : ANALYZE runs the query and annotates the plan with actuals
ANALYZE FORMAT=JSON <user query>;
```

- `r_rows` is the actual rows read ; `rows` is the estimate. `r_filtered` is the actual fraction kept after WHERE.
- Fix confirmed : `r_rows` is small and close to `rows`, `type` improved (for example `ALL` to `ref`).
- Estimate-vs-actual mismatch : statistics are stale. Recommend `ANALYZE TABLE <table>`, then re-verify. Do NOT blame the index.
- ALWAYS `ANALYZE SELECT`. `ANALYZE UPDATE` and `ANALYZE DELETE` execute the mutation : never recommend them for diagnosis.

### Rule 7 : Structured output

Deliver exactly these fields :

```
Bottleneck      : <one class from Rule 3>
Root cause      : <why the optimizer chose the current plan>
Proposed change : <DDL or rewritten SQL>
Selectivity     : <explicit selectivity assumption, or "n/a" for a pure rewrite>
Expected effect : <plan change : type=ALL -> type=ref, filesort removed, etc.>
Verification    : ANALYZE FORMAT=JSON <query>; compare r_rows vs rows
```

## Index-Recommendation Rules

Reference : `mariadb-syntax-indexing` for the complete index DDL and rules.

- Composite column order : equality predicates first, ordered by selectivity (most selective first), then the range predicate, then the `ORDER BY` columns.
- Leftmost-prefix rule : `INDEX(a, b, c)` serves predicates on `(a)`, `(a, b)`, and `(a, b, c)`. It does NOT serve `(b)`, `(c)`, or `(b, c)` alone. A composite in the wrong order is as useless as no index.
- A range predicate stops the index from being used for any column after it. Put the range column last among the predicate columns.
- One equality + one range + one sort : `INDEX(equality_col, range_col)` cannot also serve the `ORDER BY`. If sort matters more, consider `INDEX(equality_col, sort_col)` and accept a range scan filter.
- `key_len` in the plan tells you how many composite columns the optimizer actually used. A short `key_len` on a wide composite means trailing columns are dead weight for this query.

## Covering-Index Detection

A covering index lets the query be answered from the index alone, with no row fetch. The plan shows `Using index` in `Extra`.

- A query is a covering-index candidate when its `SELECT` list and `WHERE` and `ORDER BY` together touch only a small set of columns.
- To make an index covering, append the selected columns that are not already in the index.
- `SELECT *` blocks a covering index : recommend listing only the needed columns first.
- A prefix index (`INDEX(col(N))`) can NEVER be covering : the engine stores only the first N characters, so the row must still be fetched for the full value.

## Duplicate-Index Detection

- An index is redundant when its column list is a leftmost prefix of another index on the same table. `INDEX(a)` is redundant if `INDEX(a, b)` exists.
- A `UNIQUE` constraint already creates an index : a separate non-unique index on the same leading columns is redundant.
- The `PRIMARY KEY` is the clustered index in InnoDB : a secondary index that merely repeats the primary key columns is redundant.
- Every redundant index slows `INSERT`, `UPDATE`, and `DELETE` and wastes disk : ALWAYS recommend removing or merging it.

## Cross-Skill References

These three references are exact skill names. Defer to them for the underlying mechanics.

- `mariadb-impl-query-optimization` : EXPLAIN column-by-column reference, `EXPLAIN FORMAT=JSON`, `ANALYZE FORMAT=JSON`, `ANALYZE TABLE` for statistics, `optimizer_switch` flags, `optimizer_trace`. Consult for plan-reading and plan-verification mechanics.
- `mariadb-syntax-indexing` : `CREATE INDEX` / `ALTER TABLE ADD INDEX` DDL, leftmost-prefix rule, composite-index ordering, covering indexes, prefix indexes, descending indexes (10.8+), IGNORED indexes (10.6+). Consult for every index recommendation.
- `mariadb-errors-slow-queries` : slow query log setup, `log_slow_*` variables (renamed in 10.11+), finding slow queries in production. Consult when the user has not yet identified which query is slow.

## MySQL Divergence Notes

- MariaDB does NOT support MySQL 8 optimizer hint comments (`/*+ NO_BNL(t1) */`, `/*+ JOIN_ORDER(...) */`). NEVER copy them into a MariaDB recommendation. MariaDB steers the optimizer with `optimizer_switch` flags and the SQL index hints `USE INDEX` / `FORCE INDEX` / `IGNORE INDEX`.
- MariaDB index hints accept `FOR JOIN`, `FOR ORDER BY`, `FOR GROUP BY` modifiers ; the default scope is `FOR JOIN`.
- `optimizer_switch` flag names and defaults differ from MySQL 8. Verify with `SELECT @@optimizer_switch` before assuming a flag exists.
