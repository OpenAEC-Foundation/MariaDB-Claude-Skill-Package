# Examples : worked migration validations

Eight worked validations. Each shows the input fragment, the dimensions triggered, and the validation report exactly as the skill must produce it. The report format is fixed : findings table sorted BLOCKER -> WARNING -> SUGGESTION, a verdict line, and the standing `mariadb-upgrade` recommendation.

---

## Example 1 : MySQL 8 dump with JSON and caching_sha2_password (full report)

Input fragment (`shop_dump.sql`) :

```sql
CREATE TABLE `orders` (
  `id` BIGINT NOT NULL AUTO_INCREMENT,
  `payload` JSON NOT NULL,
  `status` VARCHAR(20) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE USER 'app'@'%' IDENTIFIED WITH caching_sha2_password BY 'secret';
GRANT SELECT, INSERT, UPDATE ON shop.* TO 'app'@'%';
```

Dimensions triggered : 2 (auth, BLOCKER), 1 (JSON, WARNING), 6 (AUTO_INCREMENT, SUGGESTION).

Report :

```
## Migration Validation : shop_dump.sql

| Finding | Severity | Location | MySQL vs MariaDB divergence | Remediation | Skill ref |
|---------|----------|----------|-----------------------------|-------------|-----------|
| caching_sha2_password user | BLOCKER | `CREATE USER 'app'@'%'` | MariaDB ships ed25519 / mysql_native_password / unix_socket ; it has no caching_sha2_password plugin | Drop user after import, INSTALL SONAME 'auth_ed25519', re-create IDENTIFIED VIA ed25519, re-issue GRANTs | mariadb-core-security-model |
| JSON column | WARNING | `orders.payload` | MySQL validates JSON on write ; MariaDB stores it as LONGTEXT with no write validation | After import : ALTER TABLE orders ADD CONSTRAINT orders_payload_json CHECK (JSON_VALID(payload)) | mariadb-syntax-json |
| AUTO_INCREMENT column | SUGGESTION | `orders.id` | MariaDB also supports CREATE SEQUENCE ; MySQL has only AUTO_INCREMENT | Keep AUTO_INCREMENT as-is ; do not convert to a sequence during migration | mariadb-syntax-sql-ddl |

**Verdict** : FAIL (1 blocker, 1 warning, 1 suggestion) | Re-validate after blockers are remediated.

**Standing recommendation** : run `mariadb-upgrade` exactly once after the import completes. See mariadb-impl-migration-mysql-to-mariadb.
```

---

## Example 2 : Dump with an INVISIBLE INDEX

Input fragment :

```sql
CREATE TABLE `customers` (
  `id` BIGINT NOT NULL AUTO_INCREMENT,
  `email` VARCHAR(255) NOT NULL,
  `legacy_ref` VARCHAR(64) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_email` (`email`),
  KEY `idx_legacy` (`legacy_ref`) INVISIBLE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

Dimensions triggered : 4 (INVISIBLE keyword, BLOCKER), 6 (AUTO_INCREMENT, SUGGESTION).

Report :

```
## Migration Validation : customers DDL

| Finding | Severity | Location | MySQL vs MariaDB divergence | Remediation | Skill ref |
|---------|----------|----------|-----------------------------|-------------|-----------|
| INVISIBLE INDEX keyword | BLOCKER | `customers` KEY `idx_legacy` | MySQL 8 marks an optimizer-ignored index INVISIBLE ; MariaDB 10.6+ uses IGNORED ; INVISIBLE is a syntax error | Rewrite to `KEY idx_legacy (legacy_ref) IGNORED` | mariadb-syntax-indexing |
| AUTO_INCREMENT column | SUGGESTION | `customers.id` | MariaDB also supports CREATE SEQUENCE | Keep AUTO_INCREMENT as-is | mariadb-syntax-sql-ddl |

**Verdict** : FAIL (1 blocker, 1 suggestion) | Re-validate after the keyword is rewritten.

**Standing recommendation** : run `mariadb-upgrade` exactly once after the import completes. See mariadb-impl-migration-mysql-to-mariadb.
```

---

## Example 3 : Clean MySQL 5.7 dump (mostly-pass)

Input fragment :

```sql
CREATE TABLE `products` (
  `id` BIGINT NOT NULL AUTO_INCREMENT,
  `sku` VARCHAR(32) NOT NULL,
  `price` DECIMAL(10,2) NOT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `uq_sku` (`sku`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE USER 'reader'@'%' IDENTIFIED WITH mysql_native_password BY 'pw';
GRANT SELECT ON shop.* TO 'reader'@'%';
```

Dimensions triggered : 6 (AUTO_INCREMENT, SUGGESTION) only. No JSON, no caching_sha2_password (mysql_native_password is supported by MariaDB), no INVISIBLE, no GROUPS, no functional index, InnoDB engine, utf8mb4 charset.

