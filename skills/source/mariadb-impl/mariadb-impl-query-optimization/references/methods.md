# Methods : Query Optimization Reference

Complete API surface for EXPLAIN, ANALYZE, index hints, optimizer_switch, optimizer_trace, and statistics.

## 1. EXPLAIN

```sql
-- 10.6+
EXPLAIN [EXTENDED | PARTITIONS | FORMAT=JSON] <explainable_statement>
```

`<explainable_statement>` : `SELECT`, `UPDATE`, `DELETE`, `INSERT ... SELECT`, `REPLACE`.

### Tabular output columns

| Column | Type | Notes |
|--------|------|-------|
| `id` | int | Statement / subquery sequence. Higher = deeper subquery. NULL for `UNION RESULT`. |
| `select_type` | string | See full list below. |
| `table` | string | Table or `<derivedN>` for inline derived tables. |
| `type` | string | Access method. See ranking below. |
| `possible_keys` | string | Comma-separated list of indexes considered. NULL = none usable. |
| `key` | string | Chosen index. NULL = full table scan. |
| `key_len` | int | Bytes of the chosen index used. Short value on composite key = trailing columns unused. |
| `ref` | string | Compared-against value : column name (join), `const`, `func`. |
| `rows` | int | Estimated rows the optimizer expects to read. |
| `Extra` | string | Notes : see full list below. |

`EXPLAIN EXTENDED` adds one more column :

| Column | Notes |
|--------|-------|
| `filtered` | Percentage of `rows` left after WHERE evaluation. |

### `select_type` values

`SIMPLE`, `PRIMARY`, `SUBQUERY`, `DEPENDENT SUBQUERY`, `UNION`, `DEPENDENT UNION`, `UNION RESULT`, `DERIVED`, `MATERIALIZED`, `UNCACHEABLE SUBQUERY`, `UNCACHEABLE UNION`, `LATERAL DERIVED`.

`DEPENDENT SUBQUERY` is the costliest : the subquery is re-executed once per outer row. Rewrite as JOIN or rely on semijoin transformation (`optimizer_switch=semijoin=on`).

### `type` ranking (best to worst)

1. `system` : table has 0 or 1 rows (special case of `const`).
2. `const` : at most one matching row via PRIMARY or UNIQUE.
3. `eq_ref` : index lookup, one row per join key.
4. `ref` : non-unique index lookup.
5. `ref_or_null` : `ref` plus a search for NULL.
6. `index_subquery` : subquery via key lookups.
7. `unique_subquery` : subquery via unique key lookups.
8. `index_merge` : multiple index ranges combined.
9. `range` : range of values via index.
10. `index` : full index scan (not a range).
11. `ALL` : full table scan (worst).

The boundary "fine vs fix it now" sits at `range`. `index` and `ALL` need attention on any non-trivial table.

### `Extra` values (selected)

| Value | Meaning |
|-------|---------|
| `Using where` | Predicate applied after row fetch. |
| `Using index` | Covering index : no row read. Good. |
| `Using index condition` | ICP : predicate pushed to engine. Good. |
| `Using filesort` | External sort step. Add composite index for ORDER BY. |
| `Using temporary` | Intermediate temp table (GROUP BY / DISTINCT / UNION). |
| `Using join buffer (Block Nested Loop)` | Join without indexed access. |
| `Using join buffer (Batched Key Access)` | BKA join. Enabled by `optimizer_switch=join_cache_bka=on`. |
| `Range checked for each record` | No usable index decided up-front, re-checked per outer row. |
| `Not exists` | LEFT JOIN short-circuit. |
| `Select tables optimized away` | Optimizer answered without reading data (e.g. `COUNT(*)` with PK on InnoDB). |
| `Full scan on NULL key` | Subquery scan triggered by NULL value. |
| `Impossible WHERE` | Predicate is always false ; no rows returned. |
| `Distinct` | Stops scanning after one matching row per group. |
| `Using index for group-by` | Loose-index scan for GROUP BY. |

## 2. EXPLAIN FORMAT=JSON

```sql
-- 10.6+
EXPLAIN FORMAT=JSON <statement>;
```

Returns a single JSON tree, one node per plan step. Key fields per node :

- `access_type` : same vocabulary as the tabular `type` column.
- `key`, `key_length`, `used_key_parts` : index info.
- `rows`, `filtered` : optimizer estimates.
- `read_cost`, `eval_cost` : cost-model numbers.
- `attached_condition` : predicate evaluated at this step.
- `materialized` (nested) : subquery / derived table materialization details.

