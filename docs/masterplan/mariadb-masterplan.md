# Masterplan : MariaDB

> Status : Phase 3 refined (pending user-checkpoint)
> Generated : 2026-05-19 (Phase 1 raw)
> Refined : 2026-05-19 (Phase 3, post-vooronderzoek)

## Scope

- Technology : MariaDB (NOT MySQL, focus on MariaDB-specific divergence)
- Versions : 10.6 LTS, 10.11 LTS, 11.x current, 12.x next
- Languages : SQL, Python (PyMySQL / mariadb-connector-python), Bash (admin scripts)
- Prefix : `mariadb`
- License : MIT
- Final skill count : 30 (was 28 raw, +2 from Phase 2 research findings)

## Refinement Decisions

| ID | Decision | Reason | Source |
|----|----------|--------|--------|
| D-01 | SPLIT `mariadb-syntax-stored-routines` into two skills : `mariadb-syntax-procedures-functions` and `mariadb-syntax-triggers-events-views` | Five disparate features cannot fit in <500 lines per D-003. Procedures + functions share semantics ; triggers + events + views share lifecycle | vooronderzoek §Cluster-2 |
| D-02 | ADD `mariadb-syntax-check-constraints` | CHECK constraints since 10.2.1, MariaDB precedes MySQL 8 by years, dedicated skill needed for cross-skill reference from JSON, schema-design, errors-encoding | vooronderzoek L-005 + sub-topics |
| D-03 | ADD `mariadb-core-defaults-and-sql-modes` | sql_mode and default-charset behavior changes between LTS releases ; gap not covered by version-matrix skill (which is about features, not defaults) | vooronderzoek §Cluster-3 migration gotchas |
| D-04 | DROP `GROUPS` frame coverage from `mariadb-syntax-window-and-cte` | MariaDB does not support GROUPS frame (PostgreSQL / MySQL 8 only). Skill must call out the divergence explicitly | vooronderzoek L-002 |
| D-05 | SCOPE-REDUCE `mariadb-impl-replication-setup` : semi-sync is built-in from 10.3+, no plug-in install | vooronderzoek Cluster-3 finding | vooronderzoek §Replication |
| D-06 | EXPAND `mariadb-impl-migration-mysql-to-mariadb` Keywords with "GTID incompatible", "MySQL replication one-way", "JSON not binary" | These are the three top reasons migrations break ; symptom-based keywords trigger user-prompts that don't say "migration" | vooronderzoek L-004, L-005 |
| D-07 | KEEP `mariadb-impl-galera-cluster` separate from replication-setup | Different paradigm (multi-master vs primary-replica), Galera certification semantics differ enough to warrant separate skill | vooronderzoek §Galera |
| D-08 | KEEP `mariadb-syntax-window-and-cte` combined (window functions + CTEs in one skill) | Both are SELECT-side query-shaping features, share dependency on optimizer ; merging keeps skill cohesive | raw masterplan + vooronderzoek |
| D-09 | DROP estimate of "materialized views" from any skill ; document the missing-feature as anti-pattern + workaround in `mariadb-syntax-triggers-events-views` | MariaDB does not support materialized views ; workaround is CREATE TABLE AS SELECT + EVENT-refresh | vooronderzoek L-002 (materialized views) |
| D-10 | KEY decision : all JSON-related skills MUST repeat "JSON is LONGTEXT alias" fact in Quick Reference, not only in migration skill | Cross-cutting consequence of MariaDB JSON storage divergence ; one-place documentation creates "I missed this" failures | vooronderzoek L-005 |

## Final Skill Inventory (30 skills)

### Core (6 skills) -- was 5, +1 from D-03

| ID | Skill | Scope |
|----|-------|-------|
| C-01 | `mariadb-core-architecture` | Process model (mysqld, threads, connection pools), plug-in architecture, query-execution pipeline, in-memory caches |
| C-02 | `mariadb-core-storage-engines` | Engine matrix (InnoDB / Aria / MyISAM / ColumnStore / Spider / CONNECT / MEMORY), decision tree per workload |
| C-03 | `mariadb-core-replication-model` | Async vs semi-sync vs parallel-applier, GTID model (MariaDB format vs MySQL format), binlog formats |
| C-04 | `mariadb-core-security-model` | Auth plug-ins (mysql_native_password / ed25519 / gssapi / pam / unix_socket), GRANT, roles, encryption at rest, TLS |
| C-05 | `mariadb-core-version-matrix` | LTS cadence, deprecated-features per release, breaking changes 10.6 to 10.11 to 11.x, EOL dates |
| C-06 | **`mariadb-core-defaults-and-sql-modes`** (NEW from D-03) | Default sql_mode per version, default-charset shift to utf8mb4 in 10.6, default authentication plug-in shift, default storage engine, defaults that break apps on upgrade |

