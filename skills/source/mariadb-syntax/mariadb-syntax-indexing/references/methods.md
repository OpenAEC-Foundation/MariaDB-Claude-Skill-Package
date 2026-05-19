# MariaDB Indexing : Methods Reference

Complete grammar, system variables, and inspection queries for MariaDB indexes. Verified against KB `create-index/`, `alter-table/`, `show-index/`, `full-text-index-overview/`, `ignored-indexes/`, `generated-columns/`, `storage-engine-index-types/` on 2026-05-19.

## 1. CREATE INDEX grammar (canonical, KB-verified)

```text
CREATE [OR REPLACE] [UNIQUE | FULLTEXT | SPATIAL | VECTOR] INDEX
  [IF NOT EXISTS] index_name
    [index_type]
    ON tbl_name (index_col_name, ...)
    [WAIT n | NOWAIT]
    [index_option]
    [algorithm_option | lock_option] ...

index_col_name :
    col_name [(length)] [ASC | DESC]

index_type :
    USING {BTREE | HASH | RTREE}

index_option :
    KEY_BLOCK_SIZE [=] value
  | index_type
  | WITH PARSER parser_name
  | COMMENT 'string'
  | CLUSTERING = {YES | NO}
  | IGNORED | NOT IGNORED
  | DISTANCE = {EUCLIDEAN | COSINE}
  | M = number

algorithm_option :
    ALGORITHM [=] {DEFAULT | INPLACE | COPY | NOCOPY | INSTANT}

lock_option :
    LOCK [=] {DEFAULT | NONE | SHARED | EXCLUSIVE}
```

Notes :

- `OR REPLACE` (10.1.4+) atomically drops + recreates ; do NOT mix with `IF NOT EXISTS`.
- `UNIQUE`, `FULLTEXT`, `SPATIAL`, `VECTOR` are mutually exclusive. `VECTOR` is 11.7+ (preview) and out of scope for this skill.
- `length` is in CHARACTERS for non-binary columns, BYTES for binary types. Mandatory for `TEXT` and `BLOB` columns.
- `ASC | DESC` per column : `DESC` only honoured from 10.8+. Older versions parse and silently store as ASC.
- `IGNORED | NOT IGNORED` : 10.6+. Optimizer-hidden but still maintained.
- `USING HASH` : MEMORY engine only for user tables. InnoDB has internal adaptive hash but does NOT expose explicit HASH user indexes.
- `USING RTREE` : InnoDB, MyISAM, Aria when the indexed columns are spatial.
- `ALGORITHM = INSTANT` : not supported for index creation. Use `INPLACE`.

## 2. ALTER TABLE index operations

```sql
-- Add index :
ALTER TABLE t ADD INDEX ix_a (a);
ALTER TABLE t ADD UNIQUE INDEX uk_b (b);
ALTER TABLE t ADD FULLTEXT INDEX ft_body (body);
ALTER TABLE t ADD SPATIAL INDEX sx_geom (geom);
ALTER TABLE t ADD PRIMARY KEY (id);

-- Composite + prefix + descending :
ALTER TABLE t ADD INDEX ix_multi (a, b(50), c DESC);   -- DESC honoured 10.8+

-- Drop index :
ALTER TABLE t DROP INDEX ix_a;
ALTER TABLE t DROP PRIMARY KEY;

-- Rename index (10.5.2+) :
ALTER TABLE t RENAME INDEX ix_old TO ix_new;

-- Toggle IGNORED (10.6+) :
ALTER TABLE t ALTER INDEX ix_a IGNORED;
ALTER TABLE t ALTER INDEX ix_a NOT IGNORED;

-- Online DDL options (10.6+) :
ALTER TABLE t ADD INDEX ix_a (a), ALGORITHM = INPLACE, LOCK = NONE;
```

## 3. CREATE TABLE index clauses

```sql
CREATE TABLE example (
  id BIGINT UNSIGNED AUTO_INCREMENT,
  email VARCHAR(255) NOT NULL,
  status VARCHAR(32) NOT NULL,
  created_at DATETIME(6) NOT NULL,
  body MEDIUMTEXT NOT NULL,
  geom POINT NOT NULL,
  payload JSON NOT NULL CHECK (JSON_VALID(payload)),

  PRIMARY KEY (id),
  UNIQUE KEY uk_email (email),
  INDEX ix_status_created (status, created_at),
  INDEX ix_body_prefix (body(100)),
  FULLTEXT KEY ft_body (body),
  SPATIAL KEY sx_geom (geom)
) ENGINE=InnoDB
  DEFAULT CHARSET=utf8mb4
  COLLATE=utf8mb4_uca1400_ai_ci
  ROW_FORMAT=DYNAMIC;
```

