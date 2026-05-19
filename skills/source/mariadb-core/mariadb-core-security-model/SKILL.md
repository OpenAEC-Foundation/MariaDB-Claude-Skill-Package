---
name: mariadb-core-security-model
description: >
  Use when creating users, granting privileges, choosing an authentication plug-in, enabling roles, setting up encryption at rest or TLS, or migrating user schema from MySQL.
  Prevents the common mistake of using mysql_native_password instead of ed25519, forgetting to enable encryption on binlog / tmp / redo as well as tables, or losing access after upgrading mysql.user to mysql.global_priv.
  Covers authentication plug-ins (mysql_native_password, ed25519, gssapi, pam, unix_socket), CREATE USER and GRANT model, roles since 10.0.5, encryption at rest via file-key-management plug-in, TLS for client and replication, mysql.user vs mysql.global_priv shift in 10.4+.
  Keywords: mariadb security, CREATE USER, GRANT, REVOKE, roles, SET DEFAULT ROLE, ed25519, mysql_native_password, gssapi, pam, unix_socket, encryption at rest, file-key-management, TLS, SSL, REQUIRE SSL, mysql.user, mysql.global_priv, access denied, authentication failure, how do I add a user, secure mariadb, file_key_management_use_pbkdf2, file_key_management_digest, aws-key-management, MASTER_SSL, REQUIRE X509, REQUIRE SUBJECT, REQUIRE ISSUER, MAX_QUERIES_PER_HOUR, MAX_USER_CONNECTIONS, account_locked, PASSWORD EXPIRE, root has no password, cant grant role, hardening checklist
license: MIT
compatibility: "Designed for Claude Code. Requires MariaDB 10.6-LTS, 10.11-LTS, 11.x, 12.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

# MariaDB Core Security Model

How MariaDB authenticates users, stores privileges, activates roles, encrypts data at rest, and protects connections with TLS. Read this skill before creating any user, before enabling encryption, and before assuming MySQL security commands map one-to-one onto MariaDB.

## Quick Reference

- ed25519 is the modern default authentication plug-in. mysql_native_password (SHA-1) is legacy ; use it only for clients that cannot speak ed25519.
- Privileges live in `mysql.global_priv` since 10.4+. `mysql.user` is now a backward-compatibility view over `global_priv`, NOT the authoritative table.
- Encryption at rest needs ALL of : `innodb_encrypt_tables`, `innodb_encrypt_log`, `encrypt_binlog`, `encrypt_tmp_files`, `aria_encrypt_tables`. Encrypting only tables leaks data through the binlog and temp files.
- Roles exist since 10.0.5. ALWAYS pair `GRANT <role> TO <user>` with `SET DEFAULT ROLE <role> FOR <user>` so the user gets the role on connect.
- TLS requires BOTH `REQUIRE SSL` (or stricter) on the `GRANT` AND `ssl_cert` / `ssl_key` / `ssl_ca` server variables. One without the other fails open or fails closed.
- ALWAYS run `mariadb-upgrade` exactly once after a major version upgrade to migrate `mysql.user` rows into `mysql.global_priv`. NEVER run it twice in a row.
- NEVER `GRANT ALL ON *.* TO 'app_user'@'%'`. `ALL` includes `SUPER`, `FILE`, `PROCESS`, `RELOAD`, `SHUTDOWN`. An SQL-injection in any code path escalates to server-level.
- File-key-management gained `file_key_management_use_pbkdf2` and `file_key_management_digest` in 12.0.1+ for key-derivation hardening.
- ed25519 password hashing requires the `ed25519_password()` UDF or `USING PASSWORD('secret')` shortcut. The legacy `PASSWORD()` function does NOT work with ed25519.

## Authentication Plug-ins

MariaDB authentication is plug-in based. The server consults the `plugin` column of `mysql.global_priv` (the JSON `Priv.plugin` field, since 10.4+) to pick a verification routine per user.

| Plug-in | Algorithm / Mechanism | Use when | Avoid when |
|---------|-----------------------|----------|------------|
| `mysql_native_password` | SHA-1 challenge-response | Legacy clients that cannot upgrade | New accounts, security-sensitive deployments |
| `ed25519` | Elliptic Curve DSA (same as OpenSSH) | New accounts on 10.1.21+ | Clients on a very old MariaDB client library |
| `unix_socket` | Maps OS uid to DB identity | Local `root` access on Debian / Ubuntu, automation scripts running as a specific OS user | Network access ; the plug-in fails closed on TCP |
| `gssapi` | Kerberos / SSPI | Active Directory / enterprise SSO | Standalone servers without an AD domain |
| `pam` | Pluggable Authentication Modules | Centralised Linux PAM (LDAP, OTP, etc.) | Environments without PAM tooling |

