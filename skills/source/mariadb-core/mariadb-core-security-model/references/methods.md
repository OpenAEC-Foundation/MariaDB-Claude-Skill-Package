# MariaDB Security Methods

Complete syntax reference for authentication plug-ins, the GRANT model, role lifecycle, encryption-at-rest variables, and TLS configuration. Every entry is annotated with its applicable version range.

## 1. Authentication Plug-in Catalogue

### mysql_native_password

```sql
-- All supported versions (legacy, SHA-1 challenge-response)
INSTALL SONAME 'mysql_native_password';     -- usually already installed

CREATE USER 'legacy_app'@'10.0.0.5'
  IDENTIFIED WITH mysql_native_password
  USING PASSWORD('secret');

-- Or via the older shortcut (still accepted)
CREATE USER 'legacy_app'@'10.0.0.5' IDENTIFIED BY 'secret';
```

Security trade-off : SHA-1 is cryptographically broken for collision resistance. Acceptable only for clients that cannot negotiate a stronger plug-in.

### ed25519

```sql
-- 10.1.21+ : modern default recommendation
INSTALL SONAME 'auth_ed25519';

-- Persistent loading
-- [mariadb]
-- plugin_load_add = auth_ed25519

-- User creation : server hashes the cleartext via the ed25519 routine
CREATE USER 'alice'@'%' IDENTIFIED VIA ed25519 USING PASSWORD('secret');

-- Or supply a pre-computed hash from the ed25519_password UDF
CREATE FUNCTION ed25519_password RETURNS STRING SONAME 'auth_ed25519.so';
SELECT ed25519_password('secret');
-- Output : ZIgUREUg5PVgQ6LskhXmO+eZLS0nC8be6HPjYWR4YJY

CREATE USER 'bob'@'%' IDENTIFIED VIA ed25519
  USING 'ZIgUREUg5PVgQ6LskhXmO+eZLS0nC8be6HPjYWR4YJY';
```

Security trade-off : Elliptic Curve DSA (same as OpenSSH). The legacy `PASSWORD()` function does NOT work with ed25519 ; use the `ed25519_password()` UDF or `USING PASSWORD('secret')` shortcut.

### unix_socket

```sql
-- 10.0+ : OS-uid to DB-identity mapping
INSTALL SONAME 'auth_socket';

-- Map OS user 'mysql' to DB user 'root'@'localhost'
CREATE USER 'root'@'localhost' IDENTIFIED VIA unix_socket;

-- Combine with PASSWORD for hybrid (socket OR password)
CREATE USER 'root'@'localhost'
  IDENTIFIED VIA unix_socket OR mysql_native_password USING PASSWORD('secret');
```

Security trade-off : fails closed on TCP. Use ONLY for local OS-mapped access.

### gssapi (Kerberos / AD)

```sql
-- 10.1+ : Kerberos / SSPI for AD-joined hosts
INSTALL SONAME 'auth_gssapi';

CREATE USER 'alice'@'%' IDENTIFIED VIA gssapi
  USING 'alice@EXAMPLE.COM';      -- Kerberos principal
```

Security trade-off : depends on AD / Kerberos KDC availability. Configure principal mapping via `gssapi_principal_name`.

### pam

```sql
-- 10.0+ : Linux PAM for LDAP / OTP / two-factor
INSTALL SONAME 'auth_pam';

CREATE USER 'alice'@'%' IDENTIFIED VIA pam
  USING 'mariadb';                -- /etc/pam.d/mariadb service stack
```

Security trade-off : requires the `mysql` server user to have read access to the PAM stack ; on some distros add to the `shadow` group.

## 2. Full GRANT Privilege List

### Privileges by scope

