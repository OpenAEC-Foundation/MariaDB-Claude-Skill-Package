# MariaDB JSON : Anti-Patterns

8+ real-world JSON mistakes on MariaDB with the failure mode and the fix. Verified against KB and the cluster-1 research fragment (L-005, D-010) on 2026-05-19.

## Anti-Pattern 1 : JSON column without `CHECK (JSON_VALID(col))`

### The mistake

```sql
-- DO NOT WRITE THIS.
CREATE TABLE event (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  payload JSON NOT NULL                -- no CHECK constraint
);
```

### Why it fails

MariaDB `JSON` is a LONGTEXT alias (D-010, L-005). When the explicit CHECK is absent and the column is later copied to a true `LONGTEXT` column during a dump-and-reload, the implicit CHECK is lost. An `INSERT INTO event (payload) VALUES ('not even json')` then succeeds silently and every downstream `JSON_EXTRACT` / `JSON_VALUE` returns `NULL` on that row. The data corruption is invisible until a consumer crashes.

### The fix

```sql
CREATE TABLE event (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  payload JSON NOT NULL CHECK (JSON_VALID(payload))
);
```

ALWAYS write the CHECK explicitly. It survives `mysqldump --no-data`, `pt-online-schema-change`, and any column-type change.

## Anti-Pattern 2 : Comparing `JSON_EXTRACT` to an unquoted string

### The mistake

```sql
-- DO NOT WRITE THIS.
SELECT * FROM event WHERE JSON_EXTRACT(payload, '$.status') = 'open';
```

### Why it fails

`JSON_EXTRACT` returns JSON in all cases, including for scalars. The scalar comes back JSON-quoted : `"open"`, not `open`. The string comparison `"open" = 'open'` is false because the left side includes literal double quotes. The query returns zero rows even though matching documents exist. Junior developers spend hours debugging "why is my JSON query empty".

### The fix

```sql
-- Use JSON_VALUE for unwrapped scalars in WHERE :
SELECT * FROM event WHERE JSON_VALUE(payload, '$.status') = 'open';

-- Or strip the quotes with JSON_UNQUOTE :
SELECT * FROM event WHERE JSON_UNQUOTE(JSON_EXTRACT(payload, '$.status')) = 'open';
```

ALWAYS use `JSON_VALUE` in `WHERE` and `ORDER BY` on scalar paths.

## Anti-Pattern 3 : Direct expression-index on a JSON path (MySQL 8 syntax)

### The mistake

```sql
-- DO NOT WRITE THIS on MariaDB.
CREATE INDEX ix_event_type ON event ((JSON_VALUE(payload, '$.type')));
-- ERROR 1064 : syntax error.
```

### Why it fails

Direct expression-indexes (the `((expression))` syntax from MySQL 8 and PostgreSQL) are NOT supported on MariaDB because JSON is a LONGTEXT alias and there is no native expression-index infrastructure. The parser rejects the second pair of parentheses. Users migrating from MySQL 8 hit this immediately.

### The fix

Materialise the expression in a virtual or persistent generated column first, then index that column.

```sql
-- 10.6+ : the canonical MariaDB pattern.
ALTER TABLE event
  ADD COLUMN event_type VARCHAR(64)
    AS (JSON_VALUE(payload, '$.type')) PERSISTENT,
  ADD INDEX ix_event_type (event_type);
```

Use `VIRTUAL` when disk space matters more than read latency ; use `PERSISTENT` when reads dominate and the JSON_VALUE expression is non-trivial.

## Anti-Pattern 4 : Using deprecated `JSON_MERGE`

### The mistake

```sql
-- DO NOT WRITE THIS.
UPDATE user_profile
SET prefs = JSON_MERGE(prefs, '{"theme":null}')
WHERE id = 1;
-- Intent : remove the "theme" key (PATCH semantics).
-- Actual : prefs becomes {"theme":[<old>, null]} (PRESERVE semantics).
```

### Why it fails

`JSON_MERGE` is deprecated and silently aliases `JSON_MERGE_PRESERVE`, NOT `JSON_MERGE_PATCH`. Code migrated from MySQL 5.7 where `JSON_MERGE` had PATCH-like intent (it does not, the same alias rule applies in MySQL 5.7+) produces wrong results on MariaDB. The data corruption is silent : the column contains valid JSON, just the wrong shape.

