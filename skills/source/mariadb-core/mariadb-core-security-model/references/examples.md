# MariaDB Security Examples

Working end-to-end examples for user creation, GRANT patterns, role workflows, encryption at rest, and TLS. Every example is version-annotated.

## Example 1 : Read-only application user with ed25519 and TLS

```sql
-- 10.6+ : analytics user that can only SELECT from one database, over TLS
INSTALL SONAME 'auth_ed25519';

CREATE USER 'analytics_ro'@'10.0.0.%' IDENTIFIED VIA ed25519
  USING PASSWORD('analytics-secret-2026');

GRANT SELECT ON warehouse.* TO 'analytics_ro'@'10.0.0.%'
  REQUIRE SSL
  WITH MAX_QUERIES_PER_HOUR 10000
       MAX_USER_CONNECTIONS 20
       MAX_STATEMENT_TIME 30;

FLUSH PRIVILEGES;
```

Result : the user can only SELECT from `warehouse.*`, must connect with TLS, is capped at 10k queries / hour and 20 simultaneous connections, and any single query running over 30 seconds is killed by the server.

## Example 2 : Application write user with column-level masking

```sql
-- 10.6+ : app user that can write to orders but cannot read the credit_card column
CREATE USER 'app_write'@'10.0.0.10' IDENTIFIED VIA ed25519
  USING PASSWORD('write-secret-2026');

GRANT INSERT, UPDATE, DELETE ON shop.orders TO 'app_write'@'10.0.0.10';

-- Allow SELECT on every column EXCEPT credit_card
GRANT SELECT (id, customer_id, total, status, created_at)
  ON shop.orders TO 'app_write'@'10.0.0.10';

-- Read-only on the catalogue
GRANT SELECT ON shop.products TO 'app_write'@'10.0.0.10';
GRANT SELECT ON shop.customers TO 'app_write'@'10.0.0.10';
```

Result : `app_write` can mutate orders but cannot read `credit_card` column. Column-level GRANT enforces the masking at the server, not the application.

## Example 3 : Role-based access with three tiers

```sql
-- 10.0.5+ : three roles for read, write, and admin tiers
CREATE ROLE app_read;
CREATE ROLE app_write;
CREATE ROLE app_admin;

-- Privileges per role
GRANT SELECT ON shop.* TO app_read;

GRANT app_read TO app_write;            -- write inherits read
GRANT INSERT, UPDATE, DELETE ON shop.* TO app_write;

GRANT app_write TO app_admin;           -- admin inherits write
GRANT CREATE, DROP, ALTER, INDEX, EVENT, TRIGGER ON shop.* TO app_admin;

-- Assign roles to users and make them active on connect
CREATE USER 'alice'@'%' IDENTIFIED VIA ed25519 USING PASSWORD('a');
CREATE USER 'bob'@'%'   IDENTIFIED VIA ed25519 USING PASSWORD('b');
CREATE USER 'carol'@'%' IDENTIFIED VIA ed25519 USING PASSWORD('c');

GRANT app_read  TO 'alice'@'%';   SET DEFAULT ROLE app_read  FOR 'alice'@'%';
GRANT app_write TO 'bob'@'%';     SET DEFAULT ROLE app_write FOR 'bob'@'%';
GRANT app_admin TO 'carol'@'%';   SET DEFAULT ROLE app_admin FOR 'carol'@'%';
```

Result : adding a new privilege only requires editing the role, not every user. `bob` automatically gets `app_read` privileges via the role chain.

## Example 4 : Enable encryption at rest with file-key-management

```ini
# /etc/mysql/mariadb.conf.d/encryption.cnf (MariaDB 10.6+)
[mariadb]
plugin_load_add = file_key_management
loose_file_key_management_filename = /etc/mysql/encryption/keyfile.enc
loose_file_key_management_filekey   = FILE:/etc/mysql/encryption/keyfile.key
loose_file_key_management_encryption_algorithm = AES_CTR

# 12.0.1+ : harden key derivation
loose_file_key_management_use_pbkdf2 = 1000
loose_file_key_management_digest     = sha256

# Encrypt every surface
innodb_encrypt_tables           = ON
innodb_encrypt_log              = ON
innodb_encrypt_temporary_tables = ON
encrypt_binlog                  = ON
encrypt_tmp_files               = ON
encrypt_tmp_disk_tables         = ON
aria_encrypt_tables             = ON

# Key rotation cadence
innodb_encryption_rotate_key_age = 14   # rotate after 14 days
innodb_encryption_threads        = 4
```

```bash
# Generate the keyring
sudo mkdir -p /etc/mysql/encryption
sudo openssl rand -hex 32 > /tmp/key-material
sudo bash -c 'echo "1;$(cat /tmp/key-material)" > /etc/mysql/encryption/keyfile.enc'
sudo openssl rand -hex 16 > /etc/mysql/encryption/keyfile.key
sudo chown -R mysql:mysql /etc/mysql/encryption
sudo chmod 0400 /etc/mysql/encryption/keyfile.enc /etc/mysql/encryption/keyfile.key
sudo systemctl restart mariadb

# Verify
mariadb -e "SHOW STATUS LIKE 'innodb_encryption%';"
```