JSON is the format to use when reading multi-table joins with subqueries : the nested structure is clearer than the flat tabular form.

## 3. ANALYZE statement (run-and-measure)

```sql
-- 10.1+
ANALYZE [FORMAT=JSON] <explainable_statement>;
```

Executes the statement and adds observed values :

| Column / JSON field | Meaning |
|---------------------|---------|
| `r_rows` | Rows actually read (vs `rows` estimate). |
| `r_filtered` | Actual fraction left after WHERE (vs `filtered` estimate). |
| `r_total_time_ms` | Wall time at this step (FORMAT=JSON only). |
| `r_loops` | Times this step ran (FORMAT=JSON only). |

ALWAYS use `ANALYZE SELECT`. `ANALYZE UPDATE` and `ANALYZE DELETE` execute the mutation.

## 4. Index Hints

```sql
-- 10.6+
SELECT ...
  FROM <table> [{USE|FORCE|IGNORE} INDEX [{FOR {JOIN|ORDER BY|GROUP BY}}] (idx [, idx ...])]
 WHERE ...
```

- Default modifier when `FOR` is omitted : `FOR JOIN` (affects WHERE-clause access).
- Index names can be unambiguous prefixes.
- `USE INDEX ()` (empty list) is illegal ; use `IGNORE INDEX (...)` to exclude indexes.
- Multiple hints can stack on the same table : `USE INDEX FOR ORDER BY (a) USE INDEX FOR JOIN (b)`.

### Semantic differences

| Hint | Effect on optimizer |
|------|---------------------|
| `USE INDEX (a, b)` | Consider only indexes a and b. May still pick full table scan if cheaper. |
| `FORCE INDEX (a, b)` | Consider only a and b. Treats table scan as prohibitively expensive ; only falls back if hinted indexes are unusable. |
| `IGNORE INDEX (a)` | Exclude a. Keep the rest of the cost model. |

## 5. optimizer_switch Flags

```sql
-- 10.6+
SET [SESSION | GLOBAL] optimizer_switch = '<flag>=on,<flag>=off,...';
SELECT @@SESSION.optimizer_switch;   -- inspect current value
```

### Defaults ON (verified against KB optimizer-switch)

`derived_merge` (5.3+), `derived_with_keys` (5.3+), `index_merge` (5.1+), `index_merge_union` (5.1+), `index_merge_sort_union` (5.1+), `index_merge_intersection` (5.1+), `index_condition_pushdown` (5.3+), `semijoin` (5.3+), `materialization` (5.3+), `loosescan` (5.3+), `firstmatch` (5.3+), `subquery_cache` (5.3+), `exists_to_in` (10.0+), `in_to_exists` (5.3+), `condition_pushdown_for_derived` (10.2.2+), `condition_pushdown_for_subquery` (10.4+), `condition_pushdown_from_having` (10.4.3+), `rowid_filter` (10.4.3+), `split_materialized` (10.3.4+), `extended_keys` (5.5.21+), `outer_join_with_cache` (5.3+), `semijoin_with_cache` (5.3+), `join_cache_bka` (5.3+), `join_cache_hashed` (5.3+), `join_cache_incremental` (5.3+), `optimize_join_buffer_size` (5.3+), `orderby_uses_equalities` (10.1.15+), `partial_match_rowid_merge` (5.3+), `partial_match_table_scan` (5.3+), `table_elimination` (5.1+), `sargable_casefold` (11.3.1+).

### Defaults OFF

`mrr` (5.3+), `mrr_cost_based` (5.3+), `mrr_sort_keys` (5.3+), `index_merge_sort_intersection` (5.3+), `not_null_range_scan` (10.5+), `hash_join_cardinality` (10.6.13 to 11.0.3 ; default ON from 11.0.2+ onward).

### Default-changing flags

| Flag | Note |
|------|------|
| `cset_narrowing` | Added 10.6.16, default ON from 11.7+. |
| `duplicateweedout` | Default ON from 12.0+. |
| `reorder_outer_joins` | Default ON from 12.3+. |

### Removed / deprecated

`engine_condition_pushdown` : deprecated 10.1, removed in 11.3.

## 6. optimizer_trace (10.4.3+)

```sql
-- 10.4.3+
SET optimizer_trace = 'enabled=on';
SET optimizer_trace_max_mem_size = 16777216;   -- raise from 1 MB default if truncated

<run the query under investigation>

SELECT QUERY,
       TRACE,
       MISSING_BYTES_BEYOND_MAX_MEM_SIZE,
       INSUFFICIENT_PRIVILEGES
  FROM INFORMATION_SCHEMA.OPTIMIZER_TRACE;

SET optimizer_trace = 'enabled=off';
```