### The fix

Pick PATCH or PRESERVE explicitly :

```sql
-- 10.2+ : PATCH for partial-update semantics (null removes the key).
UPDATE user_profile
SET prefs = JSON_MERGE_PATCH(prefs, '{"theme":null}')
WHERE id = 1;

-- 10.2+ : PRESERVE for combining all keys and concatenating arrays.
UPDATE audit_log
SET tags = JSON_MERGE_PRESERVE(tags, '["urgent","reviewed"]')
WHERE id = 1;
```

NEVER use `JSON_MERGE` ; CI should fail on its presence.

## Anti-Pattern 5 : Storing huge JSON blobs in a wide table

### The mistake

```sql
-- DO NOT WRITE THIS for high-volume tables.
CREATE TABLE order_event (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  customer_id BIGINT UNSIGNED NOT NULL,
  status ENUM('open','paid','shipped') NOT NULL,
  -- ... 30 more scalar columns ...
  raw_event JSON NOT NULL CHECK (JSON_VALID(raw_event))   -- often > 100 KB
);
```

### Why it fails

JSON is LONGTEXT. Large LONGTEXT values are stored off-page by InnoDB (DYNAMIC row format), but the row header still has to seek to the off-page LOB on read. Every `SELECT *` pays the off-page cost. With `ROW_FORMAT=COMPACT` (default on very old servers) the LOB inlines up to 768 bytes per column and overflows after, often exceeding the 8 KB half-page limit and refusing the row insert outright.

### The fix

Two options :

```sql
-- Option A : split the payload into its own table, joined by FK.
CREATE TABLE order_event (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  customer_id BIGINT UNSIGNED NOT NULL,
  status ENUM('open','paid','shipped') NOT NULL,
  INDEX ix_oe_customer (customer_id)
) ENGINE=InnoDB;

CREATE TABLE order_event_payload (
  order_event_id BIGINT UNSIGNED NOT NULL PRIMARY KEY,
  payload JSON NOT NULL CHECK (JSON_VALID(payload)),
  FOREIGN KEY (order_event_id) REFERENCES order_event(id) ON DELETE CASCADE
) ENGINE=InnoDB ROW_FORMAT=DYNAMIC;

-- Option B : keep the JSON inline but ALWAYS use ROW_FORMAT=DYNAMIC
-- (default on 10.2+) and never SELECT * in hot paths.
ALTER TABLE order_event ROW_FORMAT=DYNAMIC;
```

ALWAYS run `ROW_FORMAT=DYNAMIC` (default on 10.2+ but verify legacy tables). NEVER `SELECT *` on tables with a large JSON column in hot paths.

## Anti-Pattern 6 : Recursive descent with `..` (PostgreSQL / jq syntax)

### The mistake

```sql
-- DO NOT WRITE THIS on MariaDB.
SELECT JSON_EXTRACT(doc, '$..city');     -- ERROR 3143 : Invalid JSON path expression.
```

### Why it fails

MariaDB recursive descent uses the `**` token, not `..`. The double-dot syntax is from JSONPath dialects (jq, PostgreSQL jsonb_path_query). MariaDB's parser rejects it as malformed. Additionally `**` must be followed by another step : a final `$**` does not parse either.

### The fix

```sql
-- Correct MariaDB recursive descent : $**.<step>
SELECT JSON_EXTRACT(doc, '$**.city');
-- Returns every "city" value at any depth.

-- For a single deterministic path, write it out :
SELECT JSON_VALUE(doc, '$.user.addr.city');
```

## Anti-Pattern 7 : Mixing character sets on JSON insert

### The mistake

```sql
-- DO NOT WRITE THIS without setting the connection charset first.
mysql --default-character-set=latin1 -e \
  "INSERT INTO event (payload) VALUES ('{\"name\":\"\xe9\xe9\xe9\"}')"
```

### Why it fails

MariaDB JSON storage is `LONGTEXT COLLATE utf8mb4_bin`. The client sends `latin1` bytes ; the server interprets them as `latin1` and transcodes to `utf8mb4`. If the JSON document already contained UTF-8 multi-byte sequences encoded in another charset, they are mangled silently. The downstream `JSON_VALUE` returns Mojibake instead of the intended string.

