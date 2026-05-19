---
name: mariadb-agents-migration-validator
description: >
  Use when validating a MySQL SQL dump or schema before importing into MariaDB, checking a codebase for MySQL-specific syntax that breaks on MariaDB, or producing a migration remediation checklist.
  Prevents the common mistake of importing a MySQL 8 dump with JSON-binary assumptions, caching_sha2_password users, INVISIBLE INDEX syntax, or GTID-continuity expectations.
  Covers a deterministic migration-validation procedure : scan SQL dump or schema for MySQL-specific divergences, flag JSON-storage / GTID-format / auth-plugin / INVISIBLE-vs-IGNORED / sequence-syntax / sql_mode issues, grade severity, produce a remediation list with skill-references to mariadb-impl-migration-mysql-to-mariadb and others.
  Keywords: migration validation, validate MySQL dump, check schema before import, MySQL to MariaDB compatibility check, will this dump import, migration remediation, MySQL-specific syntax, caching_sha2_password, INVISIBLE INDEX, JSON binary, GTID, pre-migration check, is my dump compatible
license: MIT
compatibility: "Designed for Claude Code. Requires MariaDB 10.6-LTS, 10.11-LTS, 11.x, 12.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# mariadb-agents-migration-validator

A deterministic procedure for validating a MySQL SQL dump, schema, or codebase BEFORE importing into MariaDB. This is an ORCHESTRATION skill : it does not teach migration, it applies a fixed 10-dimension scan and produces a graded validation report. Every finding cites the canonical skill that explains the fix and the exact MySQL-vs-MariaDB divergence.

## Quick Reference

- Scan ALL 10 divergence dimensions in order. NEVER skip a dimension because the dump "looks standard".
- Grade every finding with one of three severities : BLOCKER, WARNING, SUGGESTION.
- **BLOCKER** : the dump will not import correctly or apps will break. Examples : `caching_sha2_password` users, `INVISIBLE INDEX` keyword, GTID-continuity expectation, `GROUPS` window frame, MySQL 8 functional-index syntax.
- **WARNING** : the dump imports but carries a known footgun. Examples : JSON columns (no enforcement after import), `sql_mode` default differences, non-InnoDB storage engines.
- **SUGGESTION** : a non-blocking improvement. Examples : `utf8` (utf8mb3) charset, AUTO_INCREMENT that could become a sequence.
- MariaDB JSON is a LONGTEXT alias, NOT native binary. Use `CHECK (JSON_VALID(col))` for structure ; use functional indexes on virtual columns for index access. A migrated JSON column has NO write-time validation until the CHECK is added. (D-010, L-005)
- Every finding cites the canonical skill that explains the fix. NEVER report a divergence without a skill-ref.
- Output a single findings table : finding, severity, location, MySQL-vs-MariaDB divergence, remediation step, skill-ref. Close with a verdict line.
- ALWAYS recommend running `mariadb-upgrade` exactly once AFTER the import, regardless of the verdict.
- A clean dump still gets a report : the table has zero rows and the verdict is `PASS`.

## The 10 Divergence Dimensions

| # | Dimension | What it flags | Severity | Canonical skill |
|---|-----------|---------------|----------|-----------------|
| 1 | JSON columns | every `JSON` column ; needs `CHECK (JSON_VALID(col))` post-import | WARNING | mariadb-syntax-json |
| 2 | Authentication | `caching_sha2_password` users ; no such plugin in MariaDB | BLOCKER | mariadb-core-security-model |
| 3 | GTID | any expectation of GTID continuity ; format incompatible | BLOCKER | mariadb-core-replication-model |
| 4 | INVISIBLE INDEX | MySQL 8 `INVISIBLE` keyword ; MariaDB uses `IGNORED` | BLOCKER | mariadb-syntax-indexing |
| 5 | sql_mode | MySQL 8 `sql_mode` defaults that differ | WARNING | mariadb-core-defaults-and-sql-modes |
| 6 | Sequences | MySQL has only AUTO_INCREMENT ; MariaDB `CREATE SEQUENCE` differs | SUGGESTION | mariadb-syntax-sql-ddl |
| 7 | Functional indexes | MySQL 8 expression-index syntax ; needs virtual column | BLOCKER | mariadb-syntax-indexing |
| 8 | Storage engine | non-InnoDB engines that may not exist on the target | WARNING | mariadb-core-storage-engines |
| 9 | Window / CTE | `GROUPS` window frame ; not in MariaDB | BLOCKER | mariadb-syntax-window-and-cte |
| 10 | Charset | `utf8` (utf8mb3) usage ; recommend `utf8mb4` | SUGGESTION | mariadb-errors-encoding-and-collation |

