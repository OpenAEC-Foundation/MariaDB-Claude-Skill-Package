---
name: mariadb-agents-schema-reviewer
description: >
  Use when reviewing a proposed MariaDB schema before it ships, auditing an existing schema for engine / indexing / naming / normalization problems, or validating a migration DDL.
  Prevents the common mistake of shipping a schema with MyISAM tables, UUID-text PKs, missing tenant indexes, utf8 charset, or composite indexes in the wrong column order.
  Covers a deterministic schema-review checklist : storage-engine choice, primary-key type, indexing strategy and column-order, charset / collation, normalization fitness, multi-tenant pattern detection, naming-convention adherence, with severity grading and cross-references to mariadb-core-storage-engines, mariadb-syntax-indexing, mariadb-impl-schema-design.
  Keywords: schema review, schema audit, review my schema, is this schema correct, schema checklist, design review, storage engine audit, index audit, primary key audit, normalization check, multi-tenant check, naming convention, DDL review, before I ship this schema
license: MIT
compatibility: "Designed for Claude Code. Requires MariaDB 10.6-LTS, 10.11-LTS, 11.x, 12.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# mariadb-agents-schema-reviewer

A deterministic procedure for reviewing a MariaDB schema (CREATE TABLE DDL, migration script, or existing schema dump) before it ships. This is an ORCHESTRATION skill : it does not teach syntax, it applies a fixed 10-dimension checklist and produces a graded findings report. Every check cites the canonical skill that explains the underlying rule.

## Quick Reference

- Run ALL 10 review dimensions in order. Never skip a dimension because the schema "looks fine".
- Grade every finding with one of three severities : BLOCKER, WARNING, SUGGESTION.
- **BLOCKER** : the schema must not ship. Examples : MyISAM on a transactional table, a table with no PRIMARY KEY, FLOAT used for a money column.
- **WARNING** : the schema works but carries a known cost or footgun. Examples : CHAR(36) UUID-text PK on InnoDB, `utf8` (utf8mb3) charset on text columns, a multi-tenant table with no leftmost `tenant_id` index.
- **SUGGESTION** : a stylistic or maintainability improvement. Examples : inconsistent naming case, oversized VARCHAR, TEXT where VARCHAR fits.
- Every finding cites the canonical skill that explains the rule. NEVER report a problem without a skill-ref.
- Output a single findings table : dimension, severity, location, problem, fix, skill-ref. Close with a verdict line.
- A clean schema still gets a report : the table has zero rows and the verdict is `PASS`.

## The 10 Review Dimensions

| # | Dimension | What it checks | Canonical skill |
|---|-----------|----------------|-----------------|
| 1 | Storage engine | InnoDB for transactional tables ; flag MyISAM | mariadb-core-storage-engines |
| 2 | Primary key | every table has a PK ; flag UUID-text PK | mariadb-impl-schema-design |
| 3 | Indexing | composite column-order, redundant indexes, FK indexes | mariadb-syntax-indexing |
| 4 | Charset / collation | utf8mb4 on text columns ; flag utf8 / latin1 | mariadb-errors-encoding-and-collation |
| 5 | Normalization | repeating groups, unjustified denormalization | mariadb-impl-schema-design |
| 6 | Multi-tenant | detect `tenant_id` ; flag missing leftmost tenant index | mariadb-impl-schema-design |
| 7 | Naming | case consistency, reserved words, Frappe `tab` pattern | mariadb-impl-schema-design |
| 8 | Constraints | missing CHECK / FK where invariants are implied | mariadb-syntax-check-constraints |
| 9 | Data types | DECIMAL for money, oversized VARCHAR, TEXT misuse | mariadb-syntax-sql-ddl |
| 10 | JSON | JSON column without `CHECK (JSON_VALID(col))` | mariadb-syntax-json |

The full rule set per dimension (IF condition THEN severity) is in `references/methods.md`.

## Review Decision Tree

