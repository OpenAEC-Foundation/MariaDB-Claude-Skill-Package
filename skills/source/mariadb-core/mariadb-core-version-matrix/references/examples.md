# MariaDB Version Matrix : Worked Examples

Ten end-to-end examples covering version detection, in-place upgrade, dump-and-restore
upgrade, feature detection, rollback, and Galera-rolling-upgrade scenarios. All
commands tested against MariaDB 10.6, 10.11, 11.4, 11.8 documentation as published
on `mariadb.com/kb` (verified 2026-05-19).

---

## Example 1 : Identify the running version and capabilities

```sql
-- 10.6+ : minimum useful diagnostic block
SELECT
  VERSION()                       AS full_version,    -- 11.4.4-MariaDB-1:11.4.4+maria~ubu2404
  @@version_compile_os            AS os,              -- Linux
  @@version_compile_machine       AS arch,            -- x86_64
  @@innodb_version                AS innodb_version,  -- 11.4.4
  @@have_query_cache              AS has_query_cache, -- NO on 11.x, YES on 10.x
  @@have_innodb                   AS has_innodb;      -- YES

-- Detect MariaDB vs MySQL fork (paranoid)
SELECT
  CASE
    WHEN VERSION() LIKE '%MariaDB%' THEN 'MariaDB'
    ELSE 'MySQL or fork'
  END AS server_brand;
```

Expected output on a 11.4 LTS install :

```
full_version: 11.4.4-MariaDB
os: Linux
arch: x86_64
innodb_version: 11.4.4
has_query_cache: NO
has_innodb: YES
server_brand: MariaDB
```

Use this block as the first action of any tool that generates version-conditional
SQL. Never assume.

---

## Example 2 : In-place upgrade from 10.6 LTS to 10.11 LTS (Debian / Ubuntu)

Pre-condition : 10.6 server running ; `/var/lib/mysql` is the datadir ; `mariadb`
client installed and root authentication via socket works.

```bash
# 1) Mandatory backup BEFORE any change
mariadb-dump \
  --all-databases \
  --single-transaction \
  --routines --triggers --events \
  --master-data=2 \
  --hex-blob \
  --max-allowed-packet=1G \
  > /backup/pre-10.11-upgrade-$(date +%Y%m%d-%H%M).sql

# 2) Verify backup is restorable in principle (smoke test on a sandbox)
head -50 /backup/pre-10.11-upgrade-*.sql | grep -E '^-- (Server|MariaDB)'

# 3) Stop the 10.6 server cleanly
systemctl stop mariadb

# 4) Replace the apt sources for 10.11 (replace ubuntu codename as needed)
curl -fsSL https://r.mariadb.com/downloads/mariadb_repo_setup | sudo bash -s -- \
  --mariadb-server-version="mariadb-10.11"

apt update
apt install --only-upgrade mariadb-server mariadb-client mariadb-common

# 5) Start the 10.11 server
systemctl start mariadb

# 6) Run mariadb-upgrade ONCE
mariadb-upgrade --user=root

# 7) Verify
mariadb -e "SELECT VERSION();"
# Expected: 10.11.x-MariaDB-...

# 8) Inspect warnings
mariadb -e "SHOW WARNINGS;"

# 9) Smoke-test the application against the 10.11 server before returning to full traffic
```

Rollback : stop server, swap binaries back to 10.6 packages, copy the pre-upgrade
datadir snapshot back into `/var/lib/mysql` (a datadir-level snapshot via LVM or
filesystem clone is required ; mariadb-upgrade is NOT reversible on the live datadir
alone).

---

## Example 3 : In-place upgrade from 11.4 LTS to 11.8 LTS

Same procedure as Example 2 with these substitutions :

```bash
curl -fsSL https://r.mariadb.com/downloads/mariadb_repo_setup | sudo bash -s -- \
  --mariadb-server-version="mariadb-11.8"
```

Verification post-upgrade :

```sql
-- 11.8+ : verify the upgrade landed
SELECT VERSION();
-- Expected: 11.8.x-MariaDB-...

-- 11.8+ : verify the vector data type is now available (new in 11.7+ preview, GA 11.8)
SELECT COUNT(*) AS supports_vector
FROM information_schema.routines
WHERE routine_name LIKE 'VEC_%';
```

---

## Example 4 : Dump-and-restore upgrade from 10.6 to 11.8 (cross-major jump)

When you cannot do the sequence 10.6 -> 10.11 -> 11.4 -> 11.8 incrementally, the
dump-and-restore path is the supported alternative. Costs full downtime equal to
dump+restore wall-clock.

