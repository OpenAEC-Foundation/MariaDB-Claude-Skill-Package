# Sources : MariaDB Skill Package

## Approved Sources

All skill content MUST be verified against these approved sources. No unverified blog posts or AI-generated content.

### Primary Sources

| Source | URL | Type | Last Verified |
|--------|-----|------|---------------|
| MariaDB Knowledge Base | https://mariadb.com/kb/ | Official Documentation | 2026-05-19 |
| MariaDB Server GitHub | https://github.com/MariaDB/server | Source code (canonical) | 2026-05-19 |
| MariaDB Release Notes | https://mariadb.com/kb/en/release-notes/ | Official changelogs per version | 2026-05-19 |
| MariaDB.org Documentation | https://mariadb.org/documentation/ | Official Foundation docs | 2026-05-19 |

### Secondary Sources (use only when primary is insufficient)

| Source | URL | Type | Last Verified |
|--------|-----|------|---------------|
| Galera Cluster Documentation | https://galeracluster.com/library/documentation/ | Upstream Galera (Codership) | 2026-05-19 |
| MariaDB JIRA | https://jira.mariadb.org/ | Bug-and-feature tracker for version-specific behavior | 2026-05-19 |
| mariadb-corporation/MaxScale | https://github.com/mariadb-corporation/MaxScale | Proxy / load-balancer (referenced for routing patterns) | 2026-05-19 |

## Verification Rules

1. **Primary sources ONLY** : Official docs > source code > release notes
2. **NEVER use** : Random blog posts, unverified StackOverflow answers, AI-generated content without verification
3. **Version-check** : Ensure source matches target version (10.6-LTS, 10.11-LTS, 11.x, 12.x). MariaDB KB pages have a per-page version-applicability box : honour it.
4. **Date-check** : Note last verification date per source. Re-verify if older than 6 months.
5. **Cross-reference** : If KB is sparse or contradictory, verify against source code in MariaDB/server (sql/, storage/innobase/).
6. **WebFetch** : ALWAYS use WebFetch to verify against latest official documentation (per D-006)
7. **NEVER conflate with MySQL** : dev.mysql.com docs are NOT a valid MariaDB source. Common divergence : JSON storage format, default authentication plug-in, role syntax, sequence syntax.

## Source Addition Protocol

When discovering a new source during research :
1. Verify it's official (MariaDB Foundation, MariaDB Corporation, Codership for Galera) or maintained by core team
2. Add to appropriate table above
3. Set "Last Verified" to current date (YYYY-MM-DD)
4. Record in LESSONS.md if the source revealed significant insights or contradicted another source