The full pattern-detection rule set per dimension (regex hint + IF condition THEN severity) is in `references/methods.md`.

## Validation Decision Tree

```
For the supplied dump / schema / codebase :

  1. Scan every column definition for the JSON type :
       JSON column found -> WARNING
         divergence : MySQL stores JSON as packed binary, MariaDB stores it as
                      LONGTEXT with NO write-time validation.
         remediation : after import, ADD CONSTRAINT ... CHECK (JSON_VALID(col)).
       -> ref mariadb-syntax-json

  2. Scan CREATE USER / ALTER USER / mysql.user rows for the auth plugin :
       IDENTIFIED WITH caching_sha2_password   -> BLOCKER
       IDENTIFIED WITH sha256_password         -> BLOCKER
         divergence : MariaDB ships ed25519, mysql_native_password, unix_socket ;
                      it does NOT ship caching_sha2_password or sha256_password.
         remediation : drop those users, re-create with ed25519.
       -> ref mariadb-core-security-model

  3. Inspect the migration intent for GTID continuity :
       plan expects GTID-tracked replication MySQL <-> MariaDB -> BLOCKER
         divergence : MySQL GTID is uuid:seqno ; MariaDB GTID is
                      domain-server-sequence. They are NOT interconvertible.
         remediation : one-way dump-and-load only ; if a cut-over replica is
                       needed, use binlog-positional replication (MASTER_USE_GTID=NO).
       -> ref mariadb-core-replication-model

  4. Scan DDL and app code for the INVISIBLE keyword on an index :
       INDEX ... INVISIBLE  /  ALTER INDEX idx INVISIBLE   -> BLOCKER
         divergence : MySQL 8 uses INVISIBLE ; MariaDB 10.6+ uses IGNORED.
         remediation : rewrite INVISIBLE to IGNORED.
       -> ref mariadb-syntax-indexing

  5. Compare the dump's expected sql_mode against MariaDB defaults :
       SET sql_mode containing MySQL-8-only modes, or relying on MySQL-8 default
       set differing from MariaDB -> WARNING
         remediation : review and set sql_mode explicitly post-import.
       -> ref mariadb-core-defaults-and-sql-modes

  6. Note AUTO_INCREMENT columns :
       AUTO_INCREMENT present -> SUGGESTION (usually no action)
         divergence : MariaDB additionally has CREATE SEQUENCE ; MySQL does not.
         remediation : keep AUTO_INCREMENT as-is ; convert deliberately later if wanted.
       -> ref mariadb-syntax-sql-ddl

  7. Scan index definitions for MySQL 8 functional-index syntax :
       INDEX / KEY with a parenthesised expression, e.g. (( col1 + col2 ))  -> BLOCKER
         divergence : MariaDB does NOT support inline expression indexes.
         remediation : add a generated (virtual) column, then index that column.
       -> ref mariadb-syntax-indexing

  8. Scan ENGINE= clauses :
       ENGINE not in {InnoDB, Aria, MyISAM, MEMORY, CSV, ARCHIVE} -> WARNING
       ENGINE=ROCKSDB / SPIDER / CONNECT -> WARNING (plugin must be installed)
         remediation : confirm the engine plugin exists on the target, or
                       convert to InnoDB.
       -> ref mariadb-core-storage-engines

  9. Scan window-function clauses for the frame unit :
       OVER ( ... GROUPS BETWEEN ... ) -> BLOCKER
         divergence : MariaDB supports ROWS and RANGE frames only ; GROUPS is a
                      MySQL 8 / PostgreSQL feature absent from MariaDB.
         remediation : rewrite the frame using ROWS or RANGE.
       -> ref mariadb-syntax-window-and-cte

 10. Scan CHARSET / COLLATE clauses :
       CHARACTER SET utf8 / utf8mb3 -> SUGGESTION
         remediation : convert to utf8mb4 to store the full Unicode range.
       -> ref mariadb-errors-encoding-and-collation
```

## JSON Storage Caveat

MariaDB JSON is a LONGTEXT alias, not native binary. Use `CHECK (JSON_VALID(col))` for structure ; use functional indexes on virtual columns for index access. A MySQL 8 dump validates JSON on every write ; once imported into MariaDB the same column accepts `'not even json'` silently. Dimension 1 therefore grades a JSON column as a WARNING, and the remediation step ALWAYS includes adding the CHECK.

## Output Format