**Decision rule** :
- ALWAYS use `ed25519` for new application users on 10.1.21+.
- ALWAYS use `unix_socket` for local `root` access on Debian / Ubuntu installs (this is the package default).
- NEVER create new accounts with `mysql_native_password` on a greenfield deployment.
- NEVER mix `unix_socket` with `'root'@'%'` ; `unix_socket` rejects TCP connections.

### Install and use ed25519

```sql
-- 10.1.21+ : dynamic install, no restart needed
INSTALL SONAME 'auth_ed25519';

-- Or persistent via my.cnf
-- [mariadb]
-- plugin_load_add = auth_ed25519

-- Create a user with ed25519 (server hashes 'secret' for you)
CREATE USER 'alice'@'%' IDENTIFIED VIA ed25519 USING PASSWORD('secret');

-- Or supply a pre-computed hash (computed via ed25519_password() UDF)
CREATE USER 'bob'@'%' IDENTIFIED VIA ed25519
  USING 'ZIgUREUg5PVgQ6LskhXmO+eZLS0nC8be6HPjYWR4YJY';
```

## User Schema : mysql.user vs mysql.global_priv

| Version | Authoritative table | Backward-compat view | Notes |
|---------|---------------------|----------------------|-------|
| 10.3 and earlier | `mysql.user` (column-per-privilege) | none | Legacy schema, one BOOLEAN column per global priv |
| 10.4+ | `mysql.global_priv` (JSON in `Priv`) | `mysql.user` (read mostly) | All global privs stored in JSON ; `mysql.user` updates partly compatible |
| 10.5.2+ | `mysql.global_priv` with `version_id` field | `mysql.user` view | `Priv.version_id` tracks the server that last wrote the row |

The `mysql.global_priv` table has three columns : `Host CHAR(60)`, `User CHAR(80)`, `Priv LONGTEXT` (a JSON document). The `Priv` JSON holds at minimum `access` (numeric bitmask), `plugin` (auth plug-in name), `authentication_string` (password hash), and optionally `password_last_changed`, `account_locked`, `version_id`.

**Decision rule** :
- ALWAYS query `mysql.global_priv` directly when inspecting accounts on 10.4+. The `mysql.user` view drops fields and can hide account state (`account_locked`, `password_last_changed`).
- ALWAYS run `mariadb-upgrade` once after upgrading from 10.3 to 10.4+ to migrate rows ; the server boots without this but new privileges may not work as expected.
- NEVER hand-edit `mysql.global_priv.Priv` JSON to set `access` ; use `GRANT` / `REVOKE` so the in-memory grant cache is refreshed.
- NEVER assume third-party tools that grep `mysql.user` see the full picture on 10.4+.

## GRANT Model

Full grant syntax (per MariaDB KB `grant`) :

```sql
GRANT priv_type [(column_list)] [, priv_type ...]
  ON [object_type] priv_level
  TO account_or_role [, account_or_role ...]
  [REQUIRE {NONE | tls_option ...}]
  [WITH grant_option_list]
```

| Element | Possible values | Notes |
|---------|-----------------|-------|
| `priv_type` | `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `CREATE`, `DROP`, `ALTER`, `INDEX`, `REFERENCES`, `EXECUTE`, `EVENT`, `TRIGGER`, `SUPER`, `FILE`, `PROCESS`, `RELOAD`, `SHUTDOWN`, `REPLICATION CLIENT`, `REPLICATION SLAVE`, `GRANT OPTION`, etc. | `ALL [PRIVILEGES]` is a shortcut for all of them ; avoid in production |
| `priv_level` | `*.*` (global), `db_name.*` (db), `db_name.tbl_name` (table), `tbl_name`, function or procedure scope | Smaller scope is always preferable |
| `REQUIRE` | `NONE`, `SSL`, `X509`, `SUBJECT 'subject'`, `ISSUER 'issuer'`, `CIPHER 'cipher'`, combinable with `AND` | `SUBJECT` and `ISSUER` imply `X509` ; `CIPHER` implies `SSL` |
| `WITH grant_option_list` | `GRANT OPTION`, `MAX_QUERIES_PER_HOUR n`, `MAX_UPDATES_PER_HOUR n`, `MAX_CONNECTIONS_PER_HOUR n`, `MAX_USER_CONNECTIONS n`, `MAX_STATEMENT_TIME seconds` | Resource limits enforced server-side, reset hourly |

**Decision rule** :
- ALWAYS scope to the smallest necessary `priv_level` : `db_name.*` for an application user, `db_name.tbl_name` for a service-specific user.
- ALWAYS prefer a role with the right privileges over a long per-user `GRANT` list.
- NEVER include `WITH GRANT OPTION` on application users ; reserve it for admin accounts.
- NEVER use `'%'` host on a privileged account ; pin to a known host or subnet.

## Roles (10.0.5+)

Roles are named privilege bundles. They activate explicitly with `SET ROLE` or implicitly on connect via `SET DEFAULT ROLE`.

```sql
-- 10.0.5+ : create a role, grant privileges to it, assign to a user
CREATE ROLE app_read;
GRANT SELECT ON app.* TO app_read;
GRANT app_read TO 'alice'@'%';

