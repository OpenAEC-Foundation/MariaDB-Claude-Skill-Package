# Anti-patterns : migration validation

Six anti-patterns that ship a broken or unsafe MySQL-to-MariaDB import. Each one names the mistake, shows why it fails, and gives the correct procedure.

---

## Anti-pattern 1 : Importing without validation

**The mistake** : running `mariadb -u root -p < mysql_dump.sql` directly, treating MariaDB as a perfect drop-in for any MySQL version.

**Why it fails** : MariaDB is a near drop-in for MySQL 5.5 and 5.6 only. From MySQL 5.7 and 8.0 onwards there are hard divergences : `caching_sha2_password` users that cannot authenticate, `INVISIBLE INDEX` syntax that aborts the import, `GROUPS` window frames in views that fail to parse, and JSON columns that silently lose write-time validation. An unvalidated import either aborts halfway (leaving a partial schema) or appears to succeed while carrying latent failures that surface in production.

**The correct procedure** : ALWAYS run the 10-dimension scan against the dump before importing. Produce the findings table, remediate every BLOCKER, decide on each WARNING, then import. The scan is cheap ; a half-imported production database is not.

---

## Anti-pattern 2 : Ignoring JSON columns because "the dump imported fine"

**The mistake** : seeing `JSON` columns import without error and concluding nothing needs to be done.

**Why it fails** : MariaDB `JSON` is an alias for `LONGTEXT`. The import succeeds because LONGTEXT accepts any text. But MySQL 5.7.8+ validated JSON on every write ; MariaDB does not. After the migration, the first non-JSON `UPDATE` on that column writes garbage with no error. The application that relied on the MySQL invariant "this column is always valid JSON" now reads corrupt data. The failure is silent and arrives weeks later.

**The correct procedure** : dimension 1 grades every JSON column as a WARNING precisely because the import is silent. For every JSON column, after import add `ALTER TABLE t ADD CONSTRAINT t_col_json CHECK (JSON_VALID(col))`. This restores the write-time validation the MySQL schema depended on. See `mariadb-syntax-json`.

---

## Anti-pattern 3 : Keeping caching_sha2_password users and "fixing it later"

**The mistake** : letting the `CREATE USER ... IDENTIFIED WITH caching_sha2_password` statements import, planning to deal with authentication after the apps are pointed at MariaDB.

**Why it fails** : MariaDB has no `caching_sha2_password` plugin. The user rows import (they are just rows in `mysql.global_priv`), but the users cannot authenticate at all. The application gets `Plugin 'caching_sha2_password' is not loaded` on the first connection. There is no "later" : the apps are down from the moment they are repointed. Treating this as a WARNING instead of a BLOCKER guarantees an outage.

**The correct procedure** : dimension 2 grades this as a BLOCKER. Before the apps are repointed, drop every `caching_sha2_password` and `sha256_password` user, `INSTALL SONAME 'auth_ed25519'`, re-create the users `IDENTIFIED VIA ed25519`, and re-issue their GRANTs. Use `mysql_native_password` only where a driver cannot do ed25519. See `mariadb-core-security-model`.

---

## Anti-pattern 4 : Assuming GTID continuity across the migration

**The mistake** : planning a migration where MariaDB keeps replicating from MySQL using GTID auto-positioning, so the team can "fail back to MySQL" at any time.

**Why it fails** : MySQL GTID is `uuid:seqno` ; MariaDB GTID is `domain-server-sequence`. The two formats are not interconvertible. MariaDB can replicate from a MySQL primary only via binlog position, never via GTID auto-positioning. And MySQL cannot replicate from a MariaDB primary at all, so the "fail back" path does not exist once writes have landed on MariaDB. A plan built on GTID continuity is built on a feature that is not there.

**The correct procedure** : dimension 3 grades a GTID-continuity expectation as a BLOCKER. Re-plan the migration as a one-way dump-and-load. If a live cut-over replica is genuinely needed, configure binlog-positional replication (`MASTER_USE_GTID=NO`) for the cut-over window only, and accept that rollback means repointing apps back to the frozen MySQL with data loss for the window. See `mariadb-core-replication-model`.

---

## Anti-pattern 5 : Skipping mariadb-upgrade after the import

**The mistake** : finishing the import, smoke-testing the app, and declaring the migration done without running `mariadb-upgrade`.

**Why it fails** : the imported system tables came from MySQL. `mariadb-upgrade` is what migrates `mysql.user` to `mysql.global_priv` (10.4+), runs CHECK TABLE on every table to catch format drift, and rewrites incompatible system-table structures. Skipping it leaves the privilege system and system tables in a half-migrated state : role grants do not behave correctly, and format-drift errors surface unpredictably under load.

**The correct procedure** : the validation report ALWAYS closes with the standing recommendation to run `mariadb-upgrade` exactly ONCE after the import. Run it once, never twice in a row : the second pass logs spurious warnings that mask real first-pass errors. See `mariadb-impl-migration-mysql-to-mariadb`.

---

## Anti-pattern 6 : Treating INVISIBLE INDEX as a harmless cosmetic keyword

**The mistake** : seeing `KEY idx_x (col) INVISIBLE` in the dump and assuming MariaDB will either accept it or quietly ignore the keyword.

**Why it fails** : `INVISIBLE` is not a no-op on MariaDB ; it is a syntax error. The `CREATE TABLE` statement carrying it aborts. If the dump is one large transaction the whole import rolls back ; if it is statement-by-statement the schema ends up partial, missing that table and everything after it. The keyword is not cosmetic : it controls whether the optimizer uses the index, and MariaDB spells the same concept `IGNORED`.

**The correct procedure** : dimension 4 grades the `INVISIBLE` keyword as a BLOCKER. Rewrite every `INVISIBLE` on an index to `IGNORED` and every `VISIBLE` to `NOT IGNORED` before importing : `KEY idx_x (col) IGNORED`, or `ALTER TABLE t ALTER INDEX idx_x IGNORED`. The `IGNORED` feature exists from MariaDB 10.6. See `mariadb-syntax-indexing`.
