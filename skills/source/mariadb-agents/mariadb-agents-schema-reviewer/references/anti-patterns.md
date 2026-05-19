# anti-patterns : how a schema review goes wrong

These are anti-patterns of the REVIEWER, not of the schema. Each one describes a way Claude can run this skill incorrectly and ship a broken schema with a clean-looking report. For each : the wrong behaviour, why it fails, and the correct procedure.

---

## Anti-pattern 1 : reviewing syntax instead of design

**Wrong behaviour** : the reviewer reads the DDL, confirms it parses without a syntax error, and reports `PASS`. It never asked whether the engine, keys, or indexes are correct.

**Why it fails** : a schema can be 100 % syntactically valid and still be a BLOCKER-grade design. `ENGINE=MyISAM`, `total FLOAT`, and a missing `PRIMARY KEY` all parse cleanly. The DDL compiling proves nothing about whether the schema should ship. A "PASS" verdict on a syntactically-valid but structurally-broken schema is the most damaging reviewer failure : the user trusts it and ships.

**Correct procedure** : syntax validity is a precondition, not a finding. Always run all 10 design dimensions from `methods.md`. The review is about engine choice, key strategy, index column-order, normalization, and data-type fitness, never about whether the parser accepts the text.

---

## Anti-pattern 2 : skipping the storage-engine check

**Wrong behaviour** : the reviewer jumps straight to indexes and data types because those feel like the "real" review, and never inspects the `ENGINE` clause.

**Why it fails** : storage engine is the single most expensive schema mistake and the hardest to reverse without downtime. A MyISAM table has no transactions, no row-level locking, no crash recovery, and no foreign keys. A review that grades indexes perfectly but misses `ENGINE=MyISAM` has graded the paint on a car with no brakes. The engine check is dimension 1 because it must run first.

**Correct procedure** : always start with dimension 1. For every table, locate the `ENGINE` clause. MyISAM or MEMORY on transactional data is a BLOCKER. A missing clause is a NOTE (defaults to InnoDB since 10.2). Never report a verdict without having explicitly checked the engine of every table.

---

## Anti-pattern 3 : accepting a UUID-text primary key silently

**Wrong behaviour** : the reviewer sees `id CHAR(36) PRIMARY KEY`, notes the table has a PK, marks the primary-key dimension `PASS`, and moves on.

**Why it fails** : "has a primary key" is not the same as "has a good primary key". A `CHAR(36)` UUID string is up to 144 bytes in utf8mb4 and InnoDB copies the full PK into every secondary index. On a 100-million-row table this is the difference between a 2 GB and a 14 GB index footprint, plus random-insert page splits. The schema works, so it is not a BLOCKER, but silently passing it hides a real, quantifiable cost.

**Correct procedure** : the primary-key dimension grades the TYPE of the key, not just its presence. A `CHAR(36)` / `VARCHAR(36)` UUID-text PK is always a WARNING with the fix : `BINARY(16)` plus `UUID_TO_BIN(uuid, 1)`. Only `BIGINT UNSIGNED AUTO_INCREMENT` or `BINARY(16)` time-ordered keys pass clean.

---

## Anti-pattern 4 : not checking composite-index column order

**Wrong behaviour** : the reviewer counts indexes, confirms the important columns each appear in some index, and reports the indexing dimension `PASS`.

**Why it fails** : an index existing is not an index being usable. The leftmost-prefix rule means `INDEX(created_at, customer_id)` cannot serve `WHERE customer_id = ?` : the optimiser scans the whole table while `SHOW CREATE TABLE` shows an index that looks fine. Counting indexes without checking column order against the query pattern produces a confident `PASS` on a schema that will table-scan in production.

**Correct procedure** : for every composite index, ask what query it is meant to serve. The leftmost column must be the highest-selectivity equality predicate ; range predicates go last. If the column order does not match the query pattern, it is a WARNING. Also flag redundant indexes : an `INDEX(a)` alongside `INDEX(a, b)` is a SUGGESTION to drop the prefix.

---

## Anti-pattern 5 : ignoring charset and collation

**Wrong behaviour** : the reviewer treats charset as cosmetic, sees `CHARSET=utf8`, and either skips it or treats it as a SUGGESTION at most.

**Why it fails** : `utf8` in MariaDB is the 3-byte `utf8mb3`. It physically cannot store 4-byte characters : emoji, many CJK extension characters, and some scripts trigger `Incorrect string value` on insert and the row is rejected or truncated. This is a functional data-loss bug, not a style issue. On stock 10.6 / 10.11 the server default is still `latin1`, so a table with no explicit charset is silently worse. Treating this as cosmetic ships a schema that loses user data.

**Correct procedure** : the charset dimension is a WARNING, not a SUGGESTION. `utf8` / `utf8mb3` on text columns is a WARNING (cannot store 4-byte characters). `latin1` with no documented reason is a WARNING (silent stock default, emoji break). A collation mismatch on join columns is a WARNING (kills index use). Only consistent `utf8mb4` passes.

---

## Anti-pattern 6 : reviewing without severity grading

**Wrong behaviour** : the reviewer produces a flat list of observations ("the engine is MyISAM", "there is a JSON column", "the naming is mixed-case") with no severity and no verdict. The user cannot tell what blocks shipping.

**Why it fails** : a review without grading gives the user no decision. A BLOCKER (MyISAM transactional table) and a SUGGESTION (mixed-case naming) read as equal weight. The user either ships everything because nothing said "stop", or fixes everything including cosmetics and wastes effort. The value of a review is the prioritisation, and an ungraded list throws it away.

**Correct procedure** : every finding gets exactly one of BLOCKER, WARNING, SUGGESTION (or NOTE for context). The findings table is sorted by severity. The review always ends with a verdict line : any BLOCKER means `FAIL`, BLOCKER-free with warnings means `PASS WITH WARNINGS`, fully clean means `PASS`. Never emit a review without a verdict.

---

## Anti-pattern 7 : reporting a finding without a skill reference

**Wrong behaviour** : the reviewer states a problem and a fix but leaves the skill-ref column blank, or writes a generic phrase like "see the docs".

**Why it fails** : this is an orchestration skill ; its job is to route the user to the canonical skill that explains the rule in depth. The findings table has a fixed five-fields-plus-skill-ref shape for a reason : the fix column is one line, and the real explanation lives in the referenced skill. A finding with no skill-ref dead-ends the user when they need the why and the full fix.

**Correct procedure** : every finding cites the exact canonical skill from the dimension table : `mariadb-core-storage-engines`, `mariadb-impl-schema-design`, `mariadb-syntax-indexing`, `mariadb-errors-encoding-and-collation`, `mariadb-syntax-check-constraints`, `mariadb-syntax-json`, or `mariadb-syntax-sql-ddl`. Use the exact skill name, never a paraphrase. If a finding has no canonical skill, it is not a valid finding for this skill.
