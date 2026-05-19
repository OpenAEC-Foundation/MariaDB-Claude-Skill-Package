---
name: mariadb-core-version-matrix
description: >
  Use when choosing which MariaDB version to install, upgrading between LTS releases,
  reasoning about feature availability, or interpreting EOL dates.
  Prevents the common mistake of assuming a feature works in 10.6 when it landed in 10.11,
  or running production on a non-LTS interim release.
  Covers LTS cadence (10.6 / 10.11 / 11.4 / 11.8 / next LTS), interim releases, EOL dates,
  breaking changes 10.6 to 10.11 to 11.x, feature-introduction matrix for top-30 features,
  upgrade-path with mariadb-upgrade.
  Keywords: mariadb version, LTS, 10.6, 10.11, 11.4, 11.8, 12, end of life, EOL,
  breaking change, feature matrix, when was X added, upgrade mariadb, mariadb-upgrade,
  mariadb_upgrade, version compatibility, jump version, which mariadb should I use,
  is feature X available, my version is old, support expired, production version
license: MIT
compatibility: "Designed for Claude Code. Requires MariaDB 10.6-LTS, 10.11-LTS, 11.x, 12.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# MariaDB Core : Version Matrix

Authoritative guide to MariaDB version selection, feature-availability, EOL dates and
upgrade procedure. Use this skill whenever a question depends on "is feature X in version
Y" or "should I run 10.11 or 11.4 in production".

## Quick Reference

ALWAYS pick an LTS release for production : 10.6 / 10.11 / 11.4 / 11.8 / next.

ALWAYS treat interim releases (10.x.y between LTS, 11.0, 11.1, 11.2, 11.3, 11.5, 11.6,
11.7, 12.0) as short-support, dev-only.

ALWAYS run `mariadb-upgrade` after every binary swap (in-place upgrade).

ALWAYS upgrade one major version at a time. NEVER jump 10.6 directly to 11.4 or 11.8.
Path is 10.6 to 10.11, then 10.11 to 11.4, then 11.4 to 11.8.

ALWAYS query `SELECT VERSION();` on the live server before asserting a feature works.
Distro version-labels (`apt-cache policy mariadb-server`) lag the real server VERSION().

ALWAYS check the per-feature row in `references/methods.md` before writing code that uses
a feature added after 10.6 ; many production fleets are still on 10.6 LTS through
mid-2026.

NEVER trust dev.mysql.com docs for MariaDB feature availability. MariaDB and MySQL
diverged after MariaDB 5.5, and several features (sequences, system-versioned tables,
RETURNING, dynamic columns, application-time periods, INVISIBLE columns, IGNORED
indexes, MariaDB JSON-as-LONGTEXT) are MariaDB-only or differ semantically.

NEVER run `mariadb-upgrade` twice in a row. It is idempotent but the second pass logs
noise that masks real errors from the first.

NEVER run a beta or RC of the next-LTS in production. Wait for the LTS GA tag.

NEVER mix MariaDB major versions inside a single Galera cluster. The cluster-protocol
guarantees behavior only within one major series.

## LTS Cadence at a Glance

| Version | GA Date | Community EOL | Status | When to pick |
|---------|---------|---------------|--------|--------------|
| 10.6 | 2021-07-06 | 2026-07-06 | LTS, last 12 months | Frappe v14/v15 baseline ; legacy Frappe ; mature 10.x deployments |
| 10.11 | 2023-02-16 | 2028-02-16 | LTS, active | Conservative production default in 2025-2026 |
| 11.4 | 2024-05-29 | 2029-05-29 | LTS, active | Modern production default in 2026+ |
| 11.8 | 2025-06-04 | 2028-06-04 | LTS, newest | New deployments ; Frappe v16 baseline |
| 12.x | TBD | TBD | Interim until LTS | Dev only ; wait for the LTS designation |

Each LTS gets ~5 years of community support. Enterprise and Extended support extend
beyond that ; use the source URLs in `references/methods.md` for exact dates per tier.

Interim releases (10.7, 10.8, 10.9, 10.10 ; 11.0, 11.1, 11.2, 11.3, 11.5, 11.6, 11.7 ;
12.0) get ~1 year of community support each. They are testbeds for the next LTS, not
production targets.

## Decision Trees

### Which LTS version should I install in 2026?

```
New deployment with no constraints?
  -> 11.4 LTS (conservative) or 11.8 LTS (newest)

Running Frappe v14 or v15?
  -> 10.6 LTS minimum, 10.11 LTS recommended

Running Frappe v16 or develop?
  -> 11.8 LTS (hard floor)

Migrating from MySQL 5.7/8.0?
  -> 10.11 LTS (drop-in compatibility path)
  -> 11.4 or 11.8 only after read-replica burn-in

Legacy 10.4 or 10.5 in production today?
  -> Already EOL ; upgrade to 10.6 ASAP, then to 10.11 within 12 months
```

### Is feature X available in my MariaDB version?

```
Run SELECT VERSION(); on the live server (not your laptop, not docker-compose)
Look up X in references/methods.md "Feature-Introduction Matrix"
Cell shows first version that ships the feature

VERSION() >= matrix-cell ? -> feature available
VERSION() <  matrix-cell ? -> feature absent ; do NOT generate code that uses it

For runtime detection (programmatic), do
  SELECT @@version AS v, @@version_compile_os AS os;
or for specific feature flags
  SELECT @@have_query_cache; (returns YES/NO)
```

