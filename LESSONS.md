# Lessons Learned

Observations and findings captured during MariaDB skill package development.

---

## L-001 : MariaDB KB has been restructured to /docs paths (2026-05-19)

- **Context** : Phase 2 deep research, Cluster-3 agent.
- **Observation** : Several `mariadb.com/kb/en/<topic>/` URLs return 404 because pages have been moved to `mariadb.com/docs/<topic>`. Specific examples : encryption variables, some replication topics. Other KB URLs still work.
- **Impact** : Source verification flaky if you assume the historic KB URL scheme. WebFetch must follow redirects or try the `/docs/` alternative.
- **Rule** : When a `mariadb.com/kb/en/<page>/` WebFetch returns a stub or 404, retry with `mariadb.com/docs/server/<page>/` or use the LLM-friendly aggregated dump at `mariadb.com/docs/llms-full.txt` if available.
- **Affects** : SOURCES.md should accept both URL schemes ; Phase 4 topic-research must do the retry.

---

## L-002 : MariaDB window-function `GROUPS` frame is NOT supported (2026-05-19)

- **Context** : Phase 2 Cluster-2 research.
- **Observation** : Raw masterplan claimed `GROUPS` frame was added in 10.7+. Verified against KB `window-frames` page : MariaDB supports `ROWS` and `RANGE` only. `GROUPS` is a PostgreSQL / MySQL 8.0+ feature absent from MariaDB.
- **Impact** : `mariadb-syntax-window-and-cte` skill must drop GROUPS, and the description must include a Keywords entry "GROUPS frame not supported" to trigger on user-confusion.
- **Rule** : NEVER carry over MySQL window-function features into MariaDB skills without WebFetch verification against the MariaDB `window-frames` KB page.

---

## L-003 : Galera upstream docs are bot-blocked (2026-05-19)

- **Context** : Phase 2 Cluster-3 research.
- **Observation** : `galeracluster.com/library/documentation/*` returns HTTP 403 to WebFetch, presumably bot-filtering.
- **Impact** : Galera-specific verification (e.g. `pc.weight`, `wsrep_provider_options`) cannot rely on the upstream Codership docs via WebFetch.
- **Rule** : Use MariaDB KB Galera-cluster pages as the canonical source. Cross-reference via the MariaDB/server GitHub for wsrep variables when KB is sparse.

---

## L-004 : MariaDB GTID is INCOMPATIBLE with MySQL GTID (2026-05-19)

- **Context** : Phase 2 Cluster-3 migration research.
- **Observation** : MariaDB GTID uses `domain-server-sequence`. MySQL uses `uuid:seqno`. The schemes are not interconvertible. This means MariaDB-to-MySQL or MySQL-to-MariaDB replication with GTID is a one-shot dump-and-load, not a continuous GTID-tracked replication.
- **Impact** : Migration skill must call this out as the single biggest gotcha for users coming from MySQL. The Frappe / ERPNext companion pattern must default to binlog-position replication, not GTID, when bridging the two.
- **Rule** : Migration skill description Keywords must include "GTID incompatible", "MySQL replication", "one-way migration".

---

## L-006 : MariaDB uses `IGNORED INDEX`, not MySQL 8 `INVISIBLE INDEX` (2026-05-19)

- **Context** : Batch B-03 S-03 indexing skill, WebFetch against mariadb.com/kb/en/ignored-indexes/.
- **Observation** : Vooronderzoek + raw masterplan + agent-prompt said "invisible indexes (10.6+)". Actual MariaDB keyword is `IGNORED` (`ALTER TABLE t ALTER INDEX idx IGNORED`). MySQL 8 uses `INVISIBLE` for the same feature.
- **Impact** : Users migrating from MySQL 8 typing `INVISIBLE` get a syntax error. Skill explicitly translates the keyword and includes anti-pattern.
- **Rule** : When a feature exists "in both MariaDB and MySQL" always WebFetch-verify the keyword name. Multiple identical-concept features have different keyword spellings.

---

## L-008 : Query cache is NOT removed in 11.0+, still present default-OFF (2026-05-19)

- **Context** : Batch B-08 I-06 performance-tuning skill, WebFetch against mariadb.com/docs/server/server-management/variables-and-modes/server-system-variables.
- **Observation** : Multiple sources (including initial vooronderzoek) claimed query cache "removed completely in 11.0+". WebFetch verification shows query_cache_type and query_cache_size remain in 11.x. Status : deprecated default-OFF since 10.1.7+, but NOT removed.
- **Impact** : Earlier agent prompts and the masterplan claimed "removed completely". Correction propagated to I-06 performance-tuning skill (default OFF, still present).
- **Related** : innodb_buffer_pool_instances IS REMOVED in 10.6.0+ (not just deprecated). Tuning skills must reflect both nuances.
- **Rule** : "Removed" vs "deprecated default-off" require WebFetch verification per version ; KB phrasing varies. Always check the per-variable KB page rather than rely on summary tables.

---

## L-007 : `UPDATE RETURNING` is 13.0+, NOT 10.5+ (2026-05-19)

- **Context** : Batch B-03 S-01 DML skill, WebFetch against mariadb.com/kb/en/update/.
- **Observation** : RETURNING-clause introduction is per-statement : DELETE RETURNING in 10.0+, INSERT RETURNING in 10.5+, UPDATE RETURNING only in 13.0+. Masterplan + prompt assumed all three landed in 10.5+.
- **Impact** : Including UPDATE RETURNING examples for 10.6 / 10.11 / 11.x / 12.x is wrong : users will get syntax error. Skill scopes UPDATE RETURNING to 13.0+ with a callout that current LTS does not support it.
- **Rule** : NEVER assume a clause's introduction-version is uniform across statement types. Verify per-statement KB page.

---

## L-005 : MariaDB JSON is LONGTEXT, not binary (2026-05-19)

- **Context** : Phase 2 Cluster-1 research.
- **Observation** : MariaDB's `JSON` type is an alias for LONGTEXT with a CHECK constraint hook. No native binary storage like MySQL 5.7.8+. Indexing JSON requires virtual columns with functional indexes.
- **Impact** : Cross-cuts `mariadb-syntax-json`, `mariadb-impl-schema-design`, `mariadb-impl-migration-mysql-to-mariadb`, `mariadb-errors-encoding-and-collation`.
- **Rule** : Every JSON-related skill MUST repeat the LONGTEXT-alias fact in its `## Quick Reference`, not just in the migration skill. This is the single most consequential MySQL divergence.
