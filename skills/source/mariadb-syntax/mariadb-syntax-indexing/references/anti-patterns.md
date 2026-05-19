# MariaDB Indexing : Anti-Patterns

Real-world index mistakes with the symptom, root cause, and fix. Verified against KB `create-index/`, `full-text-index-overview/`, `ignored-indexes/`, `generated-columns/`, `explain/`, `innodb-row-formats/`, and MariaDB-server `sql/sql_table.cc` source.

---

## AP-01 : Composite index in the wrong column order

**Bad code :**

```sql
-- Query :
SELECT * FROM orders WHERE customer_id = 42 ORDER BY created_at DESC;

-- Wrong index :
CREATE INDEX ix_bad ON orders (created_at, customer_id);
```

**Why it fails :** The leftmost-prefix rule says an index `(a, b)` serves predicates on `(a)` and `(a, b)`, never on `(b)` alone. The query has an equality predicate on `customer_id` (the second column) and an ORDER BY on `created_at` (the first column). Because no predicate on `created_at` exists, the optimizer cannot use `ix_bad` for the WHERE clause ; it scans the full table.

**Symptoms :** `EXPLAIN type=ALL`, query slows as table grows.

**Fix :**

```sql
-- Equality column first, then the ORDER BY column.
CREATE INDEX ix_good ON orders (customer_id, created_at);
-- Or, on 10.8+ for DESC sort :
CREATE INDEX ix_good ON orders (customer_id, created_at DESC);
```

---

## AP-02 : Indexing low-selectivity columns (boolean flags, ENUM with 2-3 values)

**Bad code :**

```sql
-- 99 % of rows have is_active = 1.
CREATE TABLE user (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  email VARCHAR(255) NOT NULL,
  is_active BOOLEAN NOT NULL DEFAULT 1,
  INDEX ix_active (is_active)
);

-- Query :
SELECT * FROM user WHERE is_active = 1;
```

**Why it fails :** Selectivity = `distinct_values / total_rows`. A boolean with 99/1 distribution has effective selectivity ~0.01 ; reading the index then chasing 99 % of the row pointers is slower than a full table scan. The optimizer rightly ignores `ix_active` and falls back to `type=ALL`. The index then exists only as write-amplification overhead.

**Symptoms :** Index never appears in EXPLAIN, write throughput is lower than necessary.

**Fix :** Drop the index. Use a partial-condition pattern instead :

```sql
-- If you really need to find "inactive" rows (the rare case) :
CREATE INDEX ix_inactive_users
  ON user ((CASE WHEN is_active = 0 THEN id END));
-- Or via virtual column + index :
ALTER TABLE user
  ADD COLUMN inactive_id BIGINT UNSIGNED AS (CASE WHEN is_active = 0 THEN id END) VIRTUAL,
  ADD INDEX ix_inactive_id (inactive_id);
```

---

## AP-03 : Redundant indexes (left-prefix duplication)

**Bad code :**

```sql
CREATE TABLE orders (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  customer_id BIGINT UNSIGNED NOT NULL,
  status VARCHAR(16) NOT NULL,
  INDEX ix_cust (customer_id),
  INDEX ix_cust_status (customer_id, status)
);
```

**Why it fails :** Every query that uses `ix_cust` is equally well served by the leftmost prefix of `ix_cust_status`. `ix_cust` adds no plan that `ix_cust_status` cannot already provide. The cost is :

- Extra disk space (a full secondary index copy).
- Extra writes on every INSERT, UPDATE, DELETE.
- Extra memory pressure in the buffer pool.

**Symptoms :** `SHOW INDEX FROM orders` lists two indexes that start with the same column ; `pt-duplicate-key-checker` (Percona Toolkit) flags it.

**Fix :**

```sql
ALTER TABLE orders DROP INDEX ix_cust;
-- ix_cust_status now serves both single-column and composite predicates.
```

---

## AP-04 : Indexing every column "just in case"

**Bad code :**

```sql
CREATE TABLE wide_table (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  col_a INT, col_b INT, col_c INT, col_d INT, col_e INT,
  col_f INT, col_g INT, col_h INT, col_i INT, col_j INT,
  INDEX (col_a), INDEX (col_b), INDEX (col_c), INDEX (col_d),
  INDEX (col_e), INDEX (col_f), INDEX (col_g), INDEX (col_h),
  INDEX (col_i), INDEX (col_j)
);
```