-- Make the role active automatically on connect
SET DEFAULT ROLE app_read FOR 'alice'@'%';

-- Switch role inside a session
SET ROLE app_read;
SET ROLE NONE;    -- deactivate all roles
```

**Decision rule** :
- ALWAYS set a default role for any user that you grant roles to. Without `SET DEFAULT ROLE`, the user connects with zero role-derived privileges and has to call `SET ROLE` manually.
- ALWAYS grant privileges TO the role, then grant the role TO the user. Mixing direct user grants with role grants makes audit harder.
- NEVER assume nested roles propagate on activation : only roles granted directly to a user can be set with `SET ROLE`. Privileges from nested roles still cascade, but the role name itself does not.
- NEVER grant `WITH ADMIN OPTION` to ordinary users : it lets them re-grant the role to anyone.

## Encryption at Rest

Three plug-ins are available : `file_key_management` (local key file, default and most common), `aws_key_management` (AWS KMS), and HashiCorp Vault via third-party plug-in.

### Minimum file-key-management config

```ini
# my.cnf : MariaDB 10.6+, all encryption surfaces ON
[mariadb]
plugin_load_add = file_key_management
loose_file_key_management_filename = /etc/mysql/encryption/keyfile.enc
loose_file_key_management_filekey   = FILE:/etc/mysql/encryption/keyfile.key
loose_file_key_management_encryption_algorithm = AES_CTR

# 12.0.1+ : harden key derivation
loose_file_key_management_use_pbkdf2 = 1000
loose_file_key_management_digest     = sha256

# Encrypt every surface : tables, redo log, binlog, temp files, aria
innodb_encrypt_tables = ON
innodb_encrypt_log    = ON
innodb_encrypt_temporary_tables = ON
encrypt_binlog        = ON
encrypt_tmp_files     = ON
encrypt_tmp_disk_tables = ON
aria_encrypt_tables   = ON
```

| Surface | Variable | Effect if OFF |
|---------|----------|---------------|
| InnoDB tablespaces | `innodb_encrypt_tables` (`ON`, `OFF`, `FORCE`) | Unencrypted `.ibd` files on disk |
| InnoDB redo log | `innodb_encrypt_log` | Plaintext redo log entries reveal recent row changes |
| InnoDB internal temp tables | `innodb_encrypt_temporary_tables` | Plaintext intermediate join / sort results |
| Binary log | `encrypt_binlog` | Plaintext row images in binlog ; key data leak for replicas |
| MyISAM / temp files | `encrypt_tmp_files` | Plaintext sort / aggregate spill files |
| Disk-based temp tables | `encrypt_tmp_disk_tables` | Plaintext temp tables once they spill to disk |
| Aria engine | `aria_encrypt_tables` | Plaintext Aria tablespaces (used by system tables) |

**Key file format** :

```
# /etc/mysql/encryption/keyfile.enc (community / enterprise <11.8)
<key_id>;<hex_encoded_key>

# Example
1;a7addd9adea9978fda19f21e6be987880e68ac92632ca052e5bb42b1a506939a

# Enterprise 11.8+ adds a version column
<key_id>;<version>;<hex_encoded_key>
```

**Decision rule** :
- ALWAYS encrypt every surface listed above when enabling at-rest encryption. Partial encryption leaks via the unprotected surfaces.
- ALWAYS chmod the key file to `0400 mysql:mysql`. World-readable key files defeat the purpose.
- ALWAYS enable `use_pbkdf2` and `digest=sha256` on 12.0.1+ for new deployments. PBKDF2 raises the cost of an offline key-file attack.
- NEVER store the key file on the same disk as the encrypted data without a separate access boundary (preferably a different volume mount with its own ACL).
- NEVER set `innodb_encrypt_tables=FORCE` without first re-encrypting existing tables ; new writes will succeed but the existing tables remain plaintext.

## TLS for Client Connections

Server-side TLS requires three variables in `my.cnf`, plus per-user `REQUIRE` on every account that should be TLS-only.

```ini
# my.cnf : enable server-side TLS (MariaDB 10.6+)
[mariadb]
ssl_ca   = /etc/mysql/certs/ca.pem
ssl_cert = /etc/mysql/certs/server-cert.pem
ssl_key  = /etc/mysql/certs/server-key.pem
tls_version = TLSv1.2,TLSv1.3
```

```sql
-- Per-user TLS enforcement (combine with the server config above)
GRANT SELECT ON app.* TO 'alice'@'%' REQUIRE SSL;

