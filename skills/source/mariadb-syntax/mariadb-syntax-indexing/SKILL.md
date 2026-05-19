---
name: mariadb-syntax-indexing
description: >
  Use when adding indexes, choosing an index type, debugging slow WHERE / ORDER BY / JOIN, indexing JSON paths, or auditing composite-index column-order.
  Prevents the missing-index full scan, the composite-index wrong column-order trap, redundant indexes wasting writes, and indexing without considering selectivity.
  Covers B-tree (default), FULLTEXT (InnoDB + Aria + MyISAM), SPATIAL R-tree, hash indexes (MEMORY engine only), composite indexes with the leftmost-prefix rule, prefix indexes for long VARCHAR / TEXT / BLOB, descending indexes (10.8+), IGNORED indexes (10.6+), functional indexes on JSON paths via virtual / persistent columns, and EXPLAIN signals (type=ALL, Using filesort, Using temporary, Using index).
  Keywords: CREATE INDEX, ADD INDEX, PRIMARY KEY, UNIQUE, B-tree, BTREE, full-text, FULLTEXT, MATCH AGAINST, IN BOOLEAN MODE, spatial, SPATIAL INDEX, R-tree, RTREE, hash, HASH index, MEMORY engine, composite index, leftmost prefix, descending index, DESC index, IGNORED index, invisible index, functional index, JSON path index, virtual column index, prefix index, INDEX col length, ALGORITHM=INPLACE, why is my query slow, missing index, full table scan, type=ALL, Using filesort, Using temporary, Using index, key_len, index selectivity, ANALYZE TABLE, innodb_ft_min_token_size, ft_min_word_len, how do I index JSON, do indexes slow down inserts
license: MIT
compatibility: "Designed for Claude Code. Requires MariaDB 10.6-LTS, 10.11-LTS, 11.x, 12.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# MariaDB Indexing : B-tree, FULLTEXT, SPATIAL, Hash, Composite, Functional

Deterministic guidance for designing indexes on MariaDB. Use this skill to choose the right index type per engine, order composite columns correctly, index JSON paths through virtual columns, manage descending and IGNORED indexes, and read EXPLAIN to diagnose missing-index symptoms.

## Quick Reference

- **ALWAYS pick the right index type per engine**. InnoDB and Aria default to BTREE ; MEMORY supports HASH and BTREE ; MyISAM and Aria support FULLTEXT and SPATIAL (R-tree) ; InnoDB also supports FULLTEXT (since 10.0.5) and SPATIAL. **HASH is NOT available on InnoDB user tables.** Verified : KB `storage-engine-index-types/`.
- **ALWAYS put the highest-selectivity equality predicate first in a composite index**. Leftmost-prefix rule : `INDEX(a, b, c)` serves queries with predicates on `(a)`, `(a, b)`, `(a, b, c)` only. It does NOT serve `(b)`, `(c)`, or `(b, c)` alone. A composite in the wrong order is no better than no index.
- **NEVER index every column "just in case"**. Every INSERT, UPDATE, and DELETE pays a write-amplification cost for every index touched. Audit with `SHOW INDEX FROM t` and drop redundant indexes ; `INDEX(a)` is fully covered by `INDEX(a, b)` and is redundant.
- **NEVER write a functional index directly on an expression in MariaDB**. Per D-010, MariaDB JSON is a LONGTEXT alias, not a native binary type. There is no `CREATE INDEX ix ON t((JSON_VALUE(doc, '$.k')))` like in MySQL 8 or PostgreSQL. ALWAYS materialise the expression in a VIRTUAL or PERSISTENT generated column first, then index that column. Verified : KB `generated-columns/`.
- **Descending indexes are 10.8+**. `CREATE INDEX ix ON t (col DESC)` avoids `Using filesort` for `ORDER BY col DESC` only on 10.8 and later. On 10.6-LTS and 10.11-LTS the `DESC` keyword parses but is silently ignored (treated as ASC). Verified : KB `create-index/` grammar and 10.8 release notes.
- **IGNORED indexes (10.6+) are MariaDB's equivalent of MySQL "invisible indexes"**. Use `ALTER TABLE t ALTER INDEX ix IGNORED` to hide the index from the optimizer without dropping it. The index is still maintained on writes. ALWAYS toggle IGNORED before `DROP INDEX` in production to prove the index is not load-bearing. The keyword is `IGNORED`, NOT `INVISIBLE` ; MySQL syntax does NOT parse on MariaDB.
- **FULLTEXT minimum word length is engine-dependent**. InnoDB : `innodb_ft_min_token_size` default 3. MyISAM / Aria : `ft_min_word_len` default 4. A search for `"AI"` returns no rows on MyISAM by default. Verified : KB `full-text-index-overview/`.
- **SPATIAL INDEX requires `NOT NULL`** on the geometry column for InnoDB and MyISAM. The R-tree cannot index NULL geometries. ALWAYS declare `geom GEOMETRY NOT NULL`.
- **Prefix indexes (`INDEX(col(N))`) are mandatory for TEXT and BLOB**, optional for long VARCHAR. **A prefix index can NEVER be a covering index** : the storage engine has only the prefix, not the full value, so the row must still be fetched for any non-prefix column.
- **EXPLAIN signals you must learn to spot** : `type=ALL` = full scan = missing or unused index ; `Using filesort` = no index satisfies the ORDER BY ; `Using temporary` = no index satisfies the GROUP BY ; `Using index` = covering index, no row fetch ; `Using index condition` = ICP push-down, good. Verified : KB `explain/`.

