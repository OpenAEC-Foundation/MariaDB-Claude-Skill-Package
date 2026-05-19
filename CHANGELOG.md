# Changelog

All notable changes to the MariaDB Skill Package.

Format follows [Keep a Changelog](https://keepachangelog.com/).

## [1.0.0] - 2026-05-20

### Added

- 31 deterministic Claude skills for MariaDB 10.6 LTS, 10.11 LTS, 11.x, 12.x across 5 categories :
  - **core** (6) : architecture, storage-engines, replication-model, security-model, version-matrix, defaults-and-sql-modes
  - **syntax** (10) : sql-dml, sql-ddl, indexing, json, dynamic-columns, check-constraints, window-and-cte, system-versioned-tables, procedures-functions, triggers-events-views
  - **impl** (7) : schema-design, query-optimization, replication-setup, galera-cluster, backup-restore, performance-tuning, migration-mysql-to-mariadb
  - **errors** (5) : deadlocks, replication-lag, galera-conflicts, slow-queries, encoding-and-collation
  - **agents** (3) : schema-reviewer, query-optimizer, migration-validator
- Each skill has SKILL.md (under 500 lines) plus three reference files (methods, examples, anti-patterns)
- All code WebFetch-verified against mariadb.com/kb and mariadb.com/docs/server
- Discovery manifests : package.json `agents.skills[]`, agents/openai.yaml (OpenAI Codex), INDEX.md catalog
- Companion cross-technology section for Frappe / ERPNext, Docker, Nextcloud
- Social preview banner, 8669-word vooronderzoek, 11-decision refined masterplan
- Compliance audit score 100%, functional sample-test 5/5 PASS

### Notes

- MariaDB-specific divergences from MySQL are called out throughout : JSON as LONGTEXT alias, GTID incompatibility, no GROUPS window frame, IGNORED vs INVISIBLE index keyword, query cache retained, semi-sync built-in since 10.3