-- Strict X509 with cert-pinning
GRANT SELECT ON app.* TO 'bob'@'%'
  REQUIRE SUBJECT '/CN=bob/O=Acme/C=NL'
  AND ISSUER '/C=FI/O=Acme CA/CN=Root'
  AND CIPHER 'ECDHE-RSA-AES256-GCM-SHA384';
```

**Decision rule** :
- ALWAYS set `ssl_ca` / `ssl_cert` / `ssl_key` on the server AND `REQUIRE SSL` (or stricter) on the user. The server config alone makes TLS available ; the GRANT clause makes it mandatory.
- ALWAYS pin `tls_version` to `TLSv1.2,TLSv1.3`. Older TLS (1.0, 1.1) is broken.
- NEVER ship `ssl_cert` without the full certificate chain ; TLS handshake fails silently with intermediate-cert clients.
- NEVER use `REQUIRE SSL` without checking that the client driver actually validates the server cert. Many drivers default to no verification.

## TLS for Replication

```sql
-- On the replica
STOP SLAVE;
CHANGE MASTER TO
   MASTER_SSL = 1,
   MASTER_SSL_CA   = '/etc/mysql/certs/ca.pem',
   MASTER_SSL_CERT = '/etc/mysql/certs/replica-cert.pem',
   MASTER_SSL_KEY  = '/etc/mysql/certs/replica-key.pem',
   MASTER_SSL_VERIFY_SERVER_CERT = 1;
START SLAVE;

-- 12.3+ shortcut : inherit server-level TLS config
CHANGE MASTER TO
   MASTER_SSL = 1,
   MASTER_SSL_CA   = DEFAULT,
   MASTER_SSL_CERT = DEFAULT,
   MASTER_SSL_KEY  = DEFAULT,
   MASTER_SSL_VERIFY_SERVER_CERT = 1;
```

**Decision rule** :
- ALWAYS set `MASTER_SSL_VERIFY_SERVER_CERT = 1` ; without it, the connection is encrypted but MITM-vulnerable.
- ALWAYS use the `DEFAULT` keyword on 12.3+ to reuse server-level TLS config and avoid per-channel cert duplication.
- NEVER omit `MASTER_SSL_CA` ; without a CA the replica trusts any presented cert.

## Hardening Checklist

Run through this checklist after every fresh install :

1. `DROP USER ''@'localhost'` and any other anonymous accounts.
2. `DROP USER 'root'@'%'` ; keep `'root'@'localhost'` only.
3. Set `'root'@'localhost' IDENTIFIED VIA unix_socket` on Debian / Ubuntu.
4. Create one application user per database with `ed25519` and `REQUIRE SSL`.
5. Create roles per access tier (read-only, app-write, admin) and grant roles to users.
6. Enable file-key-management with PBKDF2 (12.0.1+) and chmod the key file `0400 mysql:mysql`.
7. Enable encryption on tables, log, binlog, tmp files, aria.
8. Configure `ssl_ca` / `ssl_cert` / `ssl_key` server-side and `tls_version=TLSv1.2,TLSv1.3`.
9. Grant minimum privileges per role ; NEVER `GRANT ALL ON *.*`.
10. Run `mariadb-upgrade` once after the install or upgrade to migrate any stale `mysql.user` rows.

## Reference Links

- `references/methods.md` : per-plug-in install + syntax, full GRANT privilege table, role lifecycle, encryption variable matrix per version.
- `references/examples.md` : 8 complete working examples (user creation, GRANT patterns, role workflow, encryption config, TLS server + client, audit-log plug-in).
- `references/anti-patterns.md` : 8 documented anti-patterns with WHY-it-fails and the correct alternative.
- KB ed25519 plug-in : https://mariadb.com/kb/en/authentication-plugin-ed25519/
- KB GRANT statement : https://mariadb.com/kb/en/grant/
- KB roles overview : https://mariadb.com/kb/en/roles_overview/
- Docs file-key-management plug-in : https://mariadb.com/docs/server/security/encryption/data-at-rest-encryption/key-management-and-encryption-plugins/file-key-management-encryption-plugin.md
- Docs mysql.global_priv table : https://mariadb.com/docs/server/reference/system-tables/the-mysql-database-tables/mysql-global_priv-table.md
- KB CHANGE MASTER TO : https://mariadb.com/kb/en/change-master-to/
