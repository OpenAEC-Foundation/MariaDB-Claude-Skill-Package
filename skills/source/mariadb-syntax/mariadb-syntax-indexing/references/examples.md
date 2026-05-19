# MariaDB Indexing : Working Examples

Each example is annotated with the minimum MariaDB version. Every snippet has been pattern-verified against KB `create-index/`, `full-text-index-overview/`, `ignored-indexes/`, `generated-columns/`, and `explain/`. Run against MariaDB 10.6-LTS, 10.11-LTS, 11.x, or 12.x unless otherwise stated.

---

## Example 1 : Composite index proving the leftmost-prefix rule (10.6+)

```sql
CREATE TABLE orders (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  customer_id BIGINT UNSIGNED NOT NULL,
  status ENUM('open','paid','shipped','cancelled') NOT NULL,
  created_at DATETIME(6) NOT NULL,
  total DECIMAL(12,2) NOT NULL,
  INDEX ix_orders_cust_status_created (customer_id, status, created_at)
) ENGINE=InnoDB;

-- Uses the index (leftmost prefix : customer_id) :
EXPLAIN SELECT * FROM orders WHERE customer_id = 42;
-- type=ref, key=ix_orders_cust_status_created, key_len=8 (8 bytes for BIGINT).

-- Uses the index (leftmost prefix : customer_id, status) :
EXPLAIN SELECT * FROM orders WHERE customer_id = 42 AND status = 'open';
-- type=ref, key=ix_orders_cust_status_created, key_len=10 (8 + 2 for ENUM).

-- DOES NOT use the index : status is not the leftmost column.
EXPLAIN SELECT * FROM orders WHERE status = 'open';
-- type=ALL, key=NULL.   <-- full table scan, missing index.
```

---

## Example 2 : Descending index avoiding filesort (10.8+)

```sql
-- 10.8+ : DESC keyword honoured.
CREATE TABLE access_log (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  user_id BIGINT UNSIGNED NOT NULL,
  created_at DATETIME(6) NOT NULL,
  INDEX ix_log_user_created_desc (user_id, created_at DESC)
) ENGINE=InnoDB;

EXPLAIN
SELECT * FROM access_log
WHERE user_id = 7
ORDER BY created_at DESC
LIMIT 50;
-- 10.8+ : Extra=Using where  (no filesort).
-- 10.6 / 10.11 : Extra=Using where; Using filesort  (DESC ignored).
```

---

## Example 3 : FULLTEXT search in BOOLEAN MODE (InnoDB 10.0.5+)

```sql
CREATE TABLE article (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  title VARCHAR(255) NOT NULL,
  body MEDIUMTEXT NOT NULL,
  FULLTEXT KEY ft_article (title, body)
) ENGINE=InnoDB;

INSERT INTO article (title, body) VALUES
  ('MariaDB indexing primer', 'Composite indexes and the leftmost-prefix rule.'),
  ('MySQL gotchas',           'JSON storage differs significantly from MariaDB.');

-- Find articles mentioning both "mariadb" AND "indexing" but NOT "mysql" :
SELECT id, title
FROM article
WHERE MATCH (title, body)
      AGAINST ('+mariadb +indexing -mysql' IN BOOLEAN MODE);

-- Natural-language ranking (default mode) :
SELECT id, title,
       MATCH (title, body) AGAINST ('mariadb indexing') AS score
FROM article
WHERE MATCH (title, body) AGAINST ('mariadb indexing')
ORDER BY score DESC;

-- Caveat : innodb_ft_min_token_size = 3 by default. "AI" or "DB" never match.
-- To allow shorter tokens :
--   SET GLOBAL innodb_ft_min_token_size = 2;
--   -- then rebuild every FULLTEXT index :
--   OPTIMIZE TABLE article;
```

---

## Example 4 : SPATIAL R-tree index with POINT geometry (10.6+)