### `INFORMATION_SCHEMA.OPTIMIZER_TRACE` columns

| Column | Meaning |
|--------|---------|
| `QUERY` | The text of the traced statement. |
| `TRACE` | JSON document with cost calculations, considered plans, rejected indexes, transformations. |
| `MISSING_BYTES_BEYOND_MAX_MEM_SIZE` | > 0 means TRACE was truncated ; raise `optimizer_trace_max_mem_size`. |
| `INSUFFICIENT_PRIVILEGES` | 1 = trace omitted parts due to SQL SECURITY DEFINER. |

### Behaviour rules

- Each connection stores ONE trace : the one from the last executed statement.
- Tracing only applies to `SELECT`, `UPDATE`, `DELETE`, `INSERT`.
- Tracing is per-session ; setting GLOBAL persists across new connections.

## 7. ANALYZE TABLE (statistics)

```sql
-- 10.6+
ANALYZE [NO_WRITE_TO_BINLOG | LOCAL] TABLE tbl [, tbl ...]
  [PERSISTENT FOR { ALL
                  | COLUMNS ([c [, c ...]]) INDEXES ([i [, i ...]])
                  }];
```

- Without `PERSISTENT` : updates engine-internal statistics (InnoDB, Aria, MyISAM). Fast.
- With `PERSISTENT` : collects Engine-Independent Table Statistics (EITS) into `mysql.column_stats`, `mysql.table_stats`, `mysql.index_stats`. Slower, richer, NOT automatically refreshed.
- ALWAYS run after bulk insert > 10% of table size, after `ALTER TABLE` that rebuilds, after restore-from-backup, after large `DELETE`.

### Statistics-related variables

| Variable | Default | Effect |
|----------|---------|--------|
| `use_stat_tables` | `PREFERABLY_FOR_QUERIES` (10.6+) | `NEVER`, `COMPLEMENTARY`, `PREFERABLY` ; controls when EITS is consulted. |
| `histogram_type` | `DOUBLE_PREC_HB` | `SINGLE_PREC_HB`, `DOUBLE_PREC_HB`, `JSON_HB` (10.6+). |
| `histogram_size` | engine-dependent | Number of buckets ; 0 disables histograms. |
| `innodb_stats_persistent` | ON | InnoDB persistent stats stored in `mysql.innodb_table_stats`, `mysql.innodb_index_stats`. |
| `innodb_stats_auto_recalc` | ON | InnoDB recalculates after 10% changes. |
| `innodb_stats_persistent_sample_pages` | 20 | Pages sampled per index when recalculating. |

## 8. performance_schema Integration

The Performance Schema exposes query-level metrics complementary to EXPLAIN :

- `events_statements_summary_by_digest` : per-statement-shape aggregated stats (timer, rows, errors).
- `events_statements_history_long` : ring buffer of recent statements with timings.

```sql
-- 10.6+
SELECT DIGEST_TEXT,
       COUNT_STAR,
       AVG_TIMER_WAIT / 1e9 AS avg_ms,
       SUM_ROWS_EXAMINED
  FROM performance_schema.events_statements_summary_by_digest
 ORDER BY SUM_TIMER_WAIT DESC
 LIMIT 10;
```

Enable instruments via `performance_schema=ON` in `[mysqld]`. Use this surface to find the candidate queries to optimize ; use EXPLAIN to understand each one.

## 9. Slow Query Log

Complementary to EXPLAIN for production : capture queries above a threshold.

```sql
-- 10.6+
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 0.5;        -- seconds
SET GLOBAL log_queries_not_using_indexes = ON;
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';
```

Pair with `pt-query-digest` (Percona Toolkit) or `mariadb-dump-slow` to aggregate.

## 10. Variable Cheat-Sheet

| Variable | Purpose |
|----------|---------|
| `optimizer_switch` | Enable / disable individual optimizer transformations. |
| `optimizer_trace` | `'enabled=on'` to start tracing. |
| `optimizer_trace_max_mem_size` | Bytes of trace storage per connection (default 1048576). |
| `optimizer_use_condition_selectivity` | 1-5 ; how many sources the optimizer uses for selectivity (default 4 in 10.6+). |
| `use_stat_tables` | When to consult EITS. |
| `histogram_type`, `histogram_size` | Histogram-shaped stats. |
| `innodb_stats_persistent`, `innodb_stats_auto_recalc` | InnoDB stats lifecycle. |
| `slow_query_log`, `long_query_time` | Slow query capture. |