## Decision Trees

### Tree 1 : "Which index type do I need?"

```
What is the query pattern?
  Equality (=) or range (<, >, BETWEEN) on scalar column ?
    Engine is InnoDB / Aria / MyISAM ? -> BTREE (the default).
    Engine is MEMORY and equality-only ? -> HASH (faster for =, useless for range).
  Free-text "find rows containing word(s)" ?
    -> FULLTEXT. InnoDB since 10.0.5, MyISAM, Aria. Use MATCH ... AGAINST.
  Geographic / geometric "find points inside polygon" ?
    -> SPATIAL R-tree. Requires NOT NULL GEOMETRY column.
  JSON path lookup "find rows where $.status = 'active'" ?
    -> Virtual column over JSON_VALUE(doc, '$.status'), then BTREE index on the column.
  Sort by long VARCHAR / TEXT prefix ?
    -> Prefix index INDEX(col(N)). Not covering.
```

### Tree 2 : "Composite index column order"

```
List the predicates in the WHERE clause and the ORDER BY.
  Are there equality predicates (col = ?) ?
    Yes : put the equality columns FIRST, ordered by selectivity (most selective first).
    Selectivity = approx. (distinct values / total rows). Higher is better.
  Is there a range predicate (col < ?, BETWEEN, IN with > 200 values) ?
    Yes : put it LAST in the equality-prefix. The optimizer cannot use any column
          to the right of a range predicate for further index lookup.
  Is there an ORDER BY ?
    Yes : ORDER BY columns must continue the index in the SAME direction
          (all ASC, or all DESC since 10.8). Otherwise filesort is forced.

Example :
  Query : WHERE customer_id = ? AND status = 'open' AND created_at > ? ORDER BY created_at DESC
  Index : INDEX(customer_id, status, created_at DESC)        -- 10.8+
  Index : INDEX(customer_id, status, created_at)             -- 10.6 / 10.11 (DESC ignored)
```

### Tree 3 : "Do I want to drop this index?"

```
Does the index show up in production EXPLAIN plans ?
  Unknown ? Capture a representative workload first via slow-log + pt-query-digest
            or PERFORMANCE_SCHEMA.events_statements_history_long.
  Never used ? -> Mark IGNORED (10.6+) for one full business cycle :
                  ALTER TABLE t ALTER INDEX ix IGNORED;
                  Monitor query plans + latency. Index is still maintained.
                  Confirmed safe ? -> DROP INDEX ix ON t;
                  Regression detected ? -> ALTER TABLE t ALTER INDEX ix NOT IGNORED;
  Used by one slow query you can rewrite ? -> Rewrite first, then re-evaluate.
```