### Syntax (10 skills) -- was 8, +2 from D-01 split and D-02 add

| ID | Skill | Scope |
|----|-------|-------|
| S-01 | `mariadb-syntax-sql-dml` | INSERT / UPDATE / DELETE patterns, ON DUPLICATE KEY UPDATE, INSERT IGNORE, REPLACE, RETURNING (10.5+), multi-table UPDATE / DELETE |
| S-02 | `mariadb-syntax-sql-ddl` | CREATE / ALTER / DROP TABLE, ALTER algorithms (INPLACE, COPY, INSTANT, NOCOPY), atomic DDL (10.6+), sequences (10.3+), invisible / virtual / persistent columns |
| S-03 | `mariadb-syntax-indexing` | B-tree, full-text, spatial, hash, composite leftmost-prefix, prefix indexes, descending (10.8+), invisible (10.6+), functional on JSON |
| S-04 | `mariadb-syntax-json` | JSON_VALIDATE / EXTRACT / VALUE / QUERY / TABLE (10.6+) / SET / MERGE_PATCH vs MERGE_PRESERVE, path expressions, **LONGTEXT-alias caveat (D-10)** |
| S-05 | `mariadb-syntax-dynamic-columns` | COLUMN_CREATE / GET / LIST / ADD / DELETE / EXISTS, blob storage, when to choose over JSON, migration to JSON |
| S-06 | `mariadb-syntax-window-and-cte` | OVER() with ROWS / RANGE (no GROUPS per D-04), ranking, aggregate-window functions, recursive CTEs (10.2.2+), max_recursive_iterations |
| S-07 | `mariadb-syntax-system-versioned-tables` | WITH SYSTEM VERSIONING (10.3+), FOR SYSTEM_TIME, application-time periods (10.4+), bitemporal (10.5+), partitioning by system-time |
| S-08 | **`mariadb-syntax-procedures-functions`** (from D-01 split) | CREATE PROCEDURE, CREATE FUNCTION, parameters, control flow, error handling with DECLARE HANDLER, SIGNAL / RESIGNAL, deterministic vs not-deterministic |
| S-09 | **`mariadb-syntax-triggers-events-views`** (from D-01 split) | CREATE TRIGGER with FOLLOWS / PRECEDES, CREATE EVENT scheduler, CREATE VIEW algorithms (UNDEFINED / MERGE / TEMPTABLE), DEFINER vs INVOKER, materialized-view workaround |
| S-10 | **`mariadb-syntax-check-constraints`** (NEW from D-02) | CHECK constraints since 10.2.1, table-level vs column-level, NULL handling, interaction with JSON_VALID, performance impact, cross-skill anchor for JSON / schema-design / errors |

### Implementation (7 skills) -- unchanged

| ID | Skill | Scope |
|----|-------|-------|
| I-01 | `mariadb-impl-schema-design` | Multi-tenant patterns, PK type choice, ROW_FORMAT, charset defaults, FK constraints, Frappe companion sub-section |
| I-02 | `mariadb-impl-query-optimization` | EXPLAIN reading, FORMAT=JSON, ANALYZE FORMAT=JSON, index hints, optimizer_switch, optimizer_trace |
| I-03 | `mariadb-impl-replication-setup` | Async, semi-sync (built-in 10.3+ per D-05), parallel-applier, GTID, multi-source |
| I-04 | `mariadb-impl-galera-cluster` | 3-node minimum, wsrep_provider, SST via mariabackup, IST, weighted quorum, certification semantics |
| I-05 | `mariadb-impl-backup-restore` | mysqldump / mariadb-dump (10.5+), mariabackup, point-in-time recovery, incremental chain |
| I-06 | `mariadb-impl-performance-tuning` | innodb_buffer_pool_size (incl. chunk_size deprecation 10.11.12+), log file size, flush_log_at_trx_commit, io_capacity, query cache history |
| I-07 | `mariadb-impl-migration-mysql-to-mariadb` | Compatibility matrix per MySQL version, JSON-format gotcha, GTID-format gotcha, default-auth gotcha, mysql.user -> mysql.global_priv, mariadb-upgrade |

### Errors (5 skills) -- unchanged

| ID | Skill | Scope |
|----|-------|-------|
| E-01 | `mariadb-errors-deadlocks` | InnoDB deadlock detection, SHOW ENGINE INNODB STATUS, lock-order discipline, gap vs record locks, app-layer retry |
| E-02 | `mariadb-errors-replication-lag` | Measuring lag (Seconds_Behind_Master unreliable), pt-heartbeat, root-causes, parallel-applier fix |
| E-03 | `mariadb-errors-galera-conflicts` | Certification failures, deadlock at COMMIT, wsrep_local_cert_failures, hot rows |
| E-04 | `mariadb-errors-slow-queries` | Slow query log analysis with mariadb-dumpslow / pt-query-digest, performance_schema, missing-index detection, file-sort indicators |
| E-05 | `mariadb-errors-encoding-and-collation` | utf8 vs utf8mb4 (3-byte trap), collation suffixes, utf8mb4_uca1400 family, mixed-collation JOIN failures, ALTER CONVERT TO risk |

