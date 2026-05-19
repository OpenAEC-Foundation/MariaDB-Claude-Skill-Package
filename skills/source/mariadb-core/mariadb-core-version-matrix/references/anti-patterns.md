# MariaDB Version Matrix : Anti-Patterns

Ten anti-patterns observed in production and forum reports. Each entry shows the
broken pattern, why it fails, and the correct alternative.

---

## Anti-Pattern 1 : Running 10.4 (or older) in production past EOL

**Broken** :

```bash
# Production server in 2026
mariadb -e "SELECT VERSION();"
# 10.4.34-MariaDB
```

**Why it fails** : MariaDB 10.4 reached community EOL on 2024-06-18. After EOL :

- Zero security patches. Every CVE published after EOL is unpatched.
- No bug-fix backports. Even data-corruption bugs are not addressed.
- No official binaries for new distro versions. Pinning old packages prevents OS
  security updates from landing too.

**Correct** : upgrade to the nearest active LTS, one major step at a time. For 10.4 :

```bash
# 10.4 -> 10.5 (interim ; EOL ; only as transit) -> 10.6 LTS -> 10.11 LTS
# Plan each step with backup + mariadb-upgrade + post-upgrade test.
```

Treat any EOL version as an open security incident. Schedule the upgrade window
within weeks, not quarters.

---

## Anti-Pattern 2 : Jumping 10.6 directly to 11.4 or 11.8

**Broken** :

```bash
# Plan : skip 10.11 because "less downtime"
systemctl stop mariadb
# Install 11.4 packages directly over 10.6 datadir
apt install mariadb-server=11.4.*
systemctl start mariadb
mariadb-upgrade --user=root
# Server starts but mysql.* tables are inconsistent ; subtle privilege
# resolution bugs surface days later
```

**Why it fails** : the `mariadb-upgrade` tool is designed to bridge ONE major step
at a time. The KB documents upgrade guides only for adjacent steps : 10.6 to 10.11,
10.11 to 11.4, 11.4 to 11.8. A 10.6 to 11.4 jump stacks two sets of schema and
default changes (mysql.global_priv evolution, query-cache-variables removal,
collation defaults, INSTANT-DDL bitmap format) that the tool resolves serially, not
combined.

**Correct** : do the staged sequence. Each hop is independent, reversible (from
snapshot), and verifiable.

```
10.6 LTS -> 10.11 LTS -> 11.4 LTS -> 11.8 LTS
```

If staged downtime is impossible, use the dump-and-restore path instead (see
`examples.md` Example 4) which sidesteps the upgrade tool entirely.

---

## Anti-Pattern 3 : Picking an interim release (11.2, 11.3, 11.5, 11.6, 11.7, 12.0) for production

**Broken** :

```bash
# "Latest features, must be best"
apt install mariadb-server=11.7.*
# 11.7 reaches EOL ~12 months later
# Next year you are off-support and must do another major upgrade
```

**Why it fails** : interim releases get ~1 year of community support. The whole
LTS designation exists so you do NOT have to upgrade more than every ~2-3 years.
Running on an interim signs you up for an annual forced upgrade.

**Correct** : pick the nearest LTS to the version you actually want. In 2026 :

| What you want | Pick |
|---------------|------|
| Conservative, max support | 10.11 LTS (EOL 2028-02-16) |
| Modern default | 11.4 LTS (EOL 2029-05-29) |
| Newest LTS | 11.8 LTS (EOL 2028-06-04) |

Use interim releases only on developer laptops, CI, or staging environments where
"upgrade churn" is cheap.

---

## Anti-Pattern 4 : Assuming MariaDB 10.x behaves like MySQL 5.x

**Broken** :

```sql
-- Copy-pasted from a MySQL 5.7 tutorial
CREATE TABLE event (
  id   BIGINT PRIMARY KEY AUTO_INCREMENT,
  data JSON      -- "MySQL has native JSON storage, MariaDB must too"
);

-- Application stores arbitrary text into the JSON column
INSERT INTO event (data) VALUES ('this is not json at all');
-- No error. MariaDB stored it.
```

**Why it fails** : MariaDB's `JSON` type is an alias for `LONGTEXT` with no
validation. MySQL 5.7.8+ stores JSON in compact binary format with structural
validation on insert. Code written for the MySQL assumption silently accepts
invalid data on MariaDB and only fails later at the application's `json.loads()`.

Other false equivalences :