### Tree 4 : "Index JSON path"

```
The JSON document has a stable key path you query repeatedly ?
  Yes :
    Step 1 : ALTER TABLE t ADD COLUMN status VARCHAR(32)
               AS (JSON_VALUE(doc, '$.status')) VIRTUAL;
    Step 2 : CREATE INDEX ix_status ON t (status);
    Why VIRTUAL : computed on the fly, no extra disk space, index still works.
    Why PERSISTENT : if the JSON_VALUE expression is expensive and the query
                     reads the column often without using the index path.
  No, the keys are dynamic ? -> JSON cannot be efficiently indexed.
    Consider dynamic columns or a redesign.
```

## Patterns

### Pattern 1 : Composite index respecting the leftmost prefix

```sql
-- 10.6+ : composite for "filter by customer + status, sort by date".
CREATE TABLE orders (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  customer_id BIGINT UNSIGNED NOT NULL,
  status ENUM('open','paid','shipped','cancelled') NOT NULL,
  created_at DATETIME(6) NOT NULL,
  total DECIMAL(12,2) NOT NULL,
  INDEX ix_orders_cust_status_created (customer_id, status, created_at)
) ENGINE=InnoDB;
-- Serves : WHERE customer_id = ? AND status = ? ORDER BY created_at
-- Serves : WHERE customer_id = ?
-- Does NOT serve : WHERE status = ?  (status is not the leftmost column)
```

### Pattern 2 : Descending index for ORDER BY DESC (10.8+)

```sql
-- 10.8+ : avoids "Using filesort" on ORDER BY created_at DESC.
CREATE INDEX ix_log_created_desc ON access_log (created_at DESC);
-- 10.6 / 10.11 : DESC keyword parses but is silently ASC.
-- Workaround on older versions : the optimizer can scan a B-tree backwards,
-- which gives the same result but with a small per-row cost.
```

### Pattern 3 : Functional index on JSON path (via VIRTUAL column)

```sql
-- 10.6+ : index a JSON path. Required because MariaDB JSON = LONGTEXT (D-010).
CREATE TABLE event (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  payload JSON NOT NULL CHECK (JSON_VALID(payload)),
  -- VIRTUAL : computed on read, zero disk overhead, still indexable.
  event_type VARCHAR(64) AS (JSON_VALUE(payload, '$.type')) VIRTUAL,
  INDEX ix_event_type (event_type)
) ENGINE=InnoDB;
-- Query that uses the index :
SELECT * FROM event WHERE event_type = 'order.created';
```

### Pattern 4 : FULLTEXT search with BOOLEAN MODE

```sql
-- 10.0.5+ : InnoDB FULLTEXT, exact phrase + required word + excluded word.
CREATE TABLE article (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  title VARCHAR(255) NOT NULL,
  body MEDIUMTEXT NOT NULL,
  FULLTEXT KEY ft_article (title, body)
) ENGINE=InnoDB;

SELECT id, title
FROM article
WHERE MATCH (title, body) AGAINST ('+mariadb +indexing -mysql "boolean mode"' IN BOOLEAN MODE);
-- Note : innodb_ft_min_token_size = 3 by default ; a search for "AI" hits zero rows.
-- Set innodb_ft_min_token_size=2 globally and rebuild the index to allow shorter terms.
```

### Pattern 5 : SPATIAL R-tree index

```sql
-- 10.6+ : SPATIAL index requires NOT NULL geometry.
CREATE TABLE place (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(128) NOT NULL,
  location POINT NOT NULL,
  SPATIAL INDEX sx_place_location (location)
) ENGINE=InnoDB;

-- Find places inside a bounding box.
SELECT id, name
FROM place
WHERE MBRContains(
  ST_GeomFromText('POLYGON((0 0, 0 10, 10 10, 10 0, 0 0))'),
  location
);
```

### Pattern 6 : IGNORED index workflow (10.6+)