| Scope | Privilege | What it allows |
|-------|-----------|----------------|
| Global (`*.*`) | `ALL`, `SUPER`, `FILE`, `PROCESS`, `RELOAD`, `SHUTDOWN`, `REPLICATION CLIENT`, `REPLICATION SLAVE`, `CREATE USER`, `BINLOG ADMIN`, `BINLOG REPLAY`, `READ_ONLY ADMIN`, `SET USER`, `FEDERATED ADMIN` (11.x+) | Server-level operations |
| Database (`db.*`) | `CREATE`, `DROP`, `ALTER`, `INDEX`, `EVENT`, `TRIGGER`, `LOCK TABLES`, `REFERENCES`, `CREATE TEMPORARY TABLES`, `CREATE VIEW`, `SHOW VIEW`, `CREATE ROUTINE`, `ALTER ROUTINE`, `EXECUTE` | Per-database DDL and routines |
| Table (`db.tbl`) | `SELECT`, `INSERT`, `UPDATE`, `DELETE` (also valid at db scope) | Per-table DML |
| Column (`db.tbl(col)`) | `SELECT(col_list)`, `INSERT(col_list)`, `UPDATE(col_list)`, `REFERENCES(col_list)` | Per-column access |
| Routine (`PROCEDURE name`, `FUNCTION name`) | `EXECUTE`, `ALTER ROUTINE`, `GRANT OPTION` | Per-routine access |

### REQUIRE TLS options

| Clause | Meaning | Implies |
|--------|---------|---------|
| `REQUIRE NONE` | TLS optional | nothing |
| `REQUIRE SSL` | TLS mandatory, no cert validation | nothing |
| `REQUIRE X509` | TLS mandatory + client must present any valid X509 | `SSL` |
| `REQUIRE SUBJECT 'subject'` | TLS + client cert must match subject DN | `X509` |
| `REQUIRE ISSUER 'issuer'` | TLS + client cert must be issued by issuer DN | `X509` |
| `REQUIRE CIPHER 'cipher'` | TLS + specific cipher suite negotiated | `SSL` |

Clauses combine with `AND` ; the `REQUIRE` keyword appears once per statement.

### Resource limits (`WITH` clause)

| Limit | Default | Behaviour |
|-------|---------|-----------|
| `MAX_QUERIES_PER_HOUR n` | 0 (unlimited) | Caps any statement type per hour |
| `MAX_UPDATES_PER_HOUR n` | 0 | Caps `INSERT` / `UPDATE` / `DELETE` per hour |
| `MAX_CONNECTIONS_PER_HOUR n` | 0 | Caps connection attempts per hour |
| `MAX_USER_CONNECTIONS n` | 0 | Caps simultaneous open connections |
| `MAX_STATEMENT_TIME seconds` | 0 (no timeout) | Server kills long queries (10.1+) |

Counters reset hourly. `WITH MAX_USER_CONNECTIONS 0` falls back to the global `max_user_connections`.

## 3. Role Lifecycle

```sql
-- 10.0.5+
CREATE ROLE role_name [WITH ADMIN {CURRENT_USER | CURRENT_ROLE | user | role}];

-- Privilege management on the role
GRANT SELECT ON app.* TO role_name;
REVOKE INSERT ON app.* FROM role_name;

-- Assign role to user (or to another role for indirect inheritance)
GRANT role_name TO 'alice'@'%' [WITH ADMIN OPTION];

-- Activate role explicitly in a session
SET ROLE role_name;
SET ROLE NONE;            -- deactivate all active roles
SET ROLE ALL;             -- activate every granted role

-- Set default role activated on connect (10.1.1+)
SET DEFAULT ROLE role_name [FOR user];
SET DEFAULT ROLE NONE FOR 'alice'@'%';

-- Inspect role state
SELECT CURRENT_ROLE();
SELECT * FROM mysql.roles_mapping WHERE User = 'alice';

-- Remove
REVOKE role_name FROM 'alice'@'%';
DROP ROLE role_name;
```

**Constraint** : Only roles granted directly to a user can be set with `SET ROLE`. Roles granted to a role are NOT directly settable, but their privileges cascade transparently to anyone with the parent role active.

## 4. Encryption-at-Rest Variable Matrix