Report :

```
## Migration Validation : products DDL

| Finding | Severity | Location | MySQL vs MariaDB divergence | Remediation | Skill ref |
|---------|----------|----------|-----------------------------|-------------|-----------|
| AUTO_INCREMENT column | SUGGESTION | `products.id` | MariaDB also supports CREATE SEQUENCE | Keep AUTO_INCREMENT as-is | mariadb-syntax-sql-ddl |

**Verdict** : PASS WITH WARNINGS (0 blockers, 0 warnings, 1 suggestion). The dump imports cleanly ; the suggestion carries no functional risk.

**Standing recommendation** : run `mariadb-upgrade` exactly once after the import completes. See mariadb-impl-migration-mysql-to-mariadb.
```

Note : `mysql_native_password` is supported by MariaDB, so it is NOT a dimension-2 finding. Only `caching_sha2_password` and `sha256_password` are BLOCKERs.

---

## Example 4 : Schema with a GROUPS-frame view

Input fragment :

```sql
CREATE VIEW sales_window AS
SELECT
  region,
  sale_date,
  SUM(amount) OVER (
    PARTITION BY region
    ORDER BY sale_date
    GROUPS BETWEEN 2 PRECEDING AND CURRENT ROW
  ) AS rolling_amount
FROM sales;
```

Dimensions triggered : 9 (GROUPS frame, BLOCKER).

Report :

```
## Migration Validation : sales_window view

| Finding | Severity | Location | MySQL vs MariaDB divergence | Remediation | Skill ref |
|---------|----------|----------|-----------------------------|-------------|-----------|
| GROUPS window frame | BLOCKER | view `sales_window`, SUM() OVER clause | MariaDB supports ROWS and RANGE frames only ; GROUPS is a MySQL 8 feature absent from MariaDB | Rewrite the frame as ROWS BETWEEN 2 PRECEDING AND CURRENT ROW, or RANGE if peer-value semantics matter | mariadb-syntax-window-and-cte |

**Verdict** : FAIL (1 blocker) | Re-validate after the frame is rewritten.

**Standing recommendation** : run `mariadb-upgrade` exactly once after the import completes. See mariadb-impl-migration-mysql-to-mariadb.
```

---

## Example 5 : Dump with a functional index

Input fragment :

```sql
CREATE TABLE `events` (
  `id` BIGINT NOT NULL AUTO_INCREMENT,
  `doc` JSON NOT NULL,
  `created_at` DATETIME NOT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_doc_type` (( CAST(JSON_VALUE(`doc`, '$.type') AS CHAR(40)) ))
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

Dimensions triggered : 7 (functional index, BLOCKER), 1 (JSON, WARNING), 6 (AUTO_INCREMENT, SUGGESTION).

Report :

```
## Migration Validation : events DDL

| Finding | Severity | Location | MySQL vs MariaDB divergence | Remediation | Skill ref |
|---------|----------|----------|-----------------------------|-------------|-----------|
| Functional (expression) index | BLOCKER | `events` KEY `idx_doc_type` | MySQL 8.0.13+ allows an inline expression index ; MariaDB does not support indexing an expression directly | Add a virtual column `type_v VARCHAR(40) AS (JSON_VALUE(doc,'$.type')) VIRTUAL`, then `KEY idx_doc_type (type_v)` | mariadb-syntax-indexing |
| JSON column | WARNING | `events.doc` | MySQL validates JSON on write ; MariaDB stores it as LONGTEXT with no write validation | After import : ALTER TABLE events ADD CONSTRAINT events_doc_json CHECK (JSON_VALID(doc)) | mariadb-syntax-json |
| AUTO_INCREMENT column | SUGGESTION | `events.id` | MariaDB also supports CREATE SEQUENCE | Keep AUTO_INCREMENT as-is | mariadb-syntax-sql-ddl |

**Verdict** : FAIL (1 blocker, 1 warning, 1 suggestion) | Re-validate after the functional index is rewritten.

**Standing recommendation** : run `mariadb-upgrade` exactly once after the import completes. See mariadb-impl-migration-mysql-to-mariadb.
```

---

## Example 6 : Dump with a non-InnoDB engine and utf8mb3 charset

Input fragment :

```sql
CREATE TABLE `audit_log` (
  `id` BIGINT NOT NULL AUTO_INCREMENT,
  `entry` TEXT,
  `note` VARCHAR(255) CHARACTER SET utf8,
  PRIMARY KEY (`id`)
) ENGINE=ROCKSDB DEFAULT CHARSET=utf8mb3;
```

Dimensions triggered : 8 (engine, WARNING), 10 (charset, SUGGESTION twice), 6 (AUTO_INCREMENT, SUGGESTION).