### Agents (3 skills) -- unchanged

| ID | Skill | Scope |
|----|-------|-------|
| A-01 | `mariadb-agents-schema-reviewer` | Validate schema against indexing, engine choice, normalization fit, naming conventions, multi-tenant detection |
| A-02 | `mariadb-agents-query-optimizer` | Analyze query + EXPLAIN, propose index changes, rewrite for covering-index, validate against MariaDB optimizer flags |
| A-03 | `mariadb-agents-migration-validator` | MySQL-to-MariaDB compatibility check on SQL dump or schema, flag divergent syntax, produce remediation list |

## Execution Plan : Batches

Batch size = 3 skills (tmux-orchestration optimum). Dependency chain : core -> syntax -> impl -> errors -> agents. File-scope per worker is bindable to one skill ; no two skills share a file.

| Batch | Skills | Count | Dependencies | Notes |
|-------|--------|-------|--------------|-------|
| B-01 | C-01 architecture, C-02 storage-engines, C-03 replication-model | 3 | none | foundational ; everything else references these |
| B-02 | C-04 security-model, C-05 version-matrix, C-06 defaults-and-sql-modes | 3 | B-01 | finishes core layer |
| B-03 | S-01 sql-dml, S-02 sql-ddl, S-03 indexing | 3 | B-01 + B-02 | basic SQL surface |
| B-04 | S-04 json, S-05 dynamic-columns, S-10 check-constraints | 3 | B-03 | JSON / data-validation cluster |
| B-05 | S-06 window-and-cte, S-07 system-versioned-tables, S-08 procedures-functions | 3 | B-03 | advanced SQL cluster |
| B-06 | S-09 triggers-events-views, I-01 schema-design, I-02 query-optimization | 3 | B-03 + B-04 + B-05 | first impl layer + final syntax skill |
| B-07 | I-03 replication-setup, I-04 galera-cluster, I-05 backup-restore | 3 | B-01 (replication-model) + B-02 | ops cluster part 1 |
| B-08 | I-06 performance-tuning, I-07 migration-mysql-to-mariadb, E-01 deadlocks | 3 | B-01..B-07 | tuning + migration + first error skill |
| B-09 | E-02 replication-lag, E-03 galera-conflicts, E-04 slow-queries | 3 | B-07 + B-08 | error skills depending on replication / Galera / tuning |
| B-10 | E-05 encoding-collation, A-01 schema-reviewer, A-02 query-optimizer | 3 | ALL syntax + impl | encoding-collation + first two agents |
| B-11 | A-03 migration-validator (single-skill batch) | 1 | ALL | trailing agent skill, can run solo |

Total : 11 batches, 30 skills. Approx duration : 11 batches x ~15-20 min per batch = ~3 hours wall-clock in Phase 5.

## Per-Skill Agent Prompts

Each skill's agent-prompt-block has been deferred to post-user-checkpoint to keep this masterplan reviewable. After Phase 3 user-go, one opus agent expands the per-skill prompts into `docs/masterplan/agent-prompts/<skill-id>.md` (30 files), following the template below. Phase 4 + 5 (tmux-orchestration) consume those files directly.

### Per-Skill Prompt Template

```
Workspace : /home/freek/GitHub/MariaDB-Claude-Skill-Package/
Output dir : skills/source/<category>/mariadb-<category>-<topic>/
Files to create :
  - SKILL.md (max 500 lines, YAML folded scalar >, "Use when..." opener, Keywords with technical + symptom + plain-language terms)
  - references/methods.md
  - references/examples.md
  - references/anti-patterns.md

Research input :
  - docs/research/vooronderzoek-mariadb.md (cluster sections relevant to this skill)
  - docs/research/topic-research/mariadb-<category>-<topic>-research.md (Phase 4 produced, if applicable)

Scope :
  - <scope bullet 1 from masterplan>
  - <scope bullet 2>
  - ...

MUST INCLUDE Quick Reference :
  - <MariaDB-only feature flags for this topic>
  - <Diverges-from-MySQL warnings if applicable>
  - <D-10 LONGTEXT-alias repetition if JSON-related>

Quality rules :
  - English only (D-001)
  - License : MIT in frontmatter
  - compatibility : "Designed for Claude Code. Requires MariaDB 10.6-LTS, 10.11-LTS, 11.x, 12.x."
  - Section headings with `:` (NEVER em-dash, see global typografie rule)
  - Deterministic language : ALWAYS X / NEVER Y (NEVER "you might want to")
  - WebFetch all code-snippets against approved SOURCES.md URLs
  - Verify against vooronderzoek-mariadb.md citations
  - Version-annotate every example (`-- 10.6+` style)

Quality gate (validators must exit 0) :
  - node $TEMPLATE/scripts/validate-frontmatter.js <skill-pad>
  - node $TEMPLATE/scripts/validate-language.js <skill-pad>
  - node $TEMPLATE/scripts/validate-line-count.js <skill-pad>
  - node $TEMPLATE/scripts/validate-structure.js <skill-pad>
  - node $TEMPLATE/scripts/validate-emdash.js <skill-pad>
```