Notes :

- `KEY` and `INDEX` are interchangeable inside `CREATE TABLE`. Both work for normal, UNIQUE, FULLTEXT, SPATIAL.
- Primary key is implicitly UNIQUE and NOT NULL on every column.
- A table has exactly one PRIMARY KEY. Use `PRIMARY KEY` clause ; do NOT scatter PK across columns with multiple `PRIMARY KEY` clauses.

## 4. DROP INDEX standalone form

```sql
-- Equivalent to ALTER TABLE t DROP INDEX ix.
DROP INDEX ix_status_created ON orders;

-- Online DDL :
DROP INDEX ix_status_created ON orders ALGORITHM = INPLACE, LOCK = NONE;
```

## 5. Index types per storage engine (KB-verified table)

| Engine  | BTREE | HASH | FULLTEXT | RTREE (SPATIAL) |
|---------|-------|------|----------|-----------------|
| InnoDB  | YES   | NO (user-level) | YES (since 10.0.5) | YES |
| MyISAM  | YES   | NO   | YES      | YES             |
| Aria    | YES   | NO   | YES      | YES             |
| MEMORY  | YES   | YES (default) | NO | NO              |
| ARCHIVE | NO    | NO   | NO       | NO              |

Notes :

- InnoDB has an internal "adaptive hash index" controlled by `innodb_adaptive_hash_index` (default ON). This is NOT a user-defined index ; it is a per-page cache layer. You cannot `CREATE INDEX ... USING HASH` on InnoDB.
- `MEMORY` engine defaults to HASH ; specify `USING BTREE` if you need range scans.
- `ARCHIVE` engine has no indexes at all (single AUTO_INCREMENT key only).

## 6. FULLTEXT system variables

| Variable                             | Default | Engine     | Effect                                              |
|--------------------------------------|---------|------------|-----------------------------------------------------|
| `innodb_ft_min_token_size`           | 3       | InnoDB     | Minimum word length stored in the full-text index   |
| `innodb_ft_max_token_size`           | 84      | InnoDB     | Maximum word length stored                          |
| `innodb_ft_enable_stopword`          | ON      | InnoDB     | Apply stopword list during indexing                 |
| `innodb_ft_server_stopword_table`    | NULL    | InnoDB     | `db/table` of custom server-wide stopword table     |
| `innodb_ft_user_stopword_table`      | NULL    | InnoDB     | Per-session override                                |
| `innodb_ft_cache_size`               | 8M      | InnoDB     | Per-table FULLTEXT cache before flush to disk       |
| `ft_min_word_len`                    | 4       | MyISAM     | Minimum word length stored                          |
| `ft_max_word_len`                    | 84      | MyISAM     | Maximum word length stored                          |
| `ft_stopword_file`                   | built-in| MyISAM     | Path to stopword file, empty string = no stopwords  |
| `ft_query_expansion_limit`           | 20      | MyISAM     | Rows used in WITH QUERY EXPANSION                   |
| `ft_boolean_syntax`                  | `+ -><()~*:""&|` | MyISAM | Boolean-mode operator set                        |

After changing `innodb_ft_min_token_size`, you MUST rebuild every FULLTEXT index : `OPTIMIZE TABLE t;` or drop and recreate. Existing tokens below the new threshold are NOT removed automatically.

## 7. MATCH ... AGAINST grammar

```sql
MATCH (col1[, col2, ...]) AGAINST (expr [search_modifier])

search_modifier :
    IN NATURAL LANGUAGE MODE
  | IN NATURAL LANGUAGE MODE WITH QUERY EXPANSION
  | IN BOOLEAN MODE
  | WITH QUERY EXPANSION
```

Defaults to `IN NATURAL LANGUAGE MODE`. Boolean-mode operators :

| Operator | Meaning                                                   |
|----------|-----------------------------------------------------------|
| `+word`  | Word MUST be present                                       |
| `-word`  | Word MUST NOT be present                                   |
| `>word`  | Increase the word's contribution to relevance              |
| `<word`  | Decrease the word's contribution                           |
| `~word`  | Negative relevance contribution (without excluding)        |
| `*`      | Wildcard ; only valid at end of word (`mysql*`)            |
| `"..."`  | Exact phrase                                               |
| `()`     | Subexpression grouping                                     |