```sql
-- 10.6+ : test removal without dropping.
-- Step 1 : hide the index from the optimizer.
ALTER TABLE orders ALTER INDEX ix_old_status IGNORED;

-- Step 2 : observe latency + plans over one business cycle.
-- Step 3a : safe to drop ?
ALTER TABLE orders DROP INDEX ix_old_status;

-- Step 3b : regression detected ?
ALTER TABLE orders ALTER INDEX ix_old_status NOT IGNORED;
```

### Pattern 7 : Prefix index on TEXT (mandatory) and long VARCHAR (optional)

```sql
-- TEXT / BLOB : prefix length is REQUIRED (cannot index full value).
CREATE TABLE article (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  body MEDIUMTEXT NOT NULL,
  -- Index first 100 chars only. NOT a covering index ;
  -- queries that select body still fetch the row.
  INDEX ix_body_prefix (body(100))
) ENGINE=InnoDB;

-- utf8mb4 caveat : INDEX(col(N)) measures N CHARACTERS, but the on-disk
-- index entry is up to N*4 bytes. INDEX(col(255)) on utf8mb4 = 1020 bytes,
-- which exceeds the 767-byte limit on older Antelope row-format.
-- ALWAYS run InnoDB with ROW_FORMAT=DYNAMIC (default since 10.2).
```

### Pattern 8 : Hash index on MEMORY engine (the only legitimate case)

```sql
-- Hash : MEMORY engine only. Constant-time equality, useless for range.
CREATE TABLE session_cache (
  session_id CHAR(64) NOT NULL PRIMARY KEY,
  user_id BIGINT UNSIGNED NOT NULL,
  expires_at INT UNSIGNED NOT NULL,
  INDEX USING HASH ix_session_user (user_id)
) ENGINE=MEMORY;
-- WHERE user_id = ?    : index used (equality).
-- WHERE user_id > 100  : NOT used (hash has no order). Use BTREE instead.
```

### Pattern 9 : ALGORITHM and LOCK for online index changes

```sql
-- 10.6+ : add an index online without blocking writes.
ALTER TABLE orders
  ADD INDEX ix_status_created (status, created_at),
  ALGORITHM = INPLACE,
  LOCK = NONE;
-- ALGORITHM = INSTANT is NOT supported for index creation, only for
-- ADD COLUMN. Fall back to INPLACE for indexes.
-- If the server cannot honour LOCK = NONE, it fails fast instead of
-- silently degrading to a table-rebuild lock.
```

### Pattern 10 : Reading EXPLAIN to confirm the index is used

```sql
-- After adding an index, ALWAYS verify with EXPLAIN.
EXPLAIN
SELECT id, total
FROM orders
WHERE customer_id = 42 AND status = 'open'
ORDER BY created_at DESC;
-- Expect : type = ref, key = ix_orders_cust_status_created,
--          Extra = Using where (or Using index condition).
-- Red flags : type = ALL (no index used), Extra contains "Using filesort"
--             (ORDER BY not satisfied by index).
```

## Reference Links

- `references/methods.md` : full CREATE INDEX grammar, system variables, SHOW INDEX output.
- `references/examples.md` : 10+ working CREATE / ALTER / DROP INDEX examples per index type.
- `references/anti-patterns.md` : 8+ real-world index mistakes with the fix.

## External References

- MariaDB KB : `https://mariadb.com/kb/en/getting-started-with-indexes/`
- MariaDB KB : `https://mariadb.com/kb/en/create-index/`
- MariaDB KB : `https://mariadb.com/kb/en/storage-engine-index-types/`
- MariaDB KB : `https://mariadb.com/kb/en/full-text-index-overview/`
- MariaDB KB : `https://mariadb.com/kb/en/ignored-indexes/`
- MariaDB KB : `https://mariadb.com/kb/en/generated-columns/`
- MariaDB KB : `https://mariadb.com/kb/en/explain/`
- MariaDB KB : `https://mariadb.com/kb/en/index-hints-how-to-force-query-plans/`
