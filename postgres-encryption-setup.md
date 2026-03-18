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
   - [Step 8: Lambda Code + Console Testing](#step-8--lambda-code--console-testing)
   - [Step 9: Test from Lambda Console](#step-9--test-from-lambda-console)
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

### Step 8 — Lambda Code + Console Testing

`ca.crt` is already uploaded to `my-us-east-1-uploads/postgres-certs/ca.crt`.  
Credentials are passed via Lambda environment variables — never hardcoded.

#### Lambda Environment Variables

In Lambda Console → **Configuration → Environment variables**, add:

| Key | Value |
|---|---|
| `S3_BUCKET` | `my-us-east-1-uploads` |
| `S3_KEY` | `postgres-certs/ca.crt` |
| `DB_HOST` | `<EC2_PRIVATE_IP>` |
| `DB_PORT` | `5432` |
| `DB_NAME` | `yourdb` |
| `DB_USER` | `youruser` |
| `DB_PASS` | `yourpassword` |

#### Lambda Code (lambda_function.py)

```python
import boto3
import psycopg2
import tempfile
import json
import os

# ── Config — read from Lambda Environment Variables ────────
S3_BUCKET = os.environ["S3_BUCKET"]           # my-us-east-1-uploads
S3_KEY    = os.environ["S3_KEY"]              # postgres-certs/ca.crt
DB_HOST   = os.environ["DB_HOST"]             # EC2 private IP
DB_PORT   = int(os.environ.get("DB_PORT", "5432"))
DB_NAME   = os.environ["DB_NAME"]             # yourdb
DB_USER   = os.environ["DB_USER"]             # youruser
DB_PASS   = os.environ["DB_PASS"]             # yourpassword
# ───────────────────────────────────────────────────────────

# ── Cold start — runs once per Lambda instance ─────────────
def download_ca_cert():
    s3  = boto3.client("s3")
    tmp = tempfile.NamedTemporaryFile(delete=False, suffix=".crt")
    s3.download_fileobj(S3_BUCKET, S3_KEY, tmp)
    tmp.close()
    return tmp.name

CA_CERT_PATH = download_ca_cert()
# ───────────────────────────────────────────────────────────

def get_connection():
    return psycopg2.connect(
        host=DB_HOST,
        port=DB_PORT,
        dbname=DB_NAME,
        user=DB_USER,
        password=DB_PASS,
        sslmode="verify-ca",
        sslrootcert=CA_CERT_PATH,
        connect_timeout=5
    )

def handle_query(sql):
    """Run a SELECT query and return results."""
    conn   = get_connection()
    cursor = conn.cursor()
    cursor.execute(sql)
    rows   = cursor.fetchall()
    cols   = [desc[0] for desc in cursor.description]
    cursor.close()
    conn.close()
    return {"columns": cols, "rows": rows}

# ── Lambda entrypoint ───────────────────────────────────────
def lambda_handler(event, context):
    """
    AWS Lambda entrypoint — invoked by AWS for every execution.

    Supported actions (pass via test event JSON):
      ping   — checks DB connectivity and confirms SSL is active
      query  — runs a custom SELECT query (pass 'sql' key in event)
    """
    action = event.get("action", "ping")

    try:
        if action == "ping":
            conn   = get_connection()
            cursor = conn.cursor()

            # Confirm SSL is active and get cipher info
            cursor.execute("""
                SELECT ssl, version, cipher
                FROM pg_stat_ssl
                WHERE pid = pg_backend_pid()
            """)
            ssl_info = cursor.fetchone()

            # Get Postgres server version
            cursor.execute("SELECT version()")
            pg_version = cursor.fetchone()[0]

            cursor.close()
            conn.close()

            return {
                "status":      "ok",
                "action":      "ping",
                "ssl":         ssl_info[0],     # True/False
                "tls_version": ssl_info[1],     # e.g. TLSv1.3
                "cipher":      ssl_info[2],     # e.g. TLS_AES_256_GCM_SHA384
                "pg_version":  pg_version
            }

        elif action == "query":
            sql = event.get("sql")
            if not sql:
                return {
                    "status":  "error",
                    "message": "action=query requires a 'sql' field in the event"
                }
            # Safety guard — only allow SELECT via console testing
            if not sql.strip().upper().startswith("SELECT"):
                return {
                    "status":  "error",
                    "message": "Only SELECT queries are allowed via the test console"
                }
            result = handle_query(sql)
            return {
                "status":  "ok",
                "action":  "query",
                "columns": result["columns"],
                "rows":    result["rows"]
            }

        else:
            return {
                "status":  "error",
                "message": f"Unknown action '{action}'. Supported: ping, query"
            }

    except psycopg2.OperationalError as e:
        return {
            "status":  "error",
            "action":  action,
            "message": str(e)
        }
    except Exception as e:
        return {
            "status":  "error",
            "action":  action,
            "message": str(e)
        }
```

#### IAM Policy — Lambda Execution Role

Lambda needs permission to read the cert from S3:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-us-east-1-uploads/postgres-certs/ca.crt"
    }
  ]
}
```

#### `sslmode` Options Reference

| sslmode | Encrypts? | Validates CA? | Validates Hostname? | Use When |
|---|---|---|---|---|
| `disable` | ❌ | ❌ | ❌ | Local dev only |
| `require` | ✅ | ❌ | ❌ | Encrypts but no cert check |
| `verify-ca` | ✅ | ✅ | ❌ | **Recommended minimum for prod** |
| `verify-full` | ✅ | ✅ | ✅ | Use if CN matches your hostname |

---

### Step 9 — Test from Lambda Console

Go to Lambda Console → your function → **Test** tab → **Create new test event**.

#### Test Event 1: Ping (connectivity + SSL check)

```json
{
  "action": "ping"
}
```

Expected response:
```json
{
  "status": "ok",
  "action": "ping",
  "ssl": true,
  "tls_version": "TLSv1.3",
  "cipher": "TLS_AES_256_GCM_SHA384",
  "pg_version": "PostgreSQL 16.x on x86_64-pc-linux-gnu ..."
}
```

#### Test Event 2: Query (run a SELECT)

```json
{
  "action": "query",
  "sql": "SELECT current_database(), current_user, now()"
}
```

Expected response:
```json
{
  "status": "ok",
  "action": "query",
  "columns": ["current_database", "current_user", "now"],
  "rows": [["yourdb", "youruser", "2026-03-18T10:00:00"]]
}
```

#### How to Run

1. Lambda Console → your function → **Test** tab
2. Click **Create new test event** → paste either JSON above → save
3. Click **Test** → check **Execution results** panel for response and logs

> Start with `ping` — it confirms connectivity, SSL is active, and which TLS version and cipher are in use, all in one call.

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