- MySQL GTID `uuid:seqno` vs MariaDB GTID `domain-server-sequence` (L-004) :
  replication formats are NOT interconvertible.
- MySQL `mysql.user` is a real table ; MariaDB 10.4+ replaced it with
  `mysql.global_priv` and made `mysql.user` a compatibility view.
- MySQL `default_authentication_plugin` is `caching_sha2_password` ; MariaDB
  recommends `ed25519` for new installs.
- MySQL has `materialized views` via stored procedure pattern ; MariaDB has none.
- MySQL has `GROUPS` window-frame ; MariaDB does not (L-002).

**Correct** : for every cross-engine port, add validation explicitly. For JSON :

```sql
-- 10.2+ : validate on insert
CREATE TABLE event (
  id   BIGINT PRIMARY KEY AUTO_INCREMENT,
  data JSON NOT NULL CHECK (JSON_VALID(data))
);
INSERT INTO event (data) VALUES ('not json'); -- now errors
```

Always check the MariaDB KB for the feature, never assume MySQL parity.

---

## Anti-Pattern 5 : Forgetting `mariadb-upgrade` after a binary swap

**Broken** :

```bash
# Sysadmin does apt-level upgrade and reboot
apt install --only-upgrade mariadb-server
systemctl restart mariadb
# Done. Moving on.

# Three weeks later : random "Table 'mysql.global_priv' is read only" or
# "Column 'Repl_slave_priv' does not exist in mysql.user" errors during
# routine GRANT / REVOKE.
```

**Why it fails** : the binary upgrade puts new server code on disk and starts it
against the OLD `mysql.*` system-table schema. The new server tolerates the old
schema in many code paths but breaks at the edges : privilege checks, stored-
routine definitions, view metadata, time-zone tables. `mariadb-upgrade` is the
tool that rewrites these tables to the new schema. Skipping it leaves a hybrid
state that surfaces as intermittent SQL errors.

**Correct** : the upgrade procedure has FOUR steps, not three.

```bash
systemctl stop mariadb
apt install --only-upgrade mariadb-server
systemctl start mariadb
mariadb-upgrade --user=root           # this step is MANDATORY
mariadb -e "SHOW WARNINGS;"           # and read the warnings
```

Automate this in your distro's post-install hook or your config-management tool.
For Debian / Ubuntu it is usually invoked by the package post-install script, but
verify after every apt-driven upgrade.

---

## Anti-Pattern 6 : Running a beta / RC of next-LTS in production

**Broken** :

```bash
# 2025-09 : "12.0 RC is out, has the features we want"
apt install mariadb-server=12.0.0-rc.*
# Deploy to production
```

**Why it fails** : pre-GA builds receive last-minute changes between RC and GA.
Binary format on disk may change. Hidden upgrade-bugs surface during the RC test
window are fixed in GA. Putting an RC in front of paying customers means YOU are
the production-stress-test.

**Correct** : pre-GA goes on CI only. Wait for the LTS tag (e.g. `12.4 LTS` when
that designation lands). Subscribe to `https://mariadb.org/news/` for GA
announcements.

If you absolutely need a pre-GA feature in production, take the staged approach :
run the RC on a single read-replica for at least one full minor release cycle
before promoting it.

---

## Anti-Pattern 7 : Mixed-version Galera cluster across majors

**Broken** :

```bash
# 3-node cluster, rolling-upgrade started but stalled
node-a: 11.4.4-MariaDB
node-b: 11.4.4-MariaDB
node-c: 11.8.1-MariaDB  # "we'll get the others later"
# Cluster runs for weeks in this mixed state
```

**Why it fails** : Galera guarantees protocol stability within ONE major series.
Across majors :

- Replication-format details may shift (binlog row events, wsrep meta-block).
- Default sql_mode flags can differ ; statements that produce error on one node
  produce a warning on another, breaking replication consistency.
- New schema features (e.g. 11.8 vector type) cannot be replicated to a node
  that does not understand the DDL.

The cluster appears to work until the first cross-version DDL hits, at which
point one node aborts and you have a split-brain or a forced SST.

**Correct** : Galera rolling upgrade is supported within a major series only :

```
All on 10.11.7 -> all on 10.11.9 : rolling, safe
All on 11.4.4  -> all on 11.4.6  : rolling, safe
All on 11.4    -> all on 11.8    : requires FULL cluster shutdown
                                   plus dump-and-restore bootstrap
```

For cross-major Galera upgrade : shut down all nodes, dump from one, restore into
a fresh-bootstrapped cluster on the new major.