```
For each table in the schema :

  1. ENGINE clause present ?
       absent  -> NOTE : engine defaults to InnoDB (acceptable, 10.2+)
       MyISAM / MEMORY on a table with FK / transactional intent -> BLOCKER
       ARIA on user data -> WARNING
       -> ref mariadb-core-storage-engines

  2. PRIMARY KEY present ?
       absent -> BLOCKER (InnoDB synthesises a hidden 6-byte PK, unindexable)
       CHAR(36) / VARCHAR(36) UUID-text PK -> WARNING (use BINARY(16))
       -> ref mariadb-impl-schema-design

  3. Indexes :
       composite index leftmost column not the equality predicate -> WARNING
       two indexes where one is a prefix of the other -> SUGGESTION (redundant)
       FOREIGN KEY column with no index -> WARNING
       -> ref mariadb-syntax-indexing

  4. CHARSET on text columns :
       utf8 / utf8mb3 -> WARNING (3-byte, no emoji, no full Unicode)
       latin1 with no explicit reason -> WARNING
       utf8mb4 -> PASS
       -> ref mariadb-errors-encoding-and-collation

  5. Normalization :
       column names like addr1, addr2, addr3 (repeating group) -> WARNING
       comma-separated list stored in one column -> WARNING
       denormalized copy with no stated reason -> SUGGESTION
       -> ref mariadb-impl-schema-design

  6. Multi-tenant :
       tenant_id / org_id / company column present ?
         YES and no index starts with it -> WARNING
         YES and PK does not lead with it (for tenant-scoped tables) -> SUGGESTION
       -> ref mariadb-impl-schema-design

  7. Naming :
       mixed case across tables (Users + order_line) -> SUGGESTION
       reserved word as identifier without backticks -> BLOCKER
       tab<Doctype> prefix -> NOTE : Frappe/ERPNext schema, do not rename
       -> ref mariadb-impl-schema-design

  8. Constraints :
       status / type column with finite value set, no CHECK -> SUGGESTION
       child table referencing a parent, no FOREIGN KEY -> WARNING
       -> ref mariadb-syntax-check-constraints

  9. Data types :
       FLOAT / DOUBLE for a money / price / amount column -> BLOCKER
       VARCHAR(255) for a country code / boolean flag -> SUGGESTION
       TEXT for a column that is always short -> SUGGESTION
       -> ref mariadb-syntax-sql-ddl

 10. JSON :
       JSON column without CHECK (JSON_VALID(col)) -> WARNING
       JSON column queried by path with no virtual-column index -> SUGGESTION
       -> ref mariadb-syntax-json
```

## JSON Storage Caveat

MariaDB JSON is a LONGTEXT alias, not native binary. Use `CHECK (JSON_VALID(col))` for structure ; use functional indexes on virtual columns for index access. A JSON column without that CHECK accepts `'not even json'` silently, so dimension 10 grades a missing CHECK as a WARNING, not a SUGGESTION.

## Output Format

Always produce a findings table, then a verdict. Severity sort order : BLOCKER first, then WARNING, then SUGGESTION.

```
## Schema Review : <schema name or "supplied DDL">

| Dimension | Severity | Location | Problem | Fix | Skill ref |
|-----------|----------|----------|---------|-----|-----------|
| Storage engine | BLOCKER | `orders` | ENGINE=MyISAM on a transactional table | ALTER TABLE orders ENGINE=InnoDB | mariadb-core-storage-engines |
| Primary key | WARNING | `users.id` | CHAR(36) UUID-text PK bloats secondary indexes | Use BINARY(16) with UUID_TO_BIN(uuid, 1) | mariadb-impl-schema-design |
| ... | ... | ... | ... | ... | ... |

**Verdict** : FAIL (1 blocker, 1 warning) | Re-review after blockers are fixed.
```

Verdict rules :
- Any BLOCKER -> `FAIL`. The schema must not ship.
- Zero BLOCKER, one or more WARNING -> `PASS WITH WARNINGS`. May ship, but warnings carry documented cost.
- Zero BLOCKER and zero WARNING -> `PASS`.

## Cross-Reference Map

This skill orchestrates and references the following skills. When a finding needs more depth than the fix column allows, point the user at the cited skill :

- `mariadb-core-storage-engines` : engine selection, InnoDB vs MyISAM vs Aria.
- `mariadb-impl-schema-design` : PK strategy, normalization, multi-tenant pattern, naming.
- `mariadb-syntax-indexing` : leftmost-prefix rule, composite column order, FK indexes.
- `mariadb-errors-encoding-and-collation` : utf8mb3 vs utf8mb4, collation pitfalls.
- `mariadb-syntax-check-constraints` : CHECK syntax and JSON_VALID enforcement.
- `mariadb-syntax-json` : JSON-as-LONGTEXT, virtual-column functional indexes.
- `mariadb-syntax-sql-ddl` : column data types, VARCHAR vs TEXT, DECIMAL for money.

## When NOT to Use This Skill

- For tuning an individual slow query : use `mariadb-agents-query-optimizer`.
- For writing new DDL from scratch : use `mariadb-impl-schema-design` and `mariadb-syntax-sql-ddl` directly.
- This skill reviews structure, not data : it does not inspect row contents or run profiling.

## References

- `references/methods.md` : the complete 10-dimension review procedure, each check as a deterministic IF / THEN rule with severity and skill-ref.
- `references/examples.md` : 8 worked reviews including a MyISAM schema, a multi-tenant schema, a Frappe-style schema, a JSON-heavy schema, a clean schema, and a migration DDL.
- `references/anti-patterns.md` : 6 reviewer anti-patterns, why each one ships broken schemas, and the correct procedure.
