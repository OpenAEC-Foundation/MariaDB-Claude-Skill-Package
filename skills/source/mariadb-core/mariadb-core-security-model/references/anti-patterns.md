# MariaDB Security Anti-Patterns

Documented anti-patterns with WHY they fail, the underlying root cause, and the correct alternative. Every entry has a verified MariaDB KB or docs source.

## Anti-pattern 1 : GRANT ALL ON *.* TO 'app_user'@'%'

### The mistake

```sql
-- AVOID : grants every privilege at server level
CREATE USER 'app_user'@'%' IDENTIFIED BY 'secret';
GRANT ALL PRIVILEGES ON *.* TO 'app_user'@'%' WITH GRANT OPTION;
```

### Why it fails

`ALL PRIVILEGES` at global scope includes `SUPER`, `FILE`, `PROCESS`, `RELOAD`, `SHUTDOWN`, `CREATE USER`, `REPLICATION CLIENT`, `REPLICATION SLAVE`. An SQL-injection vulnerability in any code path that connects as `app_user` immediately escalates to a server takeover : the attacker can `SELECT INTO OUTFILE` arbitrary files (FILE), kill the server (SHUTDOWN), create new admin users (CREATE USER + GRANT OPTION), or seed a malicious replica (REPLICATION SLAVE). The `'%'` host means the credential is also accepted from anywhere on the network if it leaks.

### Correct alternative

```sql
-- DO : scope the user to one database, pin to a known host range
CREATE USER 'app_user'@'10.0.0.%' IDENTIFIED VIA ed25519
  USING PASSWORD('secret');
GRANT SELECT, INSERT, UPDATE, DELETE ON shop.* TO 'app_user'@'10.0.0.%'
  REQUIRE SSL;
```

Reference : https://mariadb.com/kb/en/grant/

## Anti-pattern 2 : mysql_native_password for new accounts

### The mistake

```sql
-- AVOID : new account on a 10.6+ server using legacy SHA-1
CREATE USER 'new_app'@'%' IDENTIFIED BY 'secret';
-- This defaults to mysql_native_password on most builds
```

### Why it fails

`mysql_native_password` is SHA-1 challenge-response. SHA-1 has been collision-broken since 2017 (SHAttered). The hash is also stored as a single SHA-1(SHA-1(password)) value, meaning a leaked `mysql.global_priv.Priv.authentication_string` is offline-crackable with hashcat at GPU speeds. ed25519 has been the modern recommended plug-in since 10.1.21 and uses Elliptic Curve DSA, the same algorithm OpenSSH uses for SSH key pairs.

### Correct alternative

```sql
-- DO : install ed25519 once, then use IDENTIFIED VIA ed25519
INSTALL SONAME 'auth_ed25519';

CREATE USER 'new_app'@'%' IDENTIFIED VIA ed25519
  USING PASSWORD('secret');
```

Reference : https://mariadb.com/kb/en/authentication-plugin-ed25519/

## Anti-pattern 3 : Encrypting only tables, not binlog or tmp

### The mistake

```ini
# AVOID : partial encryption, leaks via uncovered surfaces
[mariadb]
plugin_load_add = file_key_management
loose_file_key_management_filename = /etc/mysql/encryption/keyfile.enc
loose_file_key_management_filekey   = FILE:/etc/mysql/encryption/keyfile.key

innodb_encrypt_tables = ON
# encrypt_binlog       = OFF  (default)
# encrypt_tmp_files    = OFF  (default)
# innodb_encrypt_log   = OFF  (default)
```

### Why it fails

The encryption surfaces are independent. With only `innodb_encrypt_tables=ON`, the on-disk `.ibd` files are encrypted, but :
- The binlog contains row images for every change. An attacker who reads the binlog reconstructs most data.
- The redo log contains recent row changes pre-flush.
- Temp files contain sort and aggregate intermediate results, often the entire table for a `SELECT * ORDER BY`.
- Aria tablespaces (used by `mysql.*` system tables, including authentication data) remain plaintext.