Report :

```
## Migration Validation : audit_log DDL

| Finding | Severity | Location | MySQL vs MariaDB divergence | Remediation | Skill ref |
|---------|----------|----------|-----------------------------|-------------|-----------|
| Non-InnoDB storage engine | WARNING | `audit_log` ENGINE=ROCKSDB | engine availability differs between servers ; the MyRocks plugin must be installed on the MariaDB target or the CREATE TABLE fails or falls back | Run SHOW ENGINES on the target ; INSTALL SONAME 'ha_rocksdb' or convert the table to ENGINE=InnoDB before import | mariadb-core-storage-engines |
| utf8mb3 table charset | SUGGESTION | `audit_log` DEFAULT CHARSET=utf8mb3 | utf8/utf8mb3 is a 3-byte UTF-8 subset, no emoji or code points above U+FFFF | Convert to utf8mb4 with a deliberate collation | mariadb-errors-encoding-and-collation |
| utf8 column charset | SUGGESTION | `audit_log.note` | utf8 is an alias for utf8mb3, a 3-byte subset | Convert the column to utf8mb4 | mariadb-errors-encoding-and-collation |
| AUTO_INCREMENT column | SUGGESTION | `audit_log.id` | MariaDB also supports CREATE SEQUENCE | Keep AUTO_INCREMENT as-is | mariadb-syntax-sql-ddl |

**Verdict** : PASS WITH WARNINGS (0 blockers, 1 warning, 3 suggestions). May import once the engine is confirmed ; warnings carry documented cost.

**Standing recommendation** : run `mariadb-upgrade` exactly once after the import completes. See mariadb-impl-migration-mysql-to-mariadb.
```

---

## Example 7 : Dump expecting GTID continuity

Input fragment (dump header plus a migration brief line) :

```sql
-- Migration brief : "keep GTID replication so we can fail back to MySQL"
SET @@GLOBAL.GTID_PURGED='3e11fa47-71ca-11e1-9e33-c80aa9429562:1-4521';
```

Dimensions triggered : 3 (GTID continuity, BLOCKER).

Report :

```
## Migration Validation : shop_dump.sql (header + brief)

| Finding | Severity | Location | MySQL vs MariaDB divergence | Remediation | Skill ref |
|---------|----------|----------|-----------------------------|-------------|-----------|
| GTID continuity expectation | BLOCKER | dump header GTID_PURGED + migration brief | MySQL GTID is uuid:seqno ; MariaDB GTID is domain-server-sequence ; the formats are not interconvertible and MySQL cannot replicate from MariaDB | Treat the migration as a one-way dump-and-load ; discard the GTID_PURGED statement ; for a cut-over replica use binlog-positional replication with MASTER_USE_GTID=NO | mariadb-core-replication-model |

**Verdict** : FAIL (1 blocker) | The GTID-continuity plan is not achievable ; re-plan as one-way migration.

**Standing recommendation** : run `mariadb-upgrade` exactly once after the import completes. See mariadb-impl-migration-mysql-to-mariadb.
```

---

## Example 8 : Producing a remediation checklist

After validating `shop_dump.sql` from Example 1, the skill can be asked to turn the report into an ordered remediation checklist. The checklist sequences the fixes around the import :

```
## Remediation Checklist : shop_dump.sql

Before import :
  [ ] None required ; all fixes are post-import.

Import :
  [ ] Restore shop_dump.sql into a clean MariaDB datadir : `mariadb -u root -p < shop_dump.sql`
      (expect a warning on the caching_sha2_password CREATE USER ; ignore, fixed below).

After import, in order :
  [ ] BLOCKER : drop user 'app'@'%' ; INSTALL SONAME 'auth_ed25519' ;
      CREATE OR REPLACE USER 'app'@'%' IDENTIFIED VIA ed25519 USING PASSWORD('new-secret') ;
      GRANT SELECT, INSERT, UPDATE ON shop.* TO 'app'@'%'.
  [ ] WARNING : ALTER TABLE orders
      ADD CONSTRAINT orders_payload_json CHECK (JSON_VALID(payload)).
  [ ] SUGGESTION : leave orders.id AUTO_INCREMENT unchanged.

Finalise :
  [ ] Run `mariadb-upgrade` exactly once.
  [ ] Smoke-test the application against MariaDB.

Skill references : mariadb-core-security-model, mariadb-syntax-json,
mariadb-syntax-sql-ddl, mariadb-impl-migration-mysql-to-mariadb.
```

The checklist groups by phase (before import, import, after import, finalise) and orders the after-import fixes BLOCKER -> WARNING -> SUGGESTION. The standing `mariadb-upgrade` step is always the final action before smoke-testing.