**Why it fails :** Each INSERT must update every secondary index ; writes scale as `O(rows × indexes)`. On InnoDB, each index also lives in the buffer pool, consuming memory that would otherwise cache hot data. The optimizer rarely uses single-column indexes when composite predicates are involved ; the indexes earn their cost only if queries actually look like `WHERE col_x = ?` for a non-leading column.

**Symptoms :** Slow INSERTs that scale linearly with index count, buffer pool pressure, "the table got slower after we 'optimised' it".

**Fix :** Build indexes from observed query patterns, not speculation. Capture slow log + `EXPLAIN` over a representative window. Add indexes for actually-slow queries ; drop the speculative ones (use `IGNORED` first to verify safety on 10.6+).

---

## AP-05 : Direct functional index on a JSON expression

**Bad code :**

```sql
-- MySQL 8 syntax that does NOT work on MariaDB :
CREATE TABLE event (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  payload JSON NOT NULL
);
CREATE INDEX ix_json_path ON event ((JSON_VALUE(payload, '$.type')));
-- ERROR 1064 (42000): You have an error in your SQL syntax ...
```

**Why it fails :** Per D-010, MariaDB's `JSON` type is a `LONGTEXT` alias with a `CHECK (JSON_VALID(...))` hook, not a native binary type. MariaDB does NOT support indexing arbitrary expressions directly. The functional-index syntax from MySQL 8 / PostgreSQL is a parse error on MariaDB.

**Symptoms :** `ERROR 1064` on CREATE INDEX, or developers cargo-culting MySQL examples that never index the JSON path.

**Fix :** Materialise the expression in a generated column, then index that column.

```sql
ALTER TABLE event
  ADD COLUMN event_type VARCHAR(64) AS (JSON_VALUE(payload, '$.type')) VIRTUAL,
  ADD INDEX ix_event_type (event_type);
-- Query that uses the index :
SELECT * FROM event WHERE event_type = 'order.created';
```

---

## AP-06 : FULLTEXT short-term invisibility (`innodb_ft_min_token_size = 3`)

**Bad code :**

```sql
CREATE TABLE article (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  body TEXT NOT NULL,
  FULLTEXT KEY ft_body (body)
) ENGINE=InnoDB;

INSERT INTO article (body) VALUES ('AI and DB news for 5G networks.');

-- Returns ZERO rows :
SELECT * FROM article WHERE MATCH (body) AGAINST ('AI');
```

**Why it fails :** The default `innodb_ft_min_token_size` is **3**. Tokens of length 1 or 2 (such as `AI`, `DB`, `5G`) are silently dropped during indexing. The query parses successfully and returns an empty result with no error or warning. For MyISAM the equivalent variable is `ft_min_word_len`, default **4** ; the threshold is even higher.

**Symptoms :** "My full-text search misses short brand names / acronyms / language codes."

**Fix :** Lower the threshold globally AND rebuild the index. The threshold change does not retroactively re-tokenise existing data.

```sql
-- Step 1 : globally (write into [mysqld] section of my.cnf for persistence).
SET GLOBAL innodb_ft_min_token_size = 2;

-- Step 2 : rebuild every FULLTEXT index. OPTIMIZE TABLE re-tokenises.
OPTIMIZE TABLE article;
-- For MyISAM the equivalent is :
-- REPAIR TABLE article QUICK;
```

---

## AP-07 : SPATIAL INDEX on a nullable geometry column

**Bad code :**

```sql
CREATE TABLE place (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  location POINT,                            -- NULLABLE !
  SPATIAL INDEX sx_loc (location)
);
-- ERROR 1252 (42000): All parts of a SPATIAL index must be NOT NULL
```

**Why it fails :** R-tree indexes split a geometric universe into bounding rectangles. There is no defined location for a NULL geometry, so the index cannot place it. MariaDB rejects the index at DDL time. Developers sometimes "fix" the error by dropping the SPATIAL INDEX entirely, which then turns every spatial query into a full table scan.

**Symptoms :** DDL fails with error 1252, OR (worse) the SPATIAL INDEX was dropped and the table now does full scans for every `MBRContains` query.

**Fix :**

```sql
CREATE TABLE place (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  -- ALWAYS declare NOT NULL on the indexed geometry column.
  location POINT NOT NULL,
  SPATIAL INDEX sx_loc (location)
) ENGINE=InnoDB;

-- For records without a known location, use a sentinel point or a separate
-- "unknown-location" table. Do NOT make the column nullable.
```

---

## AP-08 : Prefix index too short on utf8mb4 (or hitting the legacy 767-byte limit)

**Bad code :**