Natural-language mode does NOT support these operators ; it ranks by TF/IDF instead.

## 8. SPATIAL INDEX details

- Requires `NOT NULL` on the indexed geometry column.
- Available index function is R-tree.
- Supported geometry types : `POINT`, `LINESTRING`, `POLYGON`, `MULTIPOINT`, `MULTILINESTRING`, `MULTIPOLYGON`, `GEOMETRYCOLLECTION`, `GEOMETRY` (abstract base type).
- Spatial predicates that USE the R-tree index : `MBRContains`, `MBRWithin`, `MBRIntersects`, `MBROverlaps`, `MBRTouches`, `MBRDisjoint`, `MBREquals`. The non-MBR variants (`ST_Contains`, `ST_Within`, ...) re-check on the actual geometry but only after R-tree pre-filtering.
- Always build geometries through `ST_GeomFromText`, `ST_GeomFromWKB`, `Point(x, y)`, `LineString(...)`, etc. Do NOT cast strings directly.

## 9. Generated-column functional index (mandatory for JSON)

```sql
-- VIRTUAL : computed on read, no storage cost, index is still materialised.
ALTER TABLE t
  ADD COLUMN ext_key VARCHAR(64) AS (JSON_VALUE(doc, '$.key')) VIRTUAL,
  ADD INDEX ix_ext_key (ext_key);

-- PERSISTENT (also accepted spelling : STORED) : materialised on disk.
-- Use when the expression is expensive and read many times.
ALTER TABLE t
  ADD COLUMN ext_key VARCHAR(64) AS (JSON_VALUE(doc, '$.key')) PERSISTENT,
  ADD INDEX ix_ext_key (ext_key);
```

Restrictions :

- The generated expression must be DETERMINISTIC.
- VIRTUAL generated columns CAN be indexed since 10.5+.
- A generated column cannot reference another non-persistent generated column or a column with a default that is non-deterministic.

## 10. Inspecting indexes

```sql
-- All indexes on a table :
SHOW INDEX FROM orders;
-- Columns : Table, Non_unique, Key_name, Seq_in_index, Column_name,
--          Collation (A=asc, D=desc, NULL=unsorted), Cardinality,
--          Sub_part (prefix length), Packed, Null, Index_type,
--          Comment, Index_comment, Ignored.

-- All indexes in INFORMATION_SCHEMA :
SELECT INDEX_NAME, COLUMN_NAME, SEQ_IN_INDEX, NON_UNIQUE,
       INDEX_TYPE, IGNORED
FROM INFORMATION_SCHEMA.STATISTICS
WHERE TABLE_SCHEMA = DATABASE() AND TABLE_NAME = 'orders'
ORDER BY INDEX_NAME, SEQ_IN_INDEX;

-- Storage size per index (InnoDB) :
SELECT NAME, sum(ALLOCATED_SIZE) AS bytes
FROM INFORMATION_SCHEMA.INNODB_SYS_TABLESPACES
WHERE NAME LIKE 'mydb/%' GROUP BY NAME;
```

## 11. Statistics maintenance

```sql
-- Refresh cardinality estimates after large data shifts.
ANALYZE TABLE orders;

-- Rebuild indexes (defragment + recalc) :
OPTIMIZE TABLE orders;        -- locks the table on most engines
ALTER TABLE orders FORCE;     -- alternative, sometimes lighter
```

Without `ANALYZE TABLE`, cardinality estimates may be stale after bulk inserts. The optimizer then picks worse plans, and `FORCE INDEX` "fixes" can become wrong fixes. ALWAYS run `ANALYZE TABLE` first.

## 12. Index hints (KB `index-hints-how-to-force-query-plans/`)

```sql
SELECT ... FROM t USE INDEX (ix_a, ix_b) WHERE ...;
SELECT ... FROM t FORCE INDEX (ix_a) WHERE ...;
SELECT ... FROM t IGNORE INDEX (ix_a) WHERE ...;

-- Scoped variants :
SELECT ... FROM t USE INDEX FOR JOIN (ix_a) ...;
SELECT ... FROM t USE INDEX FOR ORDER BY (ix_a) ...;
SELECT ... FROM t USE INDEX FOR GROUP BY (ix_a) ...;
```

Hints are a hammer ; prefer `ANALYZE TABLE` + correct composite design. `FORCE INDEX` locks the plan to the named index even when statistics shift and a better plan emerges later.