Encryption at rest is a chain ; any unprotected link defeats the chain.

### Correct alternative

```ini
# DO : enable every encryption surface together
[mariadb]
plugin_load_add = file_key_management
loose_file_key_management_filename = /etc/mysql/encryption/keyfile.enc
loose_file_key_management_filekey   = FILE:/etc/mysql/encryption/keyfile.key

innodb_encrypt_tables           = ON
innodb_encrypt_log              = ON
innodb_encrypt_temporary_tables = ON
encrypt_binlog                  = ON
encrypt_tmp_files               = ON
encrypt_tmp_disk_tables         = ON
aria_encrypt_tables             = ON
```

Reference : https://mariadb.com/docs/server/security/encryption/data-at-rest-encryption/

## Anti-pattern 4 : Roles created but no SET DEFAULT ROLE

### The mistake

```sql
-- AVOID : role granted but never activated on connect
CREATE ROLE app_read;
GRANT SELECT ON app.* TO app_read;

CREATE USER 'alice'@'%' IDENTIFIED VIA ed25519 USING PASSWORD('a');
GRANT app_read TO 'alice'@'%';

-- alice connects ... but has zero privileges until she runs SET ROLE app_read;
```

### Why it fails

In MariaDB, roles are NOT automatically active on connect. After `GRANT <role> TO <user>`, the user has the role available, but until they execute `SET ROLE <role_name>` in the session (or `SET ROLE ALL`), no role-derived privileges apply. Application code that connects, runs queries, and disconnects has no opportunity to call `SET ROLE`, so the user effectively has zero application privileges and every query fails with `ERROR 1142 (42000): SELECT command denied`.

### Correct alternative

```sql
-- DO : always set the default role after granting it
GRANT app_read TO 'alice'@'%';
SET DEFAULT ROLE app_read FOR 'alice'@'%';

-- Or for the current user
SET DEFAULT ROLE app_read;
```

Reference : https://mariadb.com/kb/en/set-default-role/

## Anti-pattern 5 : ssl_cert without the full certificate chain

### The mistake

```ini
# AVOID : ssl_cert is the server's leaf cert only, no intermediates
[mariadb]
ssl_cert = /etc/mysql/certs/server-leaf-only.pem
ssl_key  = /etc/mysql/certs/server-key.pem
ssl_ca   = /etc/mysql/certs/ca.pem
```

### Why it fails

Most public CAs issue certs from an intermediate CA, not directly from a root. The TLS handshake requires the server to present its leaf cert PLUS all intermediates so the client can chain back to a root in its trust store. With only the leaf cert in `ssl_cert`, clients with intermediates pre-installed succeed, but clients without (default for most language drivers) get a handshake error like `SSL connection error: unable to get local issuer certificate`. The failure mode is silent and intermittent : developers test from a workstation with the intermediates pre-installed, production servers fail.

### Correct alternative

```bash
# Build the full chain : leaf, then intermediates, then root (optional)
cat server-leaf.pem intermediate.pem root.pem > /etc/mysql/certs/server-cert.pem
```

```ini
[mariadb]
ssl_cert = /etc/mysql/certs/server-cert.pem    # full chain
ssl_key  = /etc/mysql/certs/server-key.pem
ssl_ca   = /etc/mysql/certs/ca.pem
```

Reference : https://mariadb.com/kb/en/securing-connections-for-client-and-server/

## Anti-pattern 6 : File-key-management with world-readable key file

### The mistake

```bash
# AVOID : keyring readable by every user on the host
sudo cp keyfile.enc /etc/mysql/encryption/
sudo cp keyfile.key /etc/mysql/encryption/
ls -l /etc/mysql/encryption/
# -rw-r--r-- 1 root root  98 May 19 14:00 keyfile.enc
# -rw-r--r-- 1 root root  33 May 19 14:00 keyfile.key
```

### Why it fails