### Example Per-Skill Prompt : S-04 `mariadb-syntax-json`

```
Workspace : /home/freek/GitHub/MariaDB-Claude-Skill-Package/
Output dir : skills/source/syntax/mariadb-syntax-json/
Files to create :
  - SKILL.md
  - references/methods.md  (every JSON_* function with signature, version, KB-link)
  - references/examples.md (10+ working examples : create with CHECK, JSON_TABLE, functional index, JSON_MERGE_*)
  - references/anti-patterns.md (5+ real anti-patterns from vooronderzoek Cluster-1)

Research input :
  - docs/research/vooronderzoek-mariadb.md, Cluster-1 §6 (JSON section), §Anti-patterns
  - docs/research/topic-research/mariadb-syntax-json-research.md (Phase 4 produced)

Scope :
  - **THE LONGTEXT-alias caveat** : MariaDB JSON type is LONGTEXT, not binary like MySQL 5.7.8+. This appears in Quick Reference, MUST be repeated in every example introducing a new function.
  - JSON_VALID, JSON_EXTRACT, JSON_VALUE, JSON_QUERY (10.2.3+), JSON_TABLE (10.6+)
  - JSON_SET, JSON_REPLACE, JSON_INSERT, JSON_REMOVE
  - JSON_MERGE_PATCH (RFC 7396 merge) vs JSON_MERGE_PRESERVE (concatenating arrays). JSON_MERGE is deprecated alias for PRESERVE.
  - JSON path expressions ($.key, $.arr[0], $..wildcard)
  - CHECK (JSON_VALID(col)) as the only structural validation
  - Functional indexing on JSON via virtual / persistent columns
  - JSON_TABLE for converting JSON arrays to relational rows
  - Performance notes : no native-binary parsing, every JSON_EXTRACT re-parses LONGTEXT

Quick Reference (mandatory section) :
  - JSON in MariaDB = LONGTEXT alias + CHECK (JSON_VALID(col)) for structure
  - For native binary JSON, use MySQL not MariaDB
  - Indexing requires functional indexes via virtual columns
  - JSON_VALUE returns scalar, JSON_QUERY returns JSON sub-document
  - JSON_TABLE (10.6+) converts JSON to rows

Diverges from MySQL :
  - Storage : LONGTEXT vs binary
  - JSON_TABLE syntax matches MySQL but underlying re-parsing differs
  - No JSON_SCHEMA_VALID built-in (use CHECK with custom logic)

Anti-patterns from vooronderzoek Cluster-1 §Anti-patterns :
  - JSON column without CHECK (JSON_VALID(col)) accepts arbitrary text
  - Querying JSON_EXTRACT(col, '$.path') in WHERE without functional index = full scan
  - Assuming JSON_EXTRACT returns scalar (it returns JSON, use JSON_VALUE for scalar)
  - Using JSON_MERGE (deprecated) instead of explicit JSON_MERGE_PATCH or JSON_MERGE_PRESERVE
  - Storing huge JSON blobs without considering LONGTEXT-vs-row-size impact

Quality rules : as per template above.

Quality gate : all 5 validators exit 0.
```

## Next : User Checkpoint -> Phase 4 + 5

After this masterplan is approved by user (Phase 3 user-checkpoint per BOOTSTRAP-RUNBOOK §5.4) :

1. Expand per-skill agent-prompts to `docs/masterplan/agent-prompts/<skill-id>.md` (30 files), one opus agent in single pass.
2. Phase 4 : per batch (3 skills), dispatch 3 in-process opus topic-research agents to write `docs/research/topic-research/<skill-name>-research.md`. Wait for all 3 done.
3. Phase 5 : invoke tmux-orchestration skill, 3 workers with `skill-builder` role, file-scope per worker, quality-gate every reply. Run 11 batches sequentially.
4. After all batches done : STOP for user-go on Phase 6 + 7.

Skip-criteria for Phase 4 topic-research (Docker L-001) : if vooronderzoek section for a skill is dense enough (>=300 words + 5+ citations), skip the topic-research dispatch for that skill and reference the vooronderzoek section directly in the worker prompt. Documented per-batch in DECISIONS.md.
