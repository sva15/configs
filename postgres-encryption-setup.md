# PostgreSQL Encryption Setup: In-Transit (TLS) + At-Rest

**Environment:** PostgreSQL running as a Docker container on an EC2 instance  
**Client:** Python Lambda function connecting via `host:5432` with username/password

---

## Table of Contents

1. [Encryption At Rest](#1-encryption-at-rest)
2. [Encryption In Transit — TLS/SSL](#2-encryption-in-transit--tlsssl)
   - [Step 1: Generate Self-Signed Certificates on EC2](#step-1--generate-self-signed-certificates-on-ec2)
   - [Step 2: Fix Permissions](#step-2--fix-permissions-critical)
   - [Step 3: Copy Certs into the Running Container](#step-3--copy-certs-into-the-running-container)
   - [Step 4: Update postgresql.conf](#step-4--update-postgresqlconf-inside-the-container)
   - [Step 5: Update pg_hba.conf](#step-5--update-pg_hbaconf-allow-both-plain--ssl)
   - [Step 6: Reload Config](#step-6--reload-postgres-config-no-restart-needed)
   - [Step 7: Verify SSL](#step-7--verify-ssl-is-working)
   - [Step 8: Update Lambda Connection](#step-8--update-lambda-to-use-ssl)
3. [Architecture Summary](#3-architecture-summary)
4. [Quick Wins & Security Hardening](#4-quick-wins--security-hardening)

---

## 1. Encryption At Rest

### EBS Volume Encryption

Your EBS volume is already encrypted — **this is sufficient** for the at-rest requirement.

**What EBS encryption covers:**
- All PostgreSQL data files written to disk (AES-256 via KMS)
- WAL (Write-Ahead Log) files
- Temp files and buffers flushed to disk
- EBS snapshots are also encrypted automatically
- Zero application changes required — fully transparent

**Compliance:** Satisfies most frameworks (SOC2, PCI-DSS, HIPAA) for encryption at rest.

### Do You Also Need pgcrypto (Field-Level Encryption)?

Only add PostgreSQL-level encryption on top of EBS if you have specific threat models:

| Scenario | Need pgcrypto? |
|---|---|
| General "encrypt data at rest" compliance requirement | ❌ EBS is enough |
| Protection from someone stealing the physical disk | ❌ EBS is enough |
| AWS insider threat accessing your EBS snapshot directly | ✅ Yes |
| Postgres superuser must NOT see specific columns (SSN, PII) | ✅ Yes |
| Regulatory requirement for **field-level** encryption specifically | ✅ Yes |

**For your setup** (Lambda → Postgres on EC2), EBS encryption alone is the right call. pgcrypto adds complexity — key management in app code, slower queries, harder to index — that isn't justified without a specific field-level requirement.

---

## 2. Encryption In Transit — TLS/SSL

### Goal

- ✅ Keep existing username/password connections working (plain TCP)
- ✅ Also allow SSL/TLS encrypted connections
- Both modes coexist — no existing connections break

> **Note:** "SSL" and "TLS" are used interchangeably in PostgreSQL documentation. Modern PostgreSQL uses TLS 1.2+ under the hood. This is different from SSH (which is for server logins).

---

### Step 1 — Generate Self-Signed Certificates on EC2

SSH into your EC2 instance and run:

```bash
mkdir -p ~/postgres-ssl && cd ~/postgres-ssl

# 1. Generate CA private key
openssl genrsa -out ca.key 4096

# 2. Generate CA self-signed certificate (valid 10 years)
openssl req -new -x509 -days 3650 -key ca.key -out ca.crt \
  -subj "/CN=postgres-ca"

# 3. Generate server private key
openssl genrsa -out server.key 4096

# 4. Generate server Certificate Signing Request (CSR)
#    Replace <EC2_PRIVATE_IP> with your actual EC2 private IP
openssl req -new -key server.key -out server.csr \
  -subj "/CN=<EC2_PRIVATE_IP>"

# 5. Sign the server cert with your CA
openssl x509 -req -days 3650 \
  -in server.csr \
  -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out server.crt

# Verify all files are present
ls -la ~/postgres-ssl/
# Expected: ca.crt  ca.key  ca.srl  server.crt  server.csr  server.key
```

---

### Step 2 — Fix Permissions (Critical)

PostgreSQL will **refuse to start** if key file permissions are too open.

```bash
# PostgreSQL container runs as UID 999
chmod 600 server.key          # private key — owner read only
chmod 644 server.crt ca.crt   # certs — world readable is fine
chown 999:999 server.key server.crt ca.crt
```

---

### Step 3 — Copy Certs into the Running Container

```bash
# Find your container name
docker ps

# Copy all three cert files into the container
docker cp ~/postgres-ssl/server.crt  <container_name>:/var/lib/postgresql/server.crt
docker cp ~/postgres-ssl/server.key  <container_name>:/var/lib/postgresql/server.key
docker cp ~/postgres-ssl/ca.crt      <container_name>:/var/lib/postgresql/ca.crt

# Fix ownership inside the container
docker exec -it <container_name> bash -c "
  chown postgres:postgres \
    /var/lib/postgresql/server.crt \
    /var/lib/postgresql/server.key \
    /var/lib/postgresql/ca.crt &&
  chmod 600 /var/lib/postgresql/server.key &&
  chmod 644 /var/lib/postgresql/server.crt /var/lib/postgresql/ca.crt
"

# Verify
docker exec -it <container_name> ls -la \
  /var/lib/postgresql/server.crt \
  /var/lib/postgresql/server.key \
  /var/lib/postgresql/ca.crt
```

---

### Step 4 — Update postgresql.conf Inside the Container

```bash
# Open a shell inside the container
docker exec -it <container_name> bash

# Confirm where postgresql.conf lives
psql -U postgres -c "SHOW config_file;"
# Typically: /var/lib/postgresql/data/postgresql.conf

# Append SSL configuration
cat >> /var/lib/postgresql/data/postgresql.conf << 'EOF'

# -----------------------------------------------
# SSL / TLS Configuration
# -----------------------------------------------
ssl = on
ssl_cert_file = '/var/lib/postgresql/server.crt'
ssl_key_file  = '/var/lib/postgresql/server.key'
ssl_ca_file   = '/var/lib/postgresql/ca.crt'
ssl_min_protocol_version = 'TLSv1.2'
EOF

exit
```

---

### Step 5 — Update pg_hba.conf (Allow Both Plain + SSL)

```bash
docker exec -it <container_name> bash

# View current rules first to avoid duplicating host lines
cat /var/lib/postgresql/data/pg_hba.conf

# Append both rules
cat >> /var/lib/postgresql/data/pg_hba.conf << 'EOF'

# Allow SSL (encrypted) connections with password
hostssl   all   all   0.0.0.0/0   scram-sha-256

# Allow plain (unencrypted) connections with password — preserves existing behavior
host      all   all   0.0.0.0/0   scram-sha-256
EOF

exit
```

> **Note:** If `host` lines already exist in pg_hba.conf, just add the `hostssl` line above them — don't duplicate the `host` rule.

**pg_hba.conf Rule Reference:**

| Rule type | Meaning |
|---|---|
| `hostssl` | Only accepts SSL/TLS encrypted connections |
| `host` | Accepts both plain and SSL connections |
| `hostnossl` | Only accepts plain (unencrypted) connections |

To **enforce SSL-only** later (after confirming everything works), remove the `host` line and keep only `hostssl`.

---

### Step 6 — Reload Postgres Config (No Restart Needed)

```bash
# Reload without dropping existing connections
docker exec -it <container_name> psql -U postgres -c "SELECT pg_reload_conf();"

# Confirm SSL is active
docker exec -it <container_name> psql -U postgres -c "SHOW ssl;"
# Expected output:
#  ssl
# -----
#  on
```

---

### Step 7 — Verify SSL is Working

**Test SSL connection (from EC2 or any machine with ca.crt):**

```bash
psql "host=<EC2_IP> port=5432 dbname=yourdb user=youruser \
      sslmode=verify-ca sslrootcert=~/postgres-ssl/ca.crt" \
      --password
```

**Test plain connection (existing behavior still works):**

```bash
psql "host=<EC2_IP> port=5432 dbname=yourdb user=youruser sslmode=disable" \
      --password
```

**Check SSL status from inside an active session:**

```sql
SELECT ssl, version, cipher, bits
FROM pg_stat_ssl
WHERE pid = pg_backend_pid();
```

Expected when connected via SSL:

```
 ssl | version |         cipher         | bits
-----+---------+------------------------+------
 t   | TLSv1.3 | TLS_AES_256_GCM_SHA384 |  256
```

---

### Step 8 — Update Lambda to Use SSL

Upload `ca.crt` to an S3 bucket, then update your Lambda connection code:

```python
import boto3
import psycopg2
import tempfile
import os

# Download CA cert once at Lambda cold start
def get_ca_cert():
    s3 = boto3.client('s3')
    tmp = tempfile.NamedTemporaryFile(delete=False, suffix='.crt')
    s3.download_fileobj('your-certs-bucket', 'postgres-ssl/ca.crt', tmp)
    tmp.close()
    return tmp.name

# Runs once per cold start — reused across warm invocations
CA_CERT_PATH = get_ca_cert()

def get_connection():
    return psycopg2.connect(
        host="<EC2_PRIVATE_IP>",
        port=5432,
        dbname="yourdb",
        user="youruser",
        password="yourpassword",
        sslmode="verify-ca",        # encrypts + validates the CA cert
        sslrootcert=CA_CERT_PATH
    )
```

**`sslmode` options explained:**

| sslmode | Encrypts? | Validates CA? | Validates Hostname? | Use When |
|---|---|---|---|---|
| `disable` | ❌ | ❌ | ❌ | Local dev only |
| `require` | ✅ | ❌ | ❌ | Encrypts but no cert check |
| `verify-ca` | ✅ | ✅ | ❌ | **Recommended minimum for prod** |
| `verify-full` | ✅ | ✅ | ✅ | Use if CN matches your hostname |

---

## 3. Architecture Summary

```
Lambda (Python / psycopg2)
  │
  │  sslmode=verify-ca
  │  ca.crt downloaded from S3 at cold start
  │
  ▼
EC2 Security Group
  └─ Port 5432 open to Lambda Security Group only (not 0.0.0.0/0)
  │
  ▼
Docker → PostgreSQL Container
  ├─ ssl=on in postgresql.conf
  ├─ hostssl rule in pg_hba.conf
  └─ server.crt / server.key / ca.crt mounted at /var/lib/postgresql/
  │
  ▼
EBS Volume (AES-256 encrypted via AWS KMS)
  └─ All data files, WAL logs, temp files encrypted at rest
```

**Connection modes after setup:**

| Connection Type | pg_hba.conf Rule | Lambda sslmode |
|---|---|---|
| Encrypted (SSL) | `hostssl` | `verify-ca` |
| Plain (existing, preserved) | `host` | `disable` or `prefer` |

---

## 4. Quick Wins & Security Hardening

| Action | Why |
|---|---|
| Move credentials to **AWS Secrets Manager** | Rotate passwords without Lambda redeploy |
| Lock EC2 Security Group to **Lambda's Security Group** only | No public database exposure — not open to 0.0.0.0/0 |
| Remove `host` rule from pg_hba.conf once SSL is confirmed | Enforce SSL-only; reject all plain connections |
| Store `ca.crt` in **S3 with bucket policy** (not hardcoded in Lambda) | Easier cert rotation |
| Set **cert expiry reminders** (certs above are 3650 days / ~10 years) | Avoid silent expiry breaking connections |
| Enable **CloudTrail + VPC Flow Logs** | Audit who connected and when |
| Migrate credentials to **IAM auth** if you move to RDS later | Eliminates passwords entirely |

---

*Generated for: PostgreSQL on EC2 (Docker) + Python Lambda client*