Always produce a findings table, then a verdict, then the standing recommendation. Severity sort order : BLOCKER first, then WARNING, then SUGGESTION.

```
## Migration Validation : <dump name or "supplied schema">

| Finding | Severity | Location | MySQL vs MariaDB divergence | Remediation | Skill ref |
|---------|----------|----------|-----------------------------|-------------|-----------|
| caching_sha2_password user | BLOCKER | `CREATE USER 'app'@'%'` | MariaDB has no caching_sha2_password plugin | Drop user, re-create IDENTIFIED VIA ed25519 | mariadb-core-security-model |
| JSON column | WARNING | `orders.payload` | MariaDB stores JSON as LONGTEXT, no write validation | Add CHECK (JSON_VALID(payload)) after import | mariadb-syntax-json |
| ... | ... | ... | ... | ... | ... |

**Verdict** : FAIL (1 blocker, 1 warning) | Re-validate after blockers are remediated.

**Standing recommendation** : run `mariadb-upgrade` exactly once after the import completes. See mariadb-impl-migration-mysql-to-mariadb.
```

Verdict rules :
- Any BLOCKER -> `FAIL`. The dump must not be imported until blockers are remediated.
- Zero BLOCKER, one or more WARNING -> `PASS WITH WARNINGS`. May import, but warnings carry documented cost.
- Zero BLOCKER and zero WARNING -> `PASS`.
- In every case, append the standing recommendation to run `mariadb-upgrade` once after import.

## Cross-Reference Map

This skill orchestrates and references the following skills. When a finding needs more depth than the remediation column allows, point the user at the cited skill :

- `mariadb-impl-migration-mysql-to-mariadb` : the full migration workflow, in-place upgrade vs dump-and-restore, and the `mariadb-upgrade` step.
- `mariadb-syntax-json` : JSON-as-LONGTEXT, CHECK (JSON_VALID()), virtual-column functional indexes.
- `mariadb-core-security-model` : user model, ed25519, mysql.global_priv vs mysql.user.
- `mariadb-core-replication-model` : MariaDB GTID format, why MySQL GTID does not bridge.
- `mariadb-syntax-indexing` : IGNORED keyword, virtual-column indexes replacing functional indexes.
- `mariadb-core-defaults-and-sql-modes` : sql_mode defaults per version.
- `mariadb-syntax-sql-ddl` : AUTO_INCREMENT, CREATE SEQUENCE differences.
- `mariadb-core-storage-engines` : engine availability, InnoDB vs alternatives.
- `mariadb-syntax-window-and-cte` : supported window frames (ROWS, RANGE ; no GROUPS).
- `mariadb-errors-encoding-and-collation` : utf8mb3 vs utf8mb4.

## When NOT to Use This Skill

- For actually performing the migration : use `mariadb-impl-migration-mysql-to-mariadb`.
- For reviewing a native MariaDB schema (no MySQL origin) : use `mariadb-agents-schema-reviewer`.
- This skill scans for MySQL-vs-MariaDB divergence only ; it does not profile queries or tune the server.

## References

- `references/methods.md` : the complete 10-dimension scan procedure, each dimension as a deterministic IF / THEN rule with severity, regex / pattern hint, the exact MySQL-vs-MariaDB divergence, and skill-ref.
- `references/examples.md` : 8 worked validations including a MySQL 8 dump with JSON + caching_sha2_password, an INVISIBLE INDEX dump, a clean 5.7 dump, a GROUPS-frame view, a functional-index dump, and a remediation checklist.
- `references/anti-patterns.md` : 6 validator anti-patterns, why each one ships a broken import, and the correct procedure.

## Source Verification

- `https://mariadb.com/docs/release-notes/community-server/about/compatibility-and-differences/mariadb-vs-mysql-compatibility` : JSON stored as TEXT/BLOB (not packed binary), GTID incompatibility, sha256_password absence.
- `https://mariadb.com/kb/en/ignored-indexes/` : IGNORED keyword (10.6+), `ALTER TABLE ... ALTER INDEX key_name IGNORED` syntax, distinction from MySQL 8 INVISIBLE.
- Vooronderzoek `docs/research/vooronderzoek-mariadb.md` §6 (Migration MySQL to MariaDB), migration divergence table.
- LESSONS L-002 (no GROUPS frame), L-004 (GTID incompatible), L-005 (JSON LONGTEXT alias), L-006 (IGNORED vs INVISIBLE).
- DECISIONS D-010 (LONGTEXT-alias must appear in Quick Reference of every JSON-touching skill).