```sql
-- SPATIAL INDEX requires NOT NULL on the geometry column.
CREATE TABLE place (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(128) NOT NULL,
  location POINT NOT NULL,
  SPATIAL INDEX sx_place_location (location)
) ENGINE=InnoDB;

INSERT INTO place (name, location) VALUES
  ('Brussels', ST_GeomFromText('POINT(4.35 50.85)', 4326)),
  ('Antwerp',  ST_GeomFromText('POINT(4.40 51.22)', 4326));

-- MBR (minimum bounding rectangle) predicate uses the R-tree :
SELECT id, name
FROM place
WHERE MBRContains(
  ST_GeomFromText('POLYGON((4.0 50.5, 4.0 51.5, 5.0 51.5, 5.0 50.5, 4.0 50.5))', 4326),
  location
);
-- EXPLAIN : type=range, key=sx_place_location.
```

---

## Example 5 : Functional index on JSON path via VIRTUAL column (10.6+)

```sql
-- Per D-010 : MariaDB JSON is a LONGTEXT alias. Direct functional index
-- on JSON_VALUE(...) is NOT supported. Use a generated column.
CREATE TABLE event (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  payload JSON NOT NULL CHECK (JSON_VALID(payload)),
  -- VIRTUAL : computed on read, no disk overhead, index works.
  event_type VARCHAR(64) AS (JSON_VALUE(payload, '$.type')) VIRTUAL,
  INDEX ix_event_type (event_type)
) ENGINE=InnoDB;

INSERT INTO event (payload) VALUES
  ('{"type":"order.created","id":1001}'),
  ('{"type":"order.paid","id":1001}');

-- Query uses ix_event_type :
EXPLAIN
SELECT id, payload
FROM event
WHERE event_type = 'order.created';
-- type=ref, key=ix_event_type.
```

---

## Example 6 : IGNORED index removal workflow (10.6+)

```sql
-- Step 1 : suspect that ix_old_status is unused. Hide it from the optimizer.
ALTER TABLE orders ALTER INDEX ix_old_status IGNORED;

-- Step 2 : confirm the index is hidden.
SHOW INDEX FROM orders WHERE Key_name = 'ix_old_status';
-- Column "Ignored" = YES.

-- Step 3 : observe production query plans for one full business cycle.
-- If no regression after the cycle :
ALTER TABLE orders DROP INDEX ix_old_status;

-- If a regression appears (slow queries, type=ALL where it was type=ref) :
ALTER TABLE orders ALTER INDEX ix_old_status NOT IGNORED;
-- Optimizer can use the index again immediately ; no rebuild needed.
```

---

## Example 7 : Prefix index on TEXT and long VARCHAR (10.6+)

```sql
CREATE TABLE article (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  url VARCHAR(2048) NOT NULL,    -- long VARCHAR ; prefix is optional but useful.
  body MEDIUMTEXT NOT NULL,      -- TEXT ; prefix is mandatory.
  -- Index first 191 characters of url. utf8mb4 (4 bytes/char) -> 764 bytes
  -- which fits the 767-byte legacy limit AND the 3072-byte modern limit.
  INDEX ix_url_prefix (url(191)),
  -- TEXT MUST have a prefix length :
  INDEX ix_body_prefix (body(100))
) ENGINE=InnoDB ROW_FORMAT=DYNAMIC;

-- ix_url_prefix supports lookups by URL prefix :
EXPLAIN SELECT * FROM article WHERE url = 'https://example.com/articles/123';
-- type=ref, key=ix_url_prefix, key_len varies with prefix bytes.

-- A prefix index is NEVER a covering index :
EXPLAIN SELECT url FROM article WHERE url = 'https://example.com/articles/123';
-- Extra : does NOT contain "Using index" ; row is still fetched.
```

---

## Example 8 : HASH index on the MEMORY engine (10.6+)

