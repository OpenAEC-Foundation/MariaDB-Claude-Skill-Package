# INDEX : MariaDB Skill Package Catalog

## Overview

31 deterministic Claude skills for MariaDB 10.6 LTS, 10.11 LTS, 11.x, 12.x, across 5 categories.

## Summary

| Category | Skills | Focus |
|----------|:------:|-------|
| **core** | 6 | Architecture, engines, replication model, security, versions, defaults |
| **syntax** | 10 | SQL DML / DDL, indexing, JSON, dynamic columns, window functions, temporal, routines |
| **impl** | 7 | Schema design, query optimization, replication, Galera, backup, tuning, migration |
| **errors** | 5 | Deadlocks, replication lag, Galera conflicts, slow queries, encoding |
| **agents** | 3 | Schema review, query optimization, migration validation |
| **Total** | **31** | |

## Core Skills (6)

| Skill | Description |
|-------|-------------|
| `mariadb-core-architecture` | MariaDB process model, threading, query execution pipeline, plug-in architecture, in-memory caches. |
| `mariadb-core-storage-engines` | Choosing a storage engine (InnoDB, Aria, MyISAM, ColumnStore, Spider, CONNECT, MEMORY), engine-specific behavior, ACID guarantees. |
| `mariadb-core-replication-model` | Async, semi-sync, parallel-applier replication ; GTID model ; binary log formats. |
| `mariadb-core-security-model` | Users, GRANT, roles, authentication plug-ins, encryption at rest, TLS, mysql.global_priv. |
| `mariadb-core-version-matrix` | LTS cadence, EOL dates, feature-introduction matrix, breaking changes, upgrade path. |
| `mariadb-core-defaults-and-sql-modes` | sql_mode evolution, default charset and collation shifts, defaults that break apps on upgrade. |

## Syntax Skills (10)

| Skill | Description |
|-------|-------------|
| `mariadb-syntax-sql-dml` | INSERT, UPDATE, DELETE, REPLACE, upsert, RETURNING, multi-table DML. |
| `mariadb-syntax-sql-ddl` | CREATE / ALTER / DROP TABLE, ALTER algorithms, sequences, generated columns, partitioning. |
| `mariadb-syntax-indexing` | B-tree, full-text, spatial, hash, composite leftmost-prefix, descending and ignored indexes, JSON-path indexing. |
| `mariadb-syntax-json` | JSON functions, JSON-as-LONGTEXT alias, CHECK (JSON_VALID), JSON_TABLE, functional indexing. |
| `mariadb-syntax-dynamic-columns` | COLUMN_CREATE / GET / LIST family, dynamic columns vs JSON, migration to JSON. |
| `mariadb-syntax-check-constraints` | Column and table CHECK constraints, NULL semantics, JSON validation, CHECK vs trigger. |
| `mariadb-syntax-window-and-cte` | Window functions, frames, ranking and value functions, CTEs, recursive queries. |
| `mariadb-syntax-system-versioned-tables` | System versioning, FOR SYSTEM_TIME, application-time periods, bitemporal tables. |
| `mariadb-syntax-procedures-functions` | Stored procedures and functions, control flow, cursors, DECLARE HANDLER, SIGNAL / RESIGNAL. |
| `mariadb-syntax-triggers-events-views` | Triggers, the event scheduler, views, the materialized-view workaround. |

## Implementation Skills (7)

| Skill | Description |
|-------|-------------|
| `mariadb-impl-schema-design` | Primary-key choice, multi-tenant patterns, ROW_FORMAT, charset, Frappe / ERPNext naming. |
| `mariadb-impl-query-optimization` | EXPLAIN reading, ANALYZE FORMAT=JSON, index hints, optimizer_switch, optimizer_trace. |
| `mariadb-impl-replication-setup` | Async, semi-sync, parallel-applier setup, GTID, multi-source named connections. |
| `mariadb-impl-galera-cluster` | Galera multi-master cluster setup, SST methods, quorum, certification semantics. |
| `mariadb-impl-backup-restore` | mariabackup, mysqldump / mariadb-dump, point-in-time recovery, incremental chains. |
| `mariadb-impl-performance-tuning` | innodb_buffer_pool sizing, durability vs throughput, thread pool, IO capacity. |
| `mariadb-impl-migration-mysql-to-mariadb` | Compatibility matrix per MySQL version, syntax divergences, mariadb-upgrade. |

## Error Skills (5)

| Skill | Description |
|-------|-------------|
| `mariadb-errors-deadlocks` | ER_LOCK_DEADLOCK and ER_LOCK_WAIT_TIMEOUT, lock-order discipline, retry strategy. |
| `mariadb-errors-replication-lag` | Measuring lag accurately, parallel-applier tuning, root causes of cascading lag. |
| `mariadb-errors-galera-conflicts` | Certification failures, COMMIT-time deadlocks, hot rows, split-brain prevention. |
| `mariadb-errors-slow-queries` | Slow query log, pt-query-digest, performance_schema, missing-index detection. |
| `mariadb-errors-encoding-and-collation` | utf8 vs utf8mb4 trap, Illegal mix of collations, CONVERT TO risks, connection charset. |

## Agent Skills (3)

| Skill | Description |
|-------|-------------|
| `mariadb-agents-schema-reviewer` | Deterministic 10-dimension schema-review checklist with severity grading. |
| `mariadb-agents-query-optimizer` | 7-step query-optimization procedure from EXPLAIN to verified fix. |
| `mariadb-agents-migration-validator` | 10-dimension MySQL-dump pre-import scan with remediation checklist. |

## Dependency Flow

```
core  ->  syntax  ->  impl  ->  errors  ->  agents
```

Core skills are foundational and reference nothing. Syntax skills build on core. Implementation skills compose syntax. Error skills diagnose problems across impl and syntax. Agent skills orchestrate reviews and reference all other categories.

## Companion Cross-Technology Skills

MariaDB is the default database for Frappe / ERPNext, runs in Docker containers, and backs Nextcloud. For multi-technology stacks see :

- [Frappe Claude Skill Package](https://github.com/OpenAEC-Foundation/Frappe_Claude_Skill_Package)
- [Cross-Tech-AEC Claude Skill Package](https://github.com/OpenAEC-Foundation/Cross-Tech-AEC-Claude-Skill-Package)

The `mariadb-impl-schema-design` skill includes a dedicated Frappe / ERPNext table-naming companion section.

## Discovery

- npm-agentskills manifest : [package.json](package.json) `agents.skills[]`
- OpenAI Codex : [agents/openai.yaml](agents/openai.yaml)
- GitHub topic : `agentskills`

To regenerate the flat catalog from skill frontmatter :

```bash
node /path/to/Skill-Package-Workflow-Template/scripts/generate-index.js . --write
```