---

## Anti-Pattern 8 : Trusting the AUR / PPA / homebrew version label without checking VERSION()

**Broken** :

```bash
# Mac developer
brew install mariadb
mariadb -e "SELECT VERSION();"
# 12.0.1-MariaDB
# Developer ships a feature using JSON_TABLE assuming "we run 12.x in prod"

# Production
mariadb -e "SELECT VERSION();"
# 10.6.18-MariaDB
# JSON_TABLE was added in 10.6, so it works ; coincidence
# But the same developer next week uses VECTOR (11.7+) and prod blows up
```

**Why it fails** : third-party package channels move at their own pace. Homebrew,
Arch AUR, Ubuntu universe, FreeBSD ports, Docker Hub `mariadb:latest` and
`mariadb:11` all label versions differently from the upstream LTS designation.
The only authoritative version-marker is what the server reports for `VERSION()`.

**Correct** : in any project that touches MariaDB, codify the supported version
range explicitly :

```python
# Application bootstrap
required = (10, 11)   # minimum supported MariaDB
cur.execute("SELECT VERSION()")
v = cur.fetchone()[0]
parts = tuple(int(x) for x in v.split('-')[0].split('.')[:2])
if parts < required:
    raise RuntimeError(f"MariaDB {required} required, got {v}")
```

And in your dev environment, pin to the same major.minor LTS as production. Use
`docker pull mariadb:11.4` (with a digest-pin in CI) over `mariadb:latest`.

---

## Anti-Pattern 9 : Running `mariadb-upgrade` twice "to be safe"

**Broken** :

```bash
# Sysadmin upgrades 10.11 -> 11.4
mariadb-upgrade --user=root
# Warnings scroll by ; sysadmin worried, runs it again
mariadb-upgrade --user=root
# Second run completes cleanly. False reassurance.
```

**Why it fails** : `mariadb-upgrade` is idempotent at the table-schema level but
NOT at the warning-log level. The first run emits real warnings (e.g. "table X
has a deprecated column type, please fix"). The second run finds the tables
already-upgraded, emits a different (mostly empty) warning set, and the operator
walks away thinking everything is fine. The original warnings, which deserved a
fix, were never addressed.

**Correct** : run it EXACTLY ONCE per binary upgrade. Capture and read the
warnings :

```bash
mariadb-upgrade --user=root 2>&1 | tee /var/log/mariadb-upgrade-$(date +%F).log
grep -iE 'warning|error' /var/log/mariadb-upgrade-*.log
# Fix every flagged item BEFORE returning the server to traffic
```

If you genuinely think a re-run is needed (e.g. you fixed a flagged table and
want to re-validate), use `mariadb-check --analyze` instead. That tool does
table validation without rewriting system tables.

---

## Anti-Pattern 10 : Configuring removed variables in `my.cnf` after major upgrade

**Broken** :

```bash
# 10.11 -> 11.0 upgrade
systemctl stop mariadb
apt install mariadb-server=11.0.*
systemctl start mariadb
# Server refuses to start :
# [ERROR] [MY-000067] [Server] unknown variable 'query_cache_type=ON'.
# [ERROR] Aborting

# Sysadmin spends an hour reading logs
```

**Why it fails** : the query cache was deprecated in 10.1.7 and REMOVED in 11.0.
`my.cnf` entries `query_cache_type`, `query_cache_size`, `query_cache_limit`
are unknown to the 11.x server, which refuses to start. Similar removals in
recent majors : `innodb_undo_tablespaces` (11.0+), `innodb_buffer_pool_chunk_size`
(10.11.12+ and 11.x).

**Correct** : as part of pre-upgrade preparation, audit `my.cnf` for variables
removed in the target version :

```bash
# Pre-upgrade : list all currently-set variables that involve removed names
mariadb -e "
  SHOW VARIABLES WHERE Variable_name IN (
    'query_cache_type', 'query_cache_size', 'query_cache_limit',
    'innodb_undo_tablespaces', 'innodb_buffer_pool_chunk_size'
  );
"

# Edit /etc/mysql/mariadb.conf.d/50-server.cnf and remove the offending lines
# THEN do the binary upgrade
```

Cross-reference the target version's release notes (e.g.
`mariadb.com/kb/en/changes-improvements-in-mariadb-11-0/`) for the full removal
list before every major upgrade. The pre-flight script in `examples.md` Example
10 includes the most common removed variables.
