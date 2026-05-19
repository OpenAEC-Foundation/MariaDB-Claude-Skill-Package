# Masterplan : MariaDB

> Status : Phase 1 raw : pre-research
> Generated : 2026-05-19
> Refined : pending Phase 2 + Phase 3

## Scope

- Technology : MariaDB (NOT MySQL — focus on MariaDB-specific divergence)
- Versions : 10.6 LTS, 10.11 LTS, 11.x current, 12.x next
- Languages : SQL, Python (PyMySQL / mariadb-connector-python), Bash (admin scripts)
- Prefix : `mariadb`
- License : MIT
- Estimated total skills : 22 to 28 (definitive count post-Phase-3 refinement)

## Methodology Note

Raw inventory below is pre-research. Phase 2 deep research will reveal sub-topics, merges, drops, and splits. Phase 3 produces the definitive masterplan with refinement-decisions table and per-skill agent prompts.

## Identified Topics (raw, pre-research)

### Core (5 skills planned)

- `mariadb-core-architecture` : MariaDB process model (mysqld, threads, connection pools), storage-engine plug-in architecture, query execution pipeline (parser, optimizer, executor), in-memory caches (buffer pool, query cache, key cache), divergence from MySQL on threading and optimizer
- `mariadb-core-storage-engines` : InnoDB (default, ACID, row-level locking, MVCC) vs Aria (crash-safe MyISAM replacement, used by system tables) vs MyISAM (legacy, no transactions, table-locking) vs ColumnStore (analytical, columnar) vs Spider (sharding) vs Connect (federated). Decision tree for engine choice per workload.
- `mariadb-core-replication-model` : async vs semi-sync vs parallel-applier, GTID model, binary log formats (ROW, STATEMENT, MIXED), MariaDB-specific multi-source replication and Galera multi-master
- `mariadb-core-security-model` : authentication plug-ins (mysql_native_password, ed25519, gssapi, pam), GRANT model, role-based access (10.0.5+), encryption at rest (file-key-management plug-in), TLS for client + replication
- `mariadb-core-version-matrix` : LTS cadence, deprecated features per release, breaking changes 10.6 to 10.11 to 11.x, end-of-life dates, when to upgrade

### Syntax (8 skills planned)

- `mariadb-syntax-sql-dml` : INSERT, UPDATE, DELETE patterns, INSERT ... ON DUPLICATE KEY UPDATE, INSERT IGNORE, REPLACE, multi-row INSERT, RETURNING clause (10.5+), UPDATE with JOIN, DELETE with subquery
- `mariadb-syntax-sql-ddl` : CREATE TABLE patterns, ALTER TABLE algorithms (INPLACE, COPY, INSTANT), online DDL constraints, sequences (10.3+), invisible columns, computed columns (PERSISTENT vs VIRTUAL)
- `mariadb-syntax-indexing` : B-tree default, fulltext (InnoDB + Aria), spatial (SPATIAL INDEX, GIST), hash (in-memory only), composite-key column-order rule (leftmost prefix), functional indexes on JSON paths, descending indexes (10.8+), invisible indexes
- `mariadb-syntax-json` : JSON_VALID, JSON_EXTRACT, JSON_VALUE, JSON_QUERY, JSON_TABLE (10.6+), JSON_SET, JSON_MERGE_PATCH vs JSON_MERGE_PRESERVE, JSON path expressions, divergence from MySQL native JSON binary storage (MariaDB uses LONGTEXT with CHECK constraints)
- `mariadb-syntax-dynamic-columns` : MariaDB-unique COLUMN_CREATE / COLUMN_GET / COLUMN_LIST, blob-storage of nested key-value, when to use vs JSON, migration to JSON
- `mariadb-syntax-window-and-cte` : OVER() clauses, PARTITION BY, frame definitions (ROWS, RANGE, GROUPS in 10.7+), ranking functions (ROW_NUMBER, RANK, DENSE_RANK, NTILE), aggregates as window functions, recursive CTEs (10.2.2+), CTE materialization
- `mariadb-syntax-system-versioned-tables` : WITH SYSTEM VERSIONING (10.3+), FOR SYSTEM_TIME AS OF / BETWEEN / FROM TO / ALL, application-time periods (10.4+), bitemporal (combining system + application time, 10.5+), partitioning by system-time
- `mariadb-syntax-stored-routines` : CREATE PROCEDURE, CREATE FUNCTION, CREATE TRIGGER (BEFORE / AFTER on INSERT / UPDATE / DELETE), CREATE EVENT, CREATE VIEW (algorithm UNDEFINED / MERGE / TEMPTABLE), DEFINER vs INVOKER security context, error handling with DECLARE HANDLER

### Implementation (7 skills planned)