### The fix

```sql
-- Always run the connection in utf8mb4.
SET NAMES utf8mb4;

INSERT INTO event (payload) VALUES ('{"name":"ééé"}');
-- Or send literal UTF-8 bytes after SET NAMES utf8mb4.
```

Set the server-side defaults too :

```ini
# [mysqld]
character-set-server      = utf8mb4
collation-server          = utf8mb4_uca1400_ai_ci   # 11.x ; on 10.6 / 10.11 use utf8mb4_unicode_ci
```

ALWAYS run JSON workloads end-to-end in `utf8mb4`.

## Anti-Pattern 8 : Relying on MySQL-style structural equality

### The mistake

```sql
-- DO NOT WRITE THIS expecting MySQL semantics.
SELECT * FROM event WHERE payload = '{"a":1,"b":2}';
-- Misses rows with '{"b":2,"a":1}' or extra whitespace.
```

### Why it fails

MySQL stores JSON in a canonical binary form, so its `=` comparison ignores key order and whitespace. MariaDB stores JSON as plain text, so `=` is a literal string compare ; different key order or any whitespace difference causes the row to miss. This is one of the largest behavioural divergences from MySQL JSON and is silently wrong, never raises an error.

### The fix

```sql
-- Use JSON_EQUALS for structural equality on MariaDB.
SELECT * FROM event WHERE JSON_EQUALS(payload, '{"a":1,"b":2}');

-- Or normalise to a canonical form first :
SELECT * FROM event WHERE JSON_NORMALIZE(payload) = JSON_NORMALIZE('{"a":1,"b":2}');
```

ALWAYS use `JSON_EQUALS` or `JSON_NORMALIZE` when porting MySQL JSON code that relies on structural equality.

## Anti-Pattern 9 : `JSON_EXTRACT` in `WHERE` without a functional index

### The mistake

```sql
-- DO NOT WRITE THIS on a large table.
SELECT id FROM event WHERE JSON_EXTRACT(payload, '$.type') = '"order.created"';
-- EXPLAIN : type=ALL, full table scan, even though "type" is hot.
```

### Why it fails

The optimizer cannot push a `JSON_EXTRACT` expression into an index lookup on a base JSON column. It evaluates the expression row-by-row, producing a full table scan. On a 100 M row table that is a multi-minute query. The `JSON_EXTRACT = "quoted"` form also reintroduces the quoting bug from anti-pattern 2.

### The fix

```sql
-- Step 1 : materialise the hot path in a generated column.
ALTER TABLE event
  ADD COLUMN event_type VARCHAR(64)
    AS (JSON_VALUE(payload, '$.type')) PERSISTENT,
  ADD INDEX ix_event_type (event_type);

-- Step 2 : query the generated column.
SELECT id FROM event WHERE event_type = 'order.created';
-- EXPLAIN : type=ref, key=ix_event_type. Sub-millisecond on the same table.
```

ALWAYS index the hot JSON paths through a generated column ; NEVER let production code call `JSON_EXTRACT` in a `WHERE` on an un-indexed path.

## Anti-Pattern 10 : Treating JSON as a schema substitute

### The mistake

```sql
-- DO NOT WRITE THIS.
CREATE TABLE everything (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  doc JSON NOT NULL CHECK (JSON_VALID(doc))
);
-- All entities live in one table, all queries are JSON_EXTRACT.
```

### Why it fails

JSON is a tool for storing flexible attribute bags, not a replacement for a relational schema. A single mega-table forces every query through full-scan JSON parsing, makes foreign keys impossible, makes statistics meaningless, and produces zero index usage. The migration cost out of this design grows linearly with the row count ; an 18-month-old schema becomes a multi-month rewrite.

### The fix

Use the decision tree from `SKILL.md` Tree 4 :

- Stable structure, frequent joins -> normalise into columns and rows.
- Flexible attribute bag, sparse keys -> JSON column on a typed entity table.
- Black-box payload to replay -> LONGTEXT (or JSON if you need JSON_* downstream).

JSON is a column type, not an architecture.