```sql
-- The only legitimate user-defined HASH index. NOT available on InnoDB.
CREATE TABLE session_cache (
  session_id CHAR(64) NOT NULL PRIMARY KEY,
  user_id BIGINT UNSIGNED NOT NULL,
  expires_at INT UNSIGNED NOT NULL,
  -- HASH for fast equality, useless for range.
  INDEX USING HASH ix_session_user (user_id),
  -- BTREE for range scans on expires_at.
  INDEX USING BTREE ix_session_exp (expires_at)
) ENGINE=MEMORY;

EXPLAIN SELECT * FROM session_cache WHERE user_id = 42;
-- type=ref, key=ix_session_user.

EXPLAIN SELECT * FROM session_cache WHERE user_id > 100;
-- type=ALL, key=NULL.   <-- HASH cannot serve range ; full scan.

EXPLAIN SELECT * FROM session_cache WHERE expires_at < UNIX_TIMESTAMP();
-- type=range, key=ix_session_exp.   <-- BTREE handles the range.
```

---

## Example 9 : ALGORITHM=INPLACE LOCK=NONE online index add (10.6+)

```sql
-- Add an index on a busy production table without blocking writes.
ALTER TABLE orders
  ADD INDEX ix_status_created (status, created_at),
  ALGORITHM = INPLACE,
  LOCK = NONE;
-- If the server cannot meet LOCK = NONE, the statement fails fast
-- with "LOCK=NONE is not supported. Reason: ..." instead of silently
-- escalating to a table-rewrite lock.

-- Drop an index online :
ALTER TABLE orders
  DROP INDEX ix_old,
  ALGORITHM = INPLACE,
  LOCK = NONE;

-- INSTANT is NOT valid for index add/drop. The following fails :
-- ALTER TABLE orders ADD INDEX ix_a (a), ALGORITHM = INSTANT;
-- ERROR 1845 (0A000): ALGORITHM=INSTANT is not supported.
```

---

## Example 10 : Reading EXPLAIN to confirm an index is used (10.6+)

```sql
EXPLAIN FORMAT=JSON
SELECT id, total
FROM orders
WHERE customer_id = 42 AND status = 'open'
ORDER BY created_at DESC
LIMIT 20;
```

Interpret the output :

- `query_block.table.access_type` : aim for `ref`, `eq_ref`, `range`. Reject `ALL`.
- `query_block.table.key` : must be a non-null index name.
- `query_block.table.used_key_parts` : confirms how many composite columns are used.
- `query_block.table.using_filesort` : `true` means ORDER BY is not satisfied by the index ; fix with a descending index (10.8+) or by adjusting composite order.
- `query_block.table.using_index` : `true` means COVERING INDEX, no row fetch.

---

## Example 11 : Composite UNIQUE that prevents duplicate child rows (10.6+)

```sql
-- A common pattern : ensure no two votes from the same user on the same poll.
CREATE TABLE vote (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  poll_id BIGINT UNSIGNED NOT NULL,
  user_id BIGINT UNSIGNED NOT NULL,
  choice TINYINT NOT NULL,
  created_at DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  UNIQUE KEY uk_vote_poll_user (poll_id, user_id),
  INDEX ix_vote_user_created (user_id, created_at)
) ENGINE=InnoDB;

-- The UNIQUE composite serves two purposes :
-- 1. Prevents (poll_id, user_id) duplicates at the schema level.
-- 2. Acts as a fast lookup index for "did user X vote on poll Y ?".

-- Use the UNIQUE for upsert :
INSERT INTO vote (poll_id, user_id, choice)
VALUES (10, 7, 1)
ON DUPLICATE KEY UPDATE choice = VALUES(choice);
```

---

## Example 12 : INVISIBLE-to-MariaDB-IGNORED migration pattern (10.6+)

```sql
-- A MySQL 8 schema dump contains :
--   ALTER TABLE t ALTER INDEX ix_x INVISIBLE;
-- On MariaDB this fails with a syntax error.
-- Translate to the IGNORED keyword :

ALTER TABLE t ALTER INDEX ix_x IGNORED;
-- Functionally equivalent : maintained but hidden from the optimizer.

-- And the reverse direction :
--   MySQL 8 : ALTER TABLE t ALTER INDEX ix_x VISIBLE;
--   MariaDB : ALTER TABLE t ALTER INDEX ix_x NOT IGNORED;
```