Result : every InnoDB tablespace, redo log entry, binlog event, and temp file is encrypted on disk. Existing tables are encrypted over time by the `innodb_encryption_threads`.

## Example 5 : Encrypt an existing table

```sql
-- 10.6+ : in-place re-encryption for a single table
ALTER TABLE shop.orders ENCRYPTED = YES;

-- Verify
SELECT NAME, ENCRYPTION_SCHEME
  FROM information_schema.INNODB_TABLESPACES_ENCRYPTION
  WHERE NAME = 'shop/orders';
```

Result : the table tablespace is re-written with encryption. Use `ENCRYPTED = NO` to remove encryption.

## Example 6 : Enable server-side TLS and require it per user

```ini
# /etc/mysql/mariadb.conf.d/tls.cnf (MariaDB 10.6+)
[mariadb]
ssl_ca   = /etc/mysql/certs/ca.pem
ssl_cert = /etc/mysql/certs/server-cert.pem
ssl_key  = /etc/mysql/certs/server-key.pem
tls_version = TLSv1.2,TLSv1.3
require_secure_transport = ON
```

```sql
-- Force TLS per account (combine with the server config above)
CREATE USER 'secure_app'@'%' IDENTIFIED VIA ed25519
  USING PASSWORD('secret');
GRANT SELECT, INSERT, UPDATE, DELETE ON app.* TO 'secure_app'@'%'
  REQUIRE SSL;

-- Stricter : require an X509 client cert with specific subject
CREATE USER 'cert_pinned'@'%' IDENTIFIED VIA ed25519
  USING PASSWORD('secret');
GRANT SELECT ON app.* TO 'cert_pinned'@'%'
  REQUIRE SUBJECT '/CN=cert_pinned/O=Acme/C=NL'
  AND ISSUER '/C=NL/O=Acme CA/CN=Root CA'
  AND CIPHER 'ECDHE-RSA-AES256-GCM-SHA384';
```

```bash
# Verify from a client
mariadb -u secure_app -p --ssl-ca=/etc/mysql/certs/ca.pem \
  --ssl-verify-server-cert \
  -h db.example.com app -e "STATUS;"
# Look for : SSL: Cipher in use is ECDHE-RSA-AES256-GCM-SHA384
```

Result : TCP connections without TLS are rejected. `secure_app` must connect with TLS. `cert_pinned` must additionally present an X509 cert with the specific subject and issuer.

## Example 7 : TLS for replication with verify-server-cert

```sql
-- On the replica (10.6+)
STOP SLAVE;
CHANGE MASTER TO
   MASTER_HOST                  = 'primary.example.com',
   MASTER_USER                  = 'repl',
   MASTER_PASSWORD              = 'repl-secret',
   MASTER_SSL                   = 1,
   MASTER_SSL_CA                = '/etc/mysql/certs/ca.pem',
   MASTER_SSL_CERT              = '/etc/mysql/certs/replica-cert.pem',
   MASTER_SSL_KEY               = '/etc/mysql/certs/replica-key.pem',
   MASTER_SSL_VERIFY_SERVER_CERT = 1,
   MASTER_USE_GTID              = slave_pos;
START SLAVE;

SHOW SLAVE STATUS\G
-- Verify : Master_SSL_Allowed = Yes, Master_SSL_Verify_Server_Cert = Yes
```

```sql
-- 12.3+ shortcut : inherit server-level TLS config
CHANGE MASTER TO
   MASTER_HOST                   = 'primary.example.com',
   MASTER_USER                   = 'repl',
   MASTER_PASSWORD               = 'repl-secret',
   MASTER_SSL                    = 1,
   MASTER_SSL_CA                 = DEFAULT,
   MASTER_SSL_CERT               = DEFAULT,
   MASTER_SSL_KEY                = DEFAULT,
   MASTER_SSL_VERIFY_SERVER_CERT = 1;
```

Result : the replica connects to the primary over TLS, validates the primary's cert chain against `MASTER_SSL_CA`, and refuses MITM cert substitution.

## Example 8 : Audit user activity via the audit plug-in

```sql
-- 10.0+ : load the audit plug-in (ships with MariaDB)
INSTALL SONAME 'server_audit';

-- Configure via my.cnf for persistence
-- [mariadb]
-- server_audit_logging = ON
-- server_audit_events = CONNECT,QUERY,TABLE
-- server_audit_file_path = /var/log/mysql/audit.log
-- server_audit_file_rotate_size = 100M
-- server_audit_file_rotations = 9

-- Inspect runtime state
SHOW GLOBAL VARIABLES LIKE 'server_audit%';

-- Sample audit-log line :
-- 20260519 14:32:01,db01,alice,10.0.0.5,42,QUERY,app,'SELECT * FROM orders',0
```

Result : every connect, disconnect, query, and table-access by every user is recorded to `/var/log/mysql/audit.log` with rotation. Compliance-grade trail for SOC-2 / ISO-27001.
