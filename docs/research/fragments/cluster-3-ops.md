# Cluster-3 Research : MariaDB Ops (Replication, Galera, Backup, Tuning, Security, Migration, Frappe)

> Scope : MariaDB 10.6 LTS, 10.11 LTS, 11.x current, 12.x next.
> Source policy : MariaDB KB, mariadb.com/docs, MariaDB/server GitHub, mariadb.org, galeracluster.com, jira.mariadb.org only.
> Last verified : 2026-05-19. All claims tied to citations. No dev.mysql.com.

---

## 1. Replication

MariaDB supports three replication shapes : asynchronous (default), semi-synchronous, and parallel-applier on the replica side. Asynchronous replication does not wait for replicas, which delivers the lowest latency but allows data loss if the primary fails before the binary log reaches a replica. Semi-synchronous replication closes that window by forcing the primary to wait for at least one replica to acknowledge receipt of the binlog event before returning to the client. The relevant variables are `rpl_semi_sync_master_enabled`, `rpl_semi_sync_slave_enabled`, and `rpl_semi_sync_master_timeout` (default 10000 ms). Semi-sync was built into the MariaDB server starting in 10.3, removing the need for the separate plug-in install that earlier versions required ([KB semisync](https://mariadb.com/kb/en/semisynchronous-replication/)).

```ini
# my.cnf on primary (MariaDB 10.3+)
[mariadb]
rpl_semi_sync_master_enabled=ON
rpl_semi_sync_master_timeout=20000

# my.cnf on replica
[mariadb]
rpl_semi_sync_slave_enabled=ON
```

Parallel replication is configured with `slave_parallel_threads` (worker pool size for event application) and `slave_parallel_mode`. The available modes are `optimistic` (default since 10.5.1, executes transactional DML in parallel with automatic conflict detection and rollback), `conservative` (uses group-commit information to identify non-conflicting transactions, default until 10.5.0), `aggressive` (parallel without conflict-avoidance heuristics), `minimal` (only the commit phase runs in parallel), and `none` (single-threaded applier). Both primary and replica must be MariaDB 10.0.5 or later ([KB parallel](https://mariadb.com/kb/en/parallel-replication/)).

```ini
# Replica my.cnf (MariaDB 10.5.1+)
[mariadb]
slave_parallel_threads=4
slave_parallel_mode=optimistic
slave_parallel_max_queued=262144
slave_domain_parallel_threads=2
```

**Diverges from MySQL** : MariaDB GTID format is `domain-server-sequence`, a triple of three integers (for example `0-1-10`) where domain is a 32-bit unsigned int identifying a logical replication stream, server is the originating server id, and sequence is a 64-bit unsigned counter. MySQL uses `uuid:seqno`. The KB states explicitly that "MariaDB can be a replica for a MySQL primary, but MySQL cannot be a replica for a MariaDB primary" ([KB gtid](https://mariadb.com/kb/en/gtid/)). `gtid_strict_mode` rejects any operation that could cause binlog divergence (for example replicating a GTID with a lower sequence than one already present). Operational variables are `gtid_slave_pos`, `gtid_binlog_pos`, and the composite `gtid_current_pos`.

Multi-source replication has been a MariaDB feature since the 10.0 series and uses named connections via `CHANGE MASTER 'connection_name' TO ...`. Each connection name must be unique and is case-insensitive ([KB multi-source](https://mariadb.com/kb/en/multi-source-replication/)).

```sql
-- MariaDB 10.0+ multi-source replication
CHANGE MASTER 'analytics' TO
  MASTER_HOST='server1.example.com',
  MASTER_USER='repl',
  MASTER_PASSWORD='secret123',
  MASTER_USE_GTID=slave_pos;

START SLAVE 'analytics';
SHOW ALL SLAVES STATUS;
RESET SLAVE 'analytics' ALL;   -- remove channel permanently
```

Binary log formats are `STATEMENT`, `ROW`, and `MIXED`. MariaDB still ships `MIXED` as the default per the KB ([KB binlog formats](https://mariadb.com/kb/en/binary-log-formats/)), and switches to row-based encoding for any statement the server determines is unsafe (non-deterministic functions, `LIMIT` without `ORDER BY`, certain stored procedures). `binlog_row_image` selects `FULL` (default, before+after for every column), `MINIMAL` (only changed columns plus a minimal before image), or `NOBLOB` (full before image but excludes unchanged BLOB and TEXT columns) to reduce binlog volume.

## 2. Galera Cluster

Galera is a synchronous multi-master cluster built on certification-based replication. It requires the `wsrep_provider` library (`galera-4` from MariaDB 10.4 onwards) and a `wsrep_cluster_address=gcomm://node1,node2,node3` style address. A minimum of three nodes is required for stable quorum, since a two-node cluster cannot reliably distinguish a peer failure from a network split. Galera uses a primary component (PC) algorithm and supports weighted quorum via the `pc.weight` provider option to bias which partition survives a split. SST (state-snapshot-transfer) methods are `mariabackup` (recommended, non-blocking, InnoDB-aware), `rsync` (locks the donor), and `mysqldump` (deprecated for SST). IST (incremental state transfer) applies the missing write-sets from the donor's `gcache` when the joining node has only fallen briefly behind ([KB SST](https://mariadb.com/docs/galera-cluster/high-availability/state-snapshot-transfers-ssts-in-galera-cluster/introduction-to-state-snapshot-transfers-ssts.md)).

```ini
# Galera node my.cnf (MariaDB 10.4+ with galera-4)
[mariadb]
wsrep_on=ON
wsrep_provider=/usr/lib/galera/libgalera_smm.so
wsrep_cluster_address=gcomm://node1,node2,node3
wsrep_cluster_name=prod_cluster
wsrep_node_address=10.0.0.11
wsrep_node_name=node1
wsrep_sst_method=mariabackup
wsrep_sst_auth=sstuser:sstpass
binlog_format=ROW
default_storage_engine=InnoDB
innodb_autoinc_lock_mode=2
```

**MariaDB-only** semantic : certification-based replication means write-write conflicts manifest as a `deadlock` only at `COMMIT` time, not mid-transaction as InnoDB row-locks would on a standalone server. Applications must retry on `ER_LOCK_DEADLOCK` even for transactions that never touched a conflicting row on the local node. The `wsrep_local_cert_failures` status variable tracks the count.

## 3. Backup and Restore

MariaDB ships two backup paths. `mysqldump` (renamed `mariadb-dump` from 10.5+) produces logical SQL dumps. Without `--single-transaction` it locks tables, which blocks writes on production. `--single-transaction` works only with transactional engines (InnoDB). `mariabackup` is the physical hot-backup tool, InnoDB-aware, supports incremental chains and partial backups ([KB mariabackup](https://mariadb.com/kb/en/mariabackup/), [KB full-backup](https://mariadb.com/docs/server/server-usage/backup-and-restore/mariadb-backup/full-backup-and-restore-with-mariadb-backup.md)).

```bash
# Full backup, prepare, restore (MariaDB 10.3+)
mariadb-backup --backup \
   --target-dir=/var/mariadb/backup/ \
   --user=mariadb-backup --password=mypassword

mariadb-backup --prepare \
   --target-dir=/var/mariadb/backup/

# Stop mariadbd before --copy-back
mariadb-backup --copy-back \
   --target-dir=/var/mariadb/backup/

chown -R mysql:mysql /var/lib/mysql/
```

The KB is explicit : the backup directory "must be empty or it must not exist" and `--prepare` is mandatory before any restore. Point-in-time recovery uses `mariadb-binlog` (renamed from `mysqlbinlog` in 10.5+) with `--start-datetime`, `--stop-datetime`, `--start-position`, or `--stop-position` against the binary log files. Incremental chains use `--incremental-basedir` to point each delta at its parent, then `--prepare --incremental-dir` applied in chain order. Single-table restore from a physical backup requires `--export` during prepare followed by `ALTER TABLE ... DISCARD TABLESPACE` and `IMPORT TABLESPACE` on the live server.

## 4. Performance Tuning

The buffer pool is the dominant tunable. The KB recommends "starting at several gigabytes of memory" and tracking the ratio of `innodb_buffer_pool_reads` to `innodb_buffer_pool_read_requests` (target under 1% of the read-request delta over time) ([KB buffer pool](https://mariadb.com/kb/en/innodb-buffer-pool/)). Industry convention on a dedicated DB host is 60-80% of RAM, but the KB phrases it as "not too large, because this can cause swapping, which more than undoes the benefits". From 10.11.12 onwards `innodb_buffer_pool_chunk_size` is deprecated and ignored ; the pool now resizes in 1 MB increments up to `innodb_buffer_pool_size_max`.

`innodb_flush_log_at_trx_commit` has three values per the KB ([KB innodb vars](https://mariadb.com/kb/en/innodb-system-variables/)). `1` (default) flushes the redo log to disk on every transaction commit and is the only fully ACID setting. `2` writes to the OS file cache on every commit but flushes only once per second, surviving a process crash but losing up to one second on an OS crash. `0` writes and flushes only once per second, accepting up to one second of loss on any crash. Pair with `sync_binlog=1` for crash-safe binlog replay.

```ini
# Write-heavy OLTP, dedicated host (MariaDB 10.6+)
[mariadb]
innodb_buffer_pool_size=24G
innodb_log_file_size=2G
innodb_flush_log_at_trx_commit=1
sync_binlog=1
innodb_io_capacity=2000        # SSD baseline
innodb_io_capacity_max=4000    # SSD burst
innodb_thread_concurrency=0    # unlimited, recommended default
innodb_fill_factor=90          # leave 10% for index growth
query_cache_type=OFF
query_cache_size=0
```

`innodb_io_capacity` and `_max` cap background flushing throughput, scaled to SSD vs HDD IOPS. `innodb_thread_concurrency=0` (unlimited) is the recommended modern default. `innodb_fill_factor` ranges 10-100, defaults 100, and acts as a hint to leave space in B-tree pages for future inserts ([KB fill factor](https://mariadb.com/kb/en/innodb-system-variables/#innodb_fill_factor)).

**Diverges from MySQL** : the query cache was removed in MySQL 8.0 but kept in MariaDB. The KB confirms the cache "does not scale well in environments with high throughput on multi-core machines" because "each time changes are made to the data in a table, all affected results in the query cache are cleared" ([KB query cache](https://mariadb.com/kb/en/query-cache/)). For any write-heavy workload, set `query_cache_type=OFF` and `query_cache_size=0`. Per-connection buffers (`sort_buffer_size`, `read_rnd_buffer_size`, `join_buffer_size`) multiply by `max_connections`, so over-tuning these silently inflates memory ceiling.

## 5. Security

Authentication is plug-in based. Legacy `mysql_native_password` uses SHA-1 and is deprecated. The modern recommended plug-in is `ed25519`, available since 10.1.21, which uses Elliptic Curve DSA, the same algorithm as OpenSSH ([KB ed25519](https://mariadb.com/kb/en/authentication-plugin-ed25519/)). `unix_socket` auth on the root user maps OS uid to DB identity and ships as the default on Debian and Ubuntu packages. `gssapi` (Kerberos) and `pam` are also supported for enterprise SSO.

```sql
-- Install plug-in dynamically (MariaDB 10.1.21+)
INSTALL SONAME 'auth_ed25519';

-- Or via my.cnf
-- [mariadb]
-- plugin_load_add = auth_ed25519

-- Create user with ed25519
CREATE USER 'alice'@'%' IDENTIFIED VIA ed25519 USING PASSWORD('secret');

-- Force TLS at GRANT level
GRANT SELECT ON app.* TO 'alice'@'%' REQUIRE SSL;
GRANT SELECT ON app.* TO 'bob'@'%'
  REQUIRE SUBJECT '/CN=bob/O=Acme/C=NL'
  AND ISSUER '/C=FI/O=Acme CA/CN=Root'
  AND CIPHER 'ECDHE-RSA-AES256-GCM-SHA384';
```

Role-based access has been in MariaDB since the 10.0 series, exposed via `CREATE ROLE`, `GRANT role TO user`, `SET ROLE`, and `SET DEFAULT ROLE` so the role activates on connect. `WITH ADMIN OPTION` lets the grantee re-grant the role ([KB grant](https://mariadb.com/kb/en/grant/), [KB roles](https://mariadb.com/kb/en/roles_overview/)). Only roles granted directly to a user can be set ; nested-role activation is not supported.

Encryption at rest uses the file-key-management plug-in (file-on-disk keyring), the aws-key-management plug-in (AWS KMS), or HashiCorp Vault via third-party plug-in. The minimal config for file-key-management ([docs file-key](https://mariadb.com/docs/server/security/encryption/data-at-rest-encryption/key-management-and-encryption-plugins/file-key-management-encryption-plugin.md)) :

```ini
# my.cnf (MariaDB 10.6+, file-key-management)
[mariadb]
plugin_load_add = file_key_management
loose_file_key_management_filename = /etc/mysql/encryption/keyfile.enc
loose_file_key_management_filekey   = FILE:/etc/mysql/encryption/keyfile.key
loose_file_key_management_encryption_algorithm = AES_CTR

# Per-feature toggles (community-known, verify per version in KB)
innodb_encrypt_tables=ON
innodb_encrypt_log=ON
encrypt_binlog=ON
encrypt_tmp_files=ON
```

TLS for replication is configured on the replica side via `CHANGE MASTER ... MASTER_SSL=1` plus `MASTER_SSL_CA`, `MASTER_SSL_CERT`, `MASTER_SSL_KEY`, and the strongly recommended `MASTER_SSL_VERIFY_SERVER_CERT=1` ([KB change-master](https://mariadb.com/kb/en/change-master-to/)).

```sql
STOP SLAVE;
CHANGE MASTER TO
   MASTER_SSL=1,
   MASTER_SSL_CA='/etc/my.cnf.d/certificates/ca.pem',
   MASTER_SSL_CERT='/etc/my.cnf.d/certificates/server-cert.pem',
   MASTER_SSL_KEY='/etc/my.cnf.d/certificates/server-key.pem',
   MASTER_SSL_VERIFY_SERVER_CERT=1;
START SLAVE;
```

## 6. Migration MySQL to MariaDB

The upgrade path is : stop MySQL, install MariaDB binaries on the same datadir, start mariadbd, run `mariadb-upgrade` ([KB upgrade](https://mariadb.com/kb/en/upgrading-from-mysql-to-mariadb/)). MariaDB is a drop-in replacement for MySQL 5.5 and 5.6 ; from MySQL 5.7 and 8.0 onwards several incompatibilities require remediation :

| Area | MySQL 5.7+ / 8.0 | MariaDB 10.6+ | Diverges from MySQL |
|------|------------------|---------------|---------------------|
| JSON storage | Native binary type | LONGTEXT with optional `CHECK (JSON_VALID(col))` | **yes** |
| Default auth plug-in | `caching_sha2_password` (8.0) | `mysql_native_password` / `ed25519` / `unix_socket` | **yes** |
| GTID format | `uuid:seqno` | `domain-server-sequence` | **yes**, replication path one-way |
| Sequences | not supported (AUTO_INCREMENT only) | SQL-standard `CREATE SEQUENCE ... NEXT VALUE FOR ...` (10.3+) | **MariaDB-only** |
| Role syntax | similar but not identical | `SET DEFAULT ROLE`, role mandatory on connect | similar, NOT interchangeable |
| User table | `mysql.user` | `mysql.global_priv` from 10.4+ (view `mysql.user` retained for compatibility) | **yes** |
| sql_mode defaults | strict by default (8.0) | strict by default (10.2.4+) but list differs | minor |

Per the KB, "MariaDB 10.4+ uses `mysql.global_priv` for privilege management". The `mysql.user` view is still queryable but is a compatibility layer over `global_priv`. Run `mariadb-upgrade` exactly once, NOT twice in a row, since it rewrites system tables idempotently but logs noise on a second run.

## 7. Frappe / ERPNext Companion Patterns

Frappe v14 and v15 require MariaDB 10.6.6 or newer ; Frappe v16/develop requires MariaDB 11.8 ([Frappe install](https://docs.frappe.io/framework/v15/user/en/installation)). Frappe is multi-tenant by spawning one database per site (each `bench new-site` creates a new DB), so connection pooling has to account for many tenant DBs sharing one server. Table naming convention is `tab<DoctypeName>` (for example `tabUser`, `tabSales Invoice` with embedded space). Child tables include parent linkage columns : `parent`, `parentfield`, `parenttype`, and `idx`.

The hard requirement is `utf8mb4` (4 bytes per character) charset on every Frappe DB, since the framework stores arbitrary Unicode including emoji and non-BMP code points. Without `innodb_default_row_format=dynamic`, an indexed `VARCHAR(255)` in utf8mb4 needs 1020 bytes per row in an index, exceeding the 767-byte default index-prefix limit on the older `Antelope` row format. The companion settings :

```ini
# /etc/mysql/mariadb.conf.d/frappe.cnf (MariaDB 10.6+ for Frappe v14/v15)
[mysqld]
character-set-client-handshake = FALSE
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci
innodb_file_per_table = 1
innodb_default_row_format = dynamic
innodb_large_prefix = 1
innodb_file_format = Barracuda
max_allowed_packet = 256M

[mysql]
default-character-set = utf8mb4
```

ERPNext bench typically connects via `unix_socket` auth on the root user (default Debian/Ubuntu install) and uses `bench backup` / `bench restore` wrappers around `mariadb-dump`. Production sites that need RPO under a few minutes should layer `mariabackup` incremental snapshots on top of `bench backup`. `bench --site <site> backup --with-files` includes uploaded files.

---

## Anti-Patterns

1. **`STATEMENT` binlog with non-deterministic functions**. `UUID()`, `NOW(6)` resolved at high resolution, `LIMIT` without `ORDER BY`, `SLEEP`, and user-defined functions can produce divergent results on the replica. The KB explicitly warns "the set of rows included cannot be predicted" ([KB binlog formats](https://mariadb.com/kb/en/binary-log-formats/)). Use `MIXED` or `ROW`.

2. **Galera with a 2-node cluster**. Two nodes cannot form a stable quorum. Any network partition either takes both nodes read-only (loss of availability) or causes split-brain when each node assumes the other is dead. Always run 3 nodes minimum, or 2 + one arbitrator (`garbd`) ([KB SST](https://mariadb.com/docs/galera-cluster/high-availability/state-snapshot-transfers-ssts-in-galera-cluster/introduction-to-state-snapshot-transfers-ssts.md)).

3. **Logical backup on production with `mysqldump` and no `--single-transaction`**. Default `mysqldump` issues table-locks per database, blocking writes for the entire dump duration. On any InnoDB workload, use `--single-transaction`. On mixed-engine schemas, switch to `mariabackup` for hot physical backup ([KB mariabackup](https://mariadb.com/kb/en/mariabackup/)).

4. **`innodb_buffer_pool_size` at 50% of RAM on a dedicated DB host**. Too conservative ; the host has nothing else to do. Industry convention 60-80%, KB warns only against making it large enough to cause OS swap ([KB buffer pool](https://mariadb.com/kb/en/innodb-buffer-pool/)). Set 70% as a starting point on a dedicated 32 GB+ host.

5. **Running `mariadb-upgrade` twice in a row**. The tool is idempotent but logs spurious warnings on the second pass and can mask real errors from the first pass under the second run's noise. Run once after every binary upgrade, not as a routine maintenance task ([KB upgrade](https://mariadb.com/kb/en/upgrading-from-mysql-to-mariadb/)).

6. **Encryption at rest with `innodb_encrypt_tables=ON` but `encrypt_binlog=OFF`**. Row-image leaks via the binary log. A read-only attacker with binlog access can reconstruct most of the protected data even though the tablespaces are encrypted. Encrypt the binlog and the temp files at the same time as the tablespaces.

7. **`GRANT ALL ON *.* TO 'app_user'@'%'` on production**. `ALL` includes `SUPER`, `FILE`, `PROCESS`, `RELOAD`, `SHUTDOWN`. An SQL-injection in any code path then escalates to server-level. Grant the minimal privilege set per database and per object, and use roles to keep the GRANT list short ([KB grant](https://mariadb.com/kb/en/grant/)).

8. **Migrating MySQL native JSON column to MariaDB without `JSON_VALID` `CHECK`**. MariaDB stores JSON as `LONGTEXT`, which does NOT validate JSON syntax on insert. A row that was valid JSON on MySQL can be corrupted by a subsequent non-JSON `UPDATE` on MariaDB without any error. Add `CHECK (JSON_VALID(col))` on every JSON column during the migration to preserve the MySQL invariant ([KB mariadb-vs-mysql](https://mariadb.com/docs/release-notes/community-server/about/compatibility-and-differences/mariadb-vs-mysql-compatibility.md)).

9. **Frappe install with default `utf8` (3-byte) charset**. Default `utf8` in older MariaDB is a 3-byte subset of UTF-8 that cannot store emoji or any code point above U+FFFF. The DB silently truncates or rejects on insert. Always set `character-set-server=utf8mb4` and `innodb_default_row_format=dynamic` BEFORE the first `bench new-site` ([Frappe install](https://docs.frappe.io/framework/v15/user/en/installation)).

10. **Replica with `MASTER_SSL=1` but `MASTER_SSL_VERIFY_SERVER_CERT=0`**. Connection is encrypted but vulnerable to a MITM that presents any valid cert. The KB recommends `MASTER_SSL_VERIFY_SERVER_CERT=1` to actually validate the primary's cert chain against `MASTER_SSL_CA` ([KB change-master](https://mariadb.com/kb/en/change-master-to/)).

---

## Newly Discovered Sub-Topics

1. **`MASTER_SSL_VERIFY_SERVER_CERT` and the `DEFAULT` keyword in 12.3+**. From 12.3 onwards the `CHANGE MASTER` TLS options accept `DEFAULT` to inherit server-level TLS config. Worth a dedicated section in `mariadb-impl-replication-setup` since older skill content typically hard-codes per-channel certs.

2. **`mysql.global_priv` view layer for backward compatibility**. From 10.4+ the privilege store is `mysql.global_priv`. Operators writing third-party tools that grep `mysql.user` will see consistent but read-only output via a compatibility view. Pair this in `mariadb-impl-migration-mysql-to-mariadb` with a query showing how to inspect privileges natively.

3. **`innodb_buffer_pool_chunk_size` deprecation in 10.11.12+**. The KB notes the variable is deprecated and ignored starting 10.11.12 ; the pool resizes in 1 MB increments up to `innodb_buffer_pool_size_max`. Tuning advice in any new skill must drop this variable.

4. **`file_key_management_use_pbkdf2` and `file_key_management_digest` in 12.0.1+**. Encryption keyring derivation now supports PBKDF2 iterations and selectable digest function. Older 10.6 skills will omit these ; a 12.x section is warranted in `mariadb-core-security-model`.

5. **Galera `pc.weight` weighted quorum**. Asymmetric data-center deployments (for example 2 nodes in DC-A, 1 in DC-B) benefit from biasing the survival of the larger DC during a WAN split. This is provider-option territory (`wsrep_provider_options="pc.weight=2"`) and merits a sub-section in `mariadb-impl-galera-cluster`.

6. **`slave_domain_parallel_threads` cap**. Independent of `slave_parallel_threads`, this caps parallel applier threads per replication domain. Important for multi-source setups where one domain should not starve another. Worth an inline note in the parallel-replication skill.

7. **Frappe v16 requires MariaDB 11.8**. A real version-floor change discovered in this research. Frappe v14/v15 floor is 10.6.6 ; v16/develop floor is 11.8. Cross-package note for both the Frappe and MariaDB skill packages.

8. **`mariadb-dump` and `mariadb-binlog` renames in 10.5+**. The historical `mysqldump` and `mysqlbinlog` binaries are renamed in 10.5+ ; the old names remain as compatibility symlinks. Scripts in production that hard-code `mysqldump` should be updated, otherwise package upgrades that drop the symlink will break backup cron jobs.