```sql
-- Legacy InnoDB row format (Antelope), utf8mb4 :
CREATE TABLE article (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  url VARCHAR(2048) NOT NULL,
  -- 255 chars × 4 bytes/char = 1020 bytes > 767-byte limit.
  INDEX ix_url (url(255))
) ENGINE=InnoDB ROW_FORMAT=COMPACT CHARSET=utf8mb4;
-- ERROR 1709 (HY000): Index column size too large.
```

**Why it fails :** Each utf8mb4 character is up to 4 bytes. The legacy `Antelope` family (`ROW_FORMAT=COMPACT` / `REDUNDANT`) caps a single index entry at 767 bytes. `INDEX(col(255))` on a utf8mb4 column needs 1020 bytes per row. Older 10.x defaults sometimes pin this format, especially in MySQL-compatibility shims.

**Symptoms :** Error 1709 at `CREATE TABLE` or `CREATE INDEX`. Developers sometimes "fix" by reducing the prefix to 191 chars without understanding why ; that works but only because 191 × 4 = 764 bytes.

**Fix :** Use the modern `ROW_FORMAT=DYNAMIC` (default since MariaDB 10.2), which supports per-index entries up to 3072 bytes ; OR use a sensible prefix length that the application actually needs.

```sql
CREATE TABLE article (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  url VARCHAR(2048) NOT NULL,
  INDEX ix_url (url(191))
) ENGINE=InnoDB ROW_FORMAT=DYNAMIC CHARSET=utf8mb4;
-- Or, with DYNAMIC, prefix lengths up to 768 chars in utf8mb4 (3072 / 4) work.
```

---

## AP-09 : Using `FORCE INDEX` to paper over stale statistics

**Bad code :**

```sql
-- After bulk-loading 10 M rows without ANALYZE TABLE :
SELECT * FROM orders FORCE INDEX (ix_created_at)
WHERE created_at > NOW() - INTERVAL 1 DAY AND status = 'open';
```

**Why it fails :** The optimizer picks plans from cardinality statistics. After a bulk load, those numbers are stale ; the previously good index `ix_created_at` may now point at 80 % of the table, and a different composite (`ix_status_created`) is the better choice. `FORCE INDEX` overrides the optimizer ; it locks the plan to a now-bad index. The fix lingers in code forever, hiding the real issue.

**Symptoms :** Queries that "used to be fast" become slow after data growth ; ripping out `FORCE INDEX` makes them faster.

**Fix :**

```sql
-- Step 1 : refresh statistics.
ANALYZE TABLE orders;

-- Step 2 : re-EXPLAIN without the hint and trust the optimizer.
EXPLAIN
SELECT * FROM orders
WHERE created_at > NOW() - INTERVAL 1 DAY AND status = 'open';

-- Step 3 : if a specific index really needs to be excluded :
SELECT * FROM orders IGNORE INDEX (ix_created_at)
WHERE created_at > NOW() - INTERVAL 1 DAY AND status = 'open';
-- IGNORE INDEX has a smaller blast radius than FORCE INDEX.
```

---

## AP-10 : `INVISIBLE` keyword copied from MySQL 8

**Bad code :**

```sql
-- MySQL 8 dump replayed on MariaDB :
ALTER TABLE orders ALTER INDEX ix_status INVISIBLE;
-- ERROR 1064 (42000): You have an error in your SQL syntax ...
```

**Why it fails :** MariaDB calls this feature **IGNORED**, not **INVISIBLE**. The grammar is different (`ALTER INDEX ix IGNORED` vs `ALTER INDEX ix INVISIBLE`). A blind dump-and-restore from MySQL fails at parse time. Worse : silent search-and-replace tooling sometimes deletes the entire `ALTER INDEX` line, leaving the index active when the original intent was to hide it.

**Symptoms :** ERROR 1064 at restore, or "hidden" indexes are accidentally still visible.

**Fix :** Translate the keyword.

```sql
-- MySQL 8                                     MariaDB
-- ALTER INDEX ix VISIBLE                  ->  ALTER INDEX ix NOT IGNORED
-- ALTER INDEX ix INVISIBLE                ->  ALTER INDEX ix IGNORED
ALTER TABLE orders ALTER INDEX ix_status IGNORED;
```

For automated migration, run a regex over the dump :

```bash
sed -i \
  -e 's/ALTER INDEX \([a-zA-Z0-9_]*\) INVISIBLE/ALTER INDEX \1 IGNORED/g' \
  -e 's/ALTER INDEX \([a-zA-Z0-9_]*\) VISIBLE/ALTER INDEX \1 NOT IGNORED/g' \
  mysql_dump.sql
```