### Should I upgrade from 10.6 to 11.4 in one step?

NEVER. Always one major at a time. Path :

```
10.6 LTS -> 10.11 LTS  (run mariadb-upgrade)
10.11 LTS -> 11.4 LTS  (run mariadb-upgrade)
11.4 LTS -> 11.8 LTS   (run mariadb-upgrade)
```

Reasons : mysql.global_priv schema, default authentication plugin, default collation
and tool renames each shift between majors. Multi-step jumps stack incompatibilities
the upgrade tool was not designed to resolve in a single pass.

### In-place upgrade vs dump-and-restore : which one?

```
Same major series (e.g. 10.6.18 -> 10.6.21)?
  -> In-place. Stop server, install new binaries, start server, run mariadb-upgrade.

One major step (e.g. 10.6 -> 10.11)?
  -> In-place. Same procedure. Schedule downtime ; mariadb-upgrade may take minutes
     on large mysql.* system tables.

Two-or-more major steps (e.g. 10.6 -> 11.4)?
  -> Do it as a sequence of one-step in-place upgrades, OR
  -> Dump-and-restore from 10.6 directly into a fresh 11.4 install. The dump path
     avoids cumulative system-table cruft but costs full downtime equal to dump+restore.

Cross-cluster (Galera, replication)?
  -> Rolling : upgrade one node at a time, but only within one major series.
     Cross-major requires a full cluster shutdown.
```

## Patterns

### Pattern 1 : Detect the running version before code-generation

When generating SQL that might depend on a version-specific feature, capture the
running version first :

```sql
-- Run on the target server, never assume
SELECT
  VERSION()                AS full_version,    -- e.g. 11.4.4-MariaDB
  @@version_compile_os     AS os,              -- e.g. Linux
  @@innodb_version         AS innodb_version;
```

Then compare against the Feature-Introduction Matrix in `references/methods.md` and
emit code only for features present in that version.

### Pattern 2 : In-place binary upgrade

```bash
# 1) Backup first (mandatory, never skip)
mariadb-dump --all-databases --single-transaction --routines --triggers --events \
  --master-data=2 > /backup/pre-upgrade-$(date +%F).sql

# 2) Stop the old server
systemctl stop mariadb

# 3) Install the new binaries (distro-specific)
apt-get install mariadb-server=10.11.*    # or 11.4.*, 11.8.*

# 4) Start the new server
systemctl start mariadb

# 5) Run the upgrade tool ONCE
mariadb-upgrade --user=root --password
# (10.5+ binary ; on 10.4 and older it is named mysql_upgrade)

# 6) Verify
mariadb -e "SELECT VERSION(); SHOW WARNINGS;"
```

### Pattern 3 : Dump-and-restore upgrade for big version jumps

```bash
# 1) Logical dump from old server (10.6)
mariadb-dump --all-databases --single-transaction --routines --triggers --events \
  --hex-blob --max-allowed-packet=1G > /backup/full.sql

# 2) Provision a fresh server (e.g. 11.4 LTS) with empty datadir
# 3) Restore into the fresh server
mariadb --max-allowed-packet=1G < /backup/full.sql

# 4) Run mariadb-upgrade on the new server (idempotent ; harmless on fresh install)
mariadb-upgrade --user=root --password

# 5) Verify row counts per critical table match the pre-dump source
```

### Pattern 4 : Feature-detection at application boot

```sql
-- Detect JSON_TABLE (10.6+) availability without a try-catch
SELECT COUNT(*) AS supports_json_table
FROM information_schema.routines
WHERE routine_schema = 'mysql' AND routine_name = 'JSON_TABLE';

-- Detect window-function support (10.2+)
SELECT @@have_innodb;  -- general sanity ; for window/CTE detect via SELECT VERSION()

-- Detect sequences (10.3+, MariaDB-only)
SELECT COUNT(*) AS supports_sequences
FROM information_schema.tables
WHERE table_schema = 'information_schema' AND table_name = 'SEQUENCES';
```

## What This Skill Does NOT Cover

- Detailed migration from MySQL to MariaDB : see `mariadb-impl-migration-mysql-to-mariadb`
- Replication / Galera architecture : see `mariadb-core-replication-model`
- Storage-engine selection : see `mariadb-core-storage-engines`
- Per-feature SQL syntax : see the corresponding `mariadb-syntax-*` skill

## Reference Links

- `references/methods.md` : full LTS / EOL table, complete feature-introduction matrix,
  breaking-changes-per-major table, mariadb-upgrade tool reference
- `references/examples.md` : 10 worked examples (version detection, in-place upgrade,
  dump-and-restore, feature detection, rollback strategy)
- `references/anti-patterns.md` : 10 anti-patterns with WHY-it-fails and the correct
  alternative

## Sources

All facts in this skill are verified against :

- https://mariadb.org/about/#maintenance-policy (LTS / EOL table, verified 2026-05-19)
- https://mariadb.com/kb/en/release-notes/ (per-version release notes index)
- https://mariadb.com/kb/en/about-mariadb-server/ (cadence and versioning)
- https://mariadb.com/kb/en/upgrading/ (mariadb-upgrade procedure)
- https://mariadb.com/kb/en/mariadb-vs-mysql-features/ (feature divergence)

See `references/methods.md` for the per-row source citation behind every feature
entry in the matrix.