The point of encryption at rest is to defend against unauthorised disk access. If the key file is world-readable, every local user (including any compromised application running as `nobody`, `www-data`, etc.) reads the key file and decrypts every protected page. The encryption then provides defence only against someone who steals the disk but not the running OS, a much narrower threat model. Worse, system-wide log scrapers, backup agents, and container layer shippers may exfiltrate the key file as part of routine `/etc` backups.

### Correct alternative

```bash
# DO : root-only ownership, mysql-only read, no group, no world
sudo chown -R mysql:mysql /etc/mysql/encryption
sudo chmod 0400 /etc/mysql/encryption/keyfile.enc
sudo chmod 0400 /etc/mysql/encryption/keyfile.key
ls -l /etc/mysql/encryption/
# -r-------- 1 mysql mysql  98 May 19 14:00 keyfile.enc
# -r-------- 1 mysql mysql  33 May 19 14:00 keyfile.key
```

Bonus : on 12.0.1+ enable PBKDF2 via `file_key_management_use_pbkdf2 = 1000` and `file_key_management_digest = sha256` to slow offline attacks even when the key file leaks.

Reference : https://mariadb.com/docs/server/security/encryption/data-at-rest-encryption/key-management-and-encryption-plugins/file-key-management-encryption-plugin.md

## Anti-pattern 7 : Skipping mariadb-upgrade after a 10.4+ upgrade

### The mistake

```bash
# AVOID : binary upgrade from 10.3 to 10.6, but no mariadb-upgrade
sudo apt install mariadb-server   # upgrades package from 10.3 to 10.6
sudo systemctl start mariadb
# Server boots ... but mysql.user rows have not been migrated to mysql.global_priv
```

### Why it fails

From 10.4 onwards, the authoritative privilege store is `mysql.global_priv` (JSON). The `mysql.user` table becomes a backward-compat view. If you upgrade from 10.3 to 10.4+ without running `mariadb-upgrade`, the server boots, but :
- New `CREATE USER` / `GRANT` statements write to `mysql.global_priv` correctly.
- Old user rows still live in the legacy `mysql.user` storage and are queryable via the view, but updates to them via the view may not propagate correctly.
- Authentication for legacy users may succeed inconsistently because of the dual-store state.

Symptoms include random `ERROR 1045: Access denied` for users that worked before the upgrade, role grants that silently do nothing, and `SHOW GRANTS` returning stale output.

### Correct alternative

```bash
# DO : run mariadb-upgrade EXACTLY ONCE after every major version upgrade
sudo mariadb-upgrade -u root -p

# Then restart so the in-memory grant cache picks up the migrated state
sudo systemctl restart mariadb
```

Reference : https://mariadb.com/kb/en/mariadb-upgrade/

## Anti-pattern 8 : Trusting unix_socket on a shared-host server

### The mistake

```sql
-- AVOID : unix_socket auth on a server that hosts many tenants
CREATE USER 'tenant1_admin'@'localhost' IDENTIFIED VIA unix_socket;
GRANT ALL PRIVILEGES ON tenant1.* TO 'tenant1_admin'@'localhost';
```

### Why it fails

`unix_socket` maps the OS uid of the connecting process to the DB user. On a single-tenant dedicated host this is safe. On a shared host (multiple tenants, container orchestrator, cron jobs running as various users, web hosting panel), any process running as the matching OS user authenticates as the DB user with no password challenge. A privilege escalation in a sidecar container, a misconfigured cron job, or any local-user code-execution vulnerability bypasses the database password entirely.

### Correct alternative

```sql
-- DO : on shared hosts, use ed25519 with REQUIRE SSL and a strong password
CREATE USER 'tenant1_admin'@'localhost' IDENTIFIED VIA ed25519
  USING PASSWORD('strong-secret-2026');
GRANT ALL PRIVILEGES ON tenant1.* TO 'tenant1_admin'@'localhost'
  REQUIRE SSL;
```

Reserve `unix_socket` for :
- `'root'@'localhost'` on a single-tenant Debian / Ubuntu install (the package default and the safest local-admin path).
- Automation users that run as a dedicated OS user with no shell.

Reference : https://mariadb.com/kb/en/authentication-plugin-unix-socket/