- `mariadb-impl-schema-design` : multi-tenant patterns (schema-per-tenant vs row-tenant-id), table-naming conventions (snake_case, table-prefix), choosing PK type (BIGINT auto-increment vs UUID vs ULID), normalization vs denormalization trade-offs, ERPNext / Frappe table-naming companion section
- `mariadb-impl-query-optimization` : EXPLAIN reading (id, select_type, table, type, possible_keys, key, key_len, ref, rows, Extra), EXPLAIN FORMAT=JSON, ANALYZE FORMAT=JSON (actual execution), index hints (USE / FORCE / IGNORE INDEX), optimizer_switch flags, optimizer_trace
- `mariadb-impl-replication-setup` : async setup (master + slave), semi-sync plug-in, parallel replication (slave_parallel_threads, slave_parallel_mode), GTID setup (gtid_strict_mode), multi-source replication (CHANGE MASTER ... FOR CHANNEL)
- `mariadb-impl-galera-cluster` : 3-node minimum, gcomm:// addresses, wsrep_provider, SST methods (mariabackup, rsync, mysqldump), state transfer flow, split-brain prevention with quorum, scaling read replicas off Galera
- `mariadb-impl-backup-restore` : mysqldump (logical, locks tables) vs mariabackup (physical, hot-backup), point-in-time recovery from binary logs, incremental backups with mariabackup, restoring single tables, backup verification, RPO/RTO planning
- `mariadb-impl-performance-tuning` : innodb_buffer_pool_size (60-80% RAM rule), query cache (removed in 10.0+ for write-heavy, kept for read-only), thread_pool, innodb_log_file_size, tmp_table_size, max_connections, table_open_cache, FILLFACTOR equivalent (innodb_fill_factor) for index page splits
- `mariadb-impl-migration-mysql-to-mariadb` : drop-in compatibility matrix per MySQL version (5.6, 5.7, 8.0, 8.4), syntax divergence (JSON, role syntax, default_authentication_plugin), upgrade path mysql_upgrade vs mariadb-upgrade, gotchas (timezone tables, mysql.user schema differences), benchmarks pre-and-post migration

### Errors (5 skills planned)

- `mariadb-errors-deadlocks` : InnoDB deadlock detection, reading SHOW ENGINE INNODB STATUS, lock-order discipline, gap locks vs record locks, deadlock prevention vs deadlock retry strategy at application layer, innodb_deadlock_detect tuning
- `mariadb-errors-replication-lag` : measuring lag (Seconds_Behind_Master is unreliable), pt-heartbeat, common causes (single-threaded applier on legacy, long transactions on master, IO bottleneck, large binlog events), fix with parallel-applier or row-based logging
- `mariadb-errors-galera-conflicts` : certification failures, deadlock errors only at COMMIT (Galera-specific), wsrep_local_cert_failures, application retry strategy on ER_LOCK_DEADLOCK from Galera, hot rows and how to avoid them in cluster
- `mariadb-errors-slow-queries` : slow query log analysis with mariadb-dumpslow / pt-query-digest, query profiling with SHOW PROFILE (deprecated) vs performance_schema, missing-index detection, sort-on-disk indicators in Extra column, file-sort vs index-sort
- `mariadb-errors-encoding-and-collation` : utf8 vs utf8mb4 trap (utf8 is 3-byte, not full UTF-8), collation suffixes (_ci, _cs, _bin), default-collation change in 10.6 to utf8mb4_general_ci, mixed-collation errors in JOINs, ALTER TABLE ... CONVERT TO CHARACTER SET utf8mb4 risks

### Agents (3 skills planned)

- `mariadb-agents-schema-reviewer` : validate proposed schema against indexing rules, engine choice rationale, normalization fitness for use case, naming-convention adherence, multi-tenant pattern detection
- `mariadb-agents-query-optimizer` : analyze a query + EXPLAIN output, propose index changes, rewrite to use covering index or eliminate sort, validate against MariaDB-specific optimizer flags
- `mariadb-agents-migration-validator` : MySQL-to-MariaDB compatibility check on SQL dump or schema, flag divergent syntax (JSON, default-auth plug-in, sql_mode), produce remediation list

## Estimated Skill Count

- Core : 5
- Syntax : 8
- Implementation : 7
- Errors : 5
- Agents : 3
- **Total estimate** : 28 (refinement in Phase 3 may merge / drop / split)

## Cross-Package Companion Section

This package directly impacts and is referenced by :

| Package | Relationship | Required Section |
|---------|-------------|------------------|
| Frappe / ERPNext | MariaDB is the default DB. Multi-tenant schema, table-prefix conventions, mariadb_socket auth | INDEX.md + README.md + dedicated bullet in `mariadb-impl-schema-design` |
| Docker | MariaDB containers (official + Bitnami), volume-mounts for datadir, Galera-in-Docker | Companion-link in README, no dedicated skill (deferred to Docker-pkg) |
| Nextcloud | Uses MariaDB / MySQL backend, requires utf8mb4, large_prefix InnoDB flags | Companion-link in README |

## Next : Phase 2 Deep Research

Dispatch 3 opus research-agents in parallel, each on a topic-cluster :

1. **Storage + Schema + SQL-syntax cluster** : engines, DDL, DML, indexing, sequences, computed columns, JSON, dynamic columns
2. **Advanced-SQL + Routines cluster** : window functions, CTEs, system-versioned tables, stored procedures, triggers, events, views, EXPLAIN, optimizer
3. **Ops cluster** : replication (async, semi-sync, parallel, GTID), Galera, backup / restore, performance tuning, security (auth, TLS, encryption at rest), MySQL-to-MariaDB migration, Frappe / ERPNext companion patterns

Each agent produces a section of `docs/research/vooronderzoek-mariadb.md` (>= 700 words per agent, total >= 2000 words). All sources verified via WebFetch against SOURCES.md approved URLs (mariadb.com/kb, mariadb.org, github.com/MariaDB/server). Last-Verified : 2026-05-19.

After Phase 2 : run Phase 3 (refinement) with user-checkpoint before Phase 4 + 5 (tmux-orchestration skill creation).