```bash
# === On the OLD 10.6 server ===

# 1) Dump everything, including routines, triggers, events
mariadb-dump \
  --all-databases \
  --single-transaction \
  --routines --triggers --events \
  --hex-blob \
  --max-allowed-packet=1G \
  > /backup/full-10.6.sql

# 2) Stop the OLD server
systemctl stop mariadb

# === On a FRESH 11.8 server (different host, or new datadir on same host) ===

# 3) Install 11.8 from scratch
apt install mariadb-server   # with 11.8 repo configured

# 4) Restore the dump
mariadb --max-allowed-packet=1G < /backup/full-10.6.sql

# 5) Run mariadb-upgrade (harmless on fresh install but safest)
mariadb-upgrade --user=root

# 6) Verify
mariadb -e "SELECT VERSION();"
# Expected: 11.8.x-MariaDB-...

# 7) Re-create users : the dump preserves them via mysql.global_priv, but the schema
#    transformation from 10.6 to 11.8 is non-trivial. Validate by listing users :
mariadb -e "SELECT User, Host FROM mysql.global_priv ORDER BY User;"
```

This path bypasses cumulative system-table cruft from a long upgrade history and is
the cleanest way to land on the newest LTS from an old install.

---

## Example 5 : Detect feature availability before generating SQL

```sql
-- 10.6+ : detect JSON_TABLE support
SELECT COUNT(*) > 0 AS supports_json_table
FROM information_schema.SQL_FUNCTIONS
WHERE FUNCTION = 'JSON_TABLE';
-- Returns 1 on 10.6+, 0 on 10.5 and earlier

-- 10.3+ : detect sequence support (MariaDB-only)
SELECT COUNT(*) > 0 AS supports_sequences
FROM information_schema.tables
WHERE table_schema = 'information_schema'
  AND table_name   = 'SEQUENCES';

-- 10.5+ : detect INSERT ... RETURNING support
SELECT
  CASE
    WHEN @@version >= '10.5'
      THEN 'supported'
    ELSE 'not supported'
  END AS insert_returning;

-- 10.8+ : detect descending indexes
-- (no information_schema flag ; check the running version)
SELECT
  CASE
    WHEN @@version >= '10.8'
      THEN 'descending indexes supported'
    ELSE 'descending indexes NOT supported'
  END AS desc_index;
```

---

## Example 6 : Application-side feature gating (Python with mariadb connector)

```python
# Python 3 with mariadb-connector-python
import mariadb

conn = mariadb.connect(user='app', password='...', host='db', database='shop')
cur = conn.cursor()
cur.execute("SELECT VERSION()")
version_str = cur.fetchone()[0]

# version_str is e.g. "11.4.4-MariaDB" ; strip suffix
major_minor = tuple(int(x) for x in version_str.split('-')[0].split('.')[:2])

if major_minor >= (10, 6):
    # JSON_TABLE works
    cur.execute("""
        SELECT jt.product_id, jt.name
        FROM products p,
          JSON_TABLE(p.attrs, '$.variants[*]' COLUMNS (
            product_id INT PATH '$.id',
            name VARCHAR(80) PATH '$.name'
          )) AS jt
    """)
else:
    # Fallback : application-side JSON parsing
    cur.execute("SELECT id, attrs FROM products")
    # parse attrs in Python
```

---

## Example 7 : Galera rolling upgrade (same major series)

Galera supports rolling upgrades within the same major series. Cross-major rolling
is NOT supported : a full cluster shutdown is required for 10.11 -> 11.4.

```bash
# 3-node Galera cluster, all on 10.11.7, upgrading to 10.11.9

# For each node, one at a time :

# 1) Desync node from cluster
mariadb -e "SET GLOBAL wsrep_desync = ON;"

# 2) Stop node
systemctl stop mariadb

# 3) Upgrade binaries (patch-level within 10.11)
apt install --only-upgrade mariadb-server

# 4) Start node, it rejoins via IST (Incremental State Transfer)
systemctl start mariadb

# 5) Run mariadb-upgrade on this node
mariadb-upgrade --user=root

# 6) Re-sync
mariadb -e "SET GLOBAL wsrep_desync = OFF;"

# 7) Verify it is part of the cluster again
mariadb -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
# Expected: 3

# Repeat for next node only after the previous is fully synced
```

Cross-major Galera upgrade (10.11 to 11.4) requires :

```bash
# 1) Take a logical dump from one node (--single-transaction is unsafe on Galera ;
#    use --master-data + STOP SLAVE or take from a desynced node)
# 2) Shut down ALL nodes
# 3) Bootstrap a new 11.4 cluster from a fresh datadir + the dump
# 4) Have all replicas SST from the bootstrap node
```