| Variable | 10.6 | 10.11 | 11.x | 12.x | Notes |
|----------|------|-------|------|------|-------|
| `plugin_load_add = file_key_management` | yes | yes | yes | yes | Load the plug-in |
| `file_key_management_filename` | yes | yes | yes | yes | Path to key file (required) |
| `file_key_management_filekey` | yes | yes | yes | yes | Password or `FILE:path` for encrypted key file |
| `file_key_management_encryption_algorithm` | `AES_CBC` (default), `AES_CTR`, `AES_GCM` (build-dependent) | same | same | same | `AES_CTR` recommended for new deployments |
| `file_key_management_use_pbkdf2` | no | no | no | yes (12.0.1+) | PBKDF2 iteration count ; default 0 (off) |
| `file_key_management_digest` | no | no | no | yes (12.0.1+) | Digest function for key derivation ; default `sha1`, prefer `sha256` |
| `innodb_encrypt_tables` | `ON`/`OFF`/`FORCE` | same | same | same | `FORCE` rejects unencrypted CREATE / ALTER |
| `innodb_encrypt_log` | yes | yes | yes | yes | Encrypt redo log |
| `innodb_encrypt_temporary_tables` | yes | yes | yes | yes | Encrypt InnoDB internal temp tables |
| `encrypt_binlog` | yes | yes | yes | yes | Encrypt binlog (replication metadata) |
| `encrypt_tmp_files` | yes | yes | yes | yes | Encrypt sort / aggregate spill files |
| `encrypt_tmp_disk_tables` | yes | yes | yes | yes | Encrypt disk-based MEMORY temp tables |
| `aria_encrypt_tables` | yes | yes | yes | yes | Encrypt Aria tablespaces (used by mysql.* tables) |

### Alternative key plug-ins

| Plug-in | SONAME | Use when |
|---------|--------|----------|
| `file_key_management` | `file_key_management.so` | Standalone server with local key file |
| `aws_key_management` | `aws_key_management.so` | AWS environment with KMS-backed keys |
| HashiCorp Vault (third-party) | `hashicorp_key_management.so` | Centralised secret-store with rotation |
| `eperi_gatekeeper` (third-party) | varies | Compliance-driven environments |

## 5. TLS Server Variables

| Variable | Purpose | Notes |
|----------|---------|-------|
| `ssl_ca` | Path to CA cert (PEM) | Required for client cert validation |
| `ssl_capath` | Directory of CA certs | Alternative to `ssl_ca` |
| `ssl_cert` | Server cert (PEM, full chain) | Without intermediates, handshakes fail silently |
| `ssl_key` | Server private key (PEM) | Chmod `0400 mysql:mysql` |
| `ssl_cipher` | OpenSSL cipher list (TLS 1.2 and below) | Use `:` separator |
| `tls_version` | Acceptable protocols (`TLSv1.2,TLSv1.3`) | Always pin ; defaults still include older versions on some builds |
| `tls_ciphersuites` | TLS 1.3 cipher list (11.x+) | Separate from `ssl_cipher` |
| `require_secure_transport` | Reject non-TLS connections server-wide (10.5+) | Global on/off, overrides per-user `REQUIRE NONE` |

## 6. mysql.global_priv Schema (10.4+)

```sql
DESCRIBE mysql.global_priv;
-- +-------+--------------+------+-----+---------+
-- | Field | Type         | Null | Key | Default |
-- +-------+--------------+------+-----+---------+
-- | Host  | char(60)     | NO   | PRI |         |
-- | User  | char(80)     | NO   | PRI |         |
-- | Priv  | longtext     | NO   |     | {}      |
-- +-------+--------------+------+-----+---------+

-- Example Priv JSON
SELECT JSON_PRETTY(Priv) FROM mysql.global_priv WHERE User = 'alice';
-- {
--   "access": 32768,
--   "plugin": "ed25519",
--   "authentication_string": "ZIgUREUg5PVgQ6LskhXmO+eZLS0nC8be6HPjYWR4YJY",
--   "password_last_changed": 1717113600,
--   "account_locked": false,
--   "version_id": 110400
-- }
```

The JSON `access` field is a bitmask. The JSON `version_id` field (10.5.2+) tracks which server version last wrote the row, useful when diagnosing upgrade issues.

## 7. mariadb-upgrade

```bash
# After every major-version binary upgrade : run EXACTLY ONCE
sudo mariadb-upgrade -u root -p

# Skip the system-table upgrade (rare ; only if you ran it earlier)
sudo mariadb-upgrade --upgrade-system-tables=0 -u root -p
```

The tool migrates `mysql.user` rows into `mysql.global_priv`, rebuilds the `mysql.user` compatibility view, and refreshes `mysql_system_tables_data.sql`. Running it twice in a row is idempotent but logs spurious warnings that can mask real errors from the first pass.