---

## Example 8 : Rollback strategy after failed upgrade

`mariadb-upgrade` is NOT reversible. The datadir is modified in place. Rollback is
only possible from a pre-upgrade snapshot.

```bash
# Pre-upgrade preparation (run BEFORE the upgrade)

# Option A : LVM snapshot of the datadir filesystem
lvcreate --snapshot --name mariadb-pre-upgrade --size 20G /dev/vg0/mariadb

# Option B : Filesystem-level copy with server stopped
systemctl stop mariadb
cp -a /var/lib/mysql /var/lib/mysql.backup-$(date +%F)
systemctl start mariadb

# === If upgrade fails ===

# 1) Stop the (broken) new server
systemctl stop mariadb

# 2) Restore the snapshot OR copy the backup back
lvconvert --merge /dev/vg0/mariadb-pre-upgrade
# OR
rm -rf /var/lib/mysql
cp -a /var/lib/mysql.backup-2026-05-19 /var/lib/mysql

# 3) Re-install the OLD binaries (re-pin apt to the old major)
apt install mariadb-server=10.6.* mariadb-client=10.6.* mariadb-common=10.6.*

# 4) Start
systemctl start mariadb
mariadb -e "SELECT VERSION();"
# Expected: 10.6.x-MariaDB-... (back to pre-upgrade state)
```

---

## Example 9 : Schedule an upgrade window using current LTS dates

Practical decision script for choosing an upgrade target as of 2026 :

```sql
-- Run on the production server in 2026
SELECT VERSION();
```

Decision logic :

| Current version | Action by 2026-07-06 (10.6 EOL) | Recommended target |
|-----------------|--------------------------------|--------------------|
| 10.4 or older | Already EOL ; upgrade urgently | 10.11 LTS (one major hop at a time) |
| 10.5 | Past EOL since 2025-06 ; upgrade | 10.6 LTS, then 10.11 LTS |
| 10.6 LTS | EOL 2026-07-06 ; upgrade within 12 months | 10.11 LTS |
| 10.11 LTS | Active until 2028-02-16 | Optional move to 11.4 LTS by 2027 |
| 11.4 LTS | Active until 2029-05-29 | Optional move to 11.8 LTS by 2028 |
| 11.8 LTS | Active until 2028-06-04 | Stay ; evaluate next LTS in 2026-2027 |
| Any interim (10.7-10.10, 11.0-11.3, 11.5-11.7, 12.0) | EOL or short-window | Move to nearest LTS within months |

---

## Example 10 : Pre-flight checklist for any upgrade

```bash
#!/bin/bash
# pre-upgrade-check.sh : run BEFORE any MariaDB binary upgrade

set -e

echo "=== 1. Current version ==="
mariadb -e "SELECT VERSION();"

echo "=== 2. Disk space (datadir + 2x for backup) ==="
df -h /var/lib/mysql /backup

echo "=== 3. Replication status (if applicable) ==="
mariadb -e "SHOW REPLICA STATUS\G" 2>/dev/null || echo "Not a replica"
mariadb -e "SHOW MASTER STATUS\G"  2>/dev/null || true

echo "=== 4. Open / long-running transactions ==="
mariadb -e "
  SELECT trx_id, trx_state, trx_started, trx_query
  FROM information_schema.innodb_trx
  ORDER BY trx_started;
"

echo "=== 5. Removed-in-target-version variables (for 10.x -> 11.x) ==="
mariadb -e "
  SHOW VARIABLES LIKE 'query_cache%';
  SHOW VARIABLES LIKE 'innodb_undo_tablespaces';
  SHOW VARIABLES LIKE 'innodb_buffer_pool_chunk_size';
" | tee /tmp/pre-upgrade-vars.txt

echo "=== 6. Storage engines in use ==="
mariadb -e "
  SELECT engine, COUNT(*) AS n
  FROM information_schema.tables
  WHERE table_schema NOT IN ('mysql','information_schema','performance_schema','sys')
  GROUP BY engine;
"

echo "=== 7. Backup ==="
mariadb-dump --all-databases --single-transaction --routines --triggers --events \
  > /backup/pre-upgrade-$(date +%F-%H%M).sql

echo "Pre-flight checks complete. Proceed with upgrade only if all sections are clean."
```

Run this script and review the output. If `query_cache_*` variables are non-zero on
a 10.x server about to upgrade to 11.x, edit `my.cnf` to remove them BEFORE starting
the new binary, otherwise the new server will refuse to start with "Unknown system
variable".
