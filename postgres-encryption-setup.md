# PostgreSQL Security Hardening — EC2 + Docker + Lambda

**Environment:** PostgreSQL running as a Docker container on an EC2 instance  
**Client:** Python Lambda function  
**Project:** InsightGen  

---

## Table of Contents

1. [Overview](#1-overview)
2. [Encryption In Transit — TLS/SSL](#2-encryption-in-transit--tlsssl)
   - [Step 1: Generate Self-Signed Certificates on EC2](#step-1--generate-self-signed-certificates-on-ec2)
   - [Step 2: Fix Certificate Permissions](#step-2--fix-certificate-permissions)
   - [Step 3: Deploy Certificates into PostgreSQL Container](#step-3--deploy-certificates-into-postgresql-container)
   - [Step 4: Enable SSL in postgresql.conf](#step-4--enable-ssl-in-postgresqlconf)
   - [Step 5: Configure pg_hba.conf](#step-5--configure-pg_hbaconf)
   - [Step 6: Reload PostgreSQL Configuration](#step-6--reload-postgresql-configuration)
   - [Step 7: Verify SSL is Active](#step-7--verify-ssl-is-active)
3. [Encryption At Rest — EBS Volume](#3-encryption-at-rest--ebs-volume)
4. [Secrets Management — AWS Parameter Store](#4-secrets-management--aws-parameter-store)
   - [Step 1: Create Dedicated KMS Key](#step-1--create-a-dedicated-kms-key)
   - [Step 2: Create KMS Key Policy](#step-2--create-kms-key-policy)
   - [Step 3: Store Parameters as SecureString](#step-3--store-parameters-as-securestring)
   - [Step 4: Lambda IAM Role Policy](#step-4--lambda-iam-role-policy)
   - [Step 5: Lambda Code](#step-5--lambda-code)
   - [Step 6: Verify Access Control](#step-6--verify-access-control)
5. [Validate End-to-End Encryption](#5-validate-end-to-end-encryption)
6. [Architecture Summary](#6-architecture-summary)
7. [Pending Tasks](#7-pending-tasks)

---

## 1. Overview

| Requirement | Approach | Status |
|---|---|---|
| Encryption in transit | TLS/SSL on PostgreSQL container | ✅ Done |
| Encryption at rest | EBS volume encryption via KMS | 🔄 Pending |
| Secrets management | AWS SSM Parameter Store (SecureString + KMS) | ✅ Done |
| Connection validation | `sslmode=require` — encrypted, no cert validation | ✅ Done |
| Cert-based validation | `sslmode=verify-ca` — upgrade pending cert mismatch fix | ⏳ Pending |

---

## 2. Encryption In Transit — TLS/SSL

### Goal

- Encrypt all traffic between Lambda and PostgreSQL
- Keep existing username/password authentication
- Both plain and SSL connections supported during transition period

> **Note:** PostgreSQL uses TLS under the hood. "SSL" and "TLS" are used interchangeably in PostgreSQL documentation. This is unrelated to SSH.

---

### Step 1 — Generate Self-Signed Certificates on EC2

SSH into the EC2 instance and run:

```bash
mkdir -p ~/postgres-ssl && cd ~/postgres-ssl

# Generate CA private key
openssl genrsa -out ca.key 4096

# Generate CA self-signed certificate (valid 10 years)
openssl req -new -x509 -days 3650 -key ca.key -out ca.crt \
  -subj "/CN=postgres-ca"

# Generate server private key
openssl genrsa -out server.key 4096

# Generate server Certificate Signing Request
# Replace <EC2_PRIVATE_IP> with your actual EC2 private IP
openssl req -new -key server.key -out server.csr \
  -subj "/CN=<EC2_PRIVATE_IP>"

# Sign the server cert with the CA
openssl x509 -req -days 3650 \
  -in server.csr \
  -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out server.crt

# Verify all files are present
ls -la ~/postgres-ssl/
# Expected: ca.crt  ca.key  ca.srl  server.crt  server.csr  server.key
```

---

### Step 2 — Fix Certificate Permissions

PostgreSQL will refuse to start if the private key permissions are too open.

```bash
# PostgreSQL container runs as UID 999
chmod 600 server.key          # private key — owner read only
chmod 644 server.crt ca.crt   # certificates — world readable is fine
chown 999:999 server.key server.crt ca.crt
```

---

### Step 3 — Deploy Certificates into PostgreSQL Container

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

### Step 4 — Enable SSL in postgresql.conf

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

### Step 5 — Configure pg_hba.conf

```bash
docker exec -it <container_name> bash

# View current rules before editing
cat /var/lib/postgresql/data/pg_hba.conf

# Append both SSL and plain connection rules
cat >> /var/lib/postgresql/data/pg_hba.conf << 'EOF'

# Allow SSL (encrypted) connections
hostssl   all   all   0.0.0.0/0   scram-sha-256

# Allow plain (unencrypted) connections — preserved during transition
host      all   all   0.0.0.0/0   scram-sha-256
EOF

exit
```

> **Note:** Once `sslmode=verify-ca` is confirmed working, remove the `host` line to enforce SSL-only connections.

**pg_hba.conf Rule Reference:**

| Rule | Meaning |
|---|---|
| `hostssl` | SSL/TLS connections only |
| `host` | Both plain and SSL connections |
| `hostnossl` | Plain connections only |

---

### Step 6 — Reload PostgreSQL Configuration

```bash
# Reload config — no downtime, no dropped connections
docker exec -it <container_name> psql -U postgres -c "SELECT pg_reload_conf();"

# Confirm SSL is active
docker exec -it <container_name> psql -U postgres -c "SHOW ssl;"
# Expected:
#  ssl
# -----
#  on
```

---

### Step 7 — Verify SSL is Active

```bash
# Test SSL connection from EC2
psql "host=<EC2_PRIVATE_IP> port=5432 dbname=yourdb \
      user=youruser sslmode=require" --password

# Check SSL status inside an active session
psql -U postgres -c "
  SELECT ssl, version, cipher, bits
  FROM pg_stat_ssl
  WHERE pid = pg_backend_pid();"
```

Expected output:

```
 ssl | version |         cipher         | bits
-----+---------+------------------------+------
 t   | TLSv1.3 | TLS_AES_256_GCM_SHA384 |  256
```

---

## 3. Encryption At Rest — EBS Volume

### Current State

EBS volume attached to EC2 is **unencrypted**. Encryption cannot be enabled in-place — migration via snapshot is required with planned downtime.

### What EBS Encryption Covers

- All PostgreSQL data files on disk (AES-256 via KMS)
- WAL (Write-Ahead Log) files
- Temp files and buffers flushed to disk
- EBS snapshots encrypted automatically
- Fully transparent to PostgreSQL and Docker — zero application changes needed

### Migration Steps

**Estimated downtime: 5–15 minutes**

```bash
# Step 1 — Stop EC2 instance
aws ec2 stop-instances --instance-ids i-xxxxxxxxxxxxxxxxx

# Step 2 — Get current volume ID
aws ec2 describe-instances --instance-ids i-xxxxxxxxxxxxxxxxx \
  --query "Reservations[].Instances[].BlockDeviceMappings[].Ebs.VolumeId"

# Step 3 — Snapshot current unencrypted volume
aws ec2 create-snapshot \
  --volume-id vol-xxxxxxxxxxxxxxxxx \
  --description "pre-encryption-backup-$(date +%Y%m%d)"

# Wait until snapshot state = "completed"
aws ec2 describe-snapshots --snapshot-ids snap-xxxxxxxxxxxxxxxxx \
  --query "Snapshots[].State"

# Step 4 — Copy snapshot with encryption enabled
aws ec2 copy-snapshot \
  --source-region us-east-1 \
  --source-snapshot-id snap-xxxxxxxxxxxxxxxxx \
  --encrypted \
  --kms-key-id alias/aws/ebs \
  --description "insightgen-postgres-encrypted"

# Wait until encrypted snapshot state = "completed"
aws ec2 describe-snapshots --snapshot-ids snap-yyyyyyyyyyyyyyyyy \
  --query "Snapshots[].State"

# Step 5 — Create new encrypted volume from snapshot
aws ec2 create-volume \
  --snapshot-id snap-yyyyyyyyyyyyyyyyy \
  --availability-zone us-east-1a \
  --volume-type gp3 \
  --encrypted

# Step 6 — Swap volumes
aws ec2 detach-volume --volume-id vol-xxxxxxxxxxxxxxxxx
aws ec2 attach-volume \
  --volume-id vol-yyyyyyyyyyyyyyyyy \
  --instance-id i-xxxxxxxxxxxxxxxxx \
  --device /dev/xvda

# Step 7 — Start EC2 instance
aws ec2 start-instances --instance-ids i-xxxxxxxxxxxxxxxxx

# Step 8 — Verify encryption
aws ec2 describe-volumes --volume-ids vol-yyyyyyyyyyyyyyyyy \
  --query "Volumes[].Encrypted"
# Must return: true
```

> **Important:** Retain the old unencrypted volume for at least one week before deleting — use as a rollback safety net.

---

## 4. Secrets Management — AWS Parameter Store

### Why Parameter Store

| | Secrets Manager | Parameter Store |
|---|---|---|
| Cost | ~$0.40/secret/month | **Free** (Standard tier) |
| Automatic rotation | ✅ Built-in | ❌ Manual |
| Encryption | ✅ KMS | ✅ KMS (SecureString) |
| Best for | Auto-rotating secrets | Config + credentials |

Parameter Store is used here — free, KMS-encrypted, sufficient without auto-rotation. All parameters stored as `SecureString` per security policy — including host, port, and dbname.

---

### Step 1 — Create a Dedicated KMS Key

A dedicated key gives full control over who can encrypt and decrypt. Do not use the default AWS managed key.

```bash
# Create the key
aws kms create-key \
  --description "InsightGen PostgreSQL credentials encryption key"

# Note the KeyId returned: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

# Create a friendly alias
aws kms create-alias \
  --alias-name alias/insightgen-postgres-credentials \
  --target-key-id <KeyId>
```

---

### Step 2 — Create KMS Key Policy

Controls who can **administer** the key and who can **decrypt** using it. These are intentionally separated — KMS admins cannot decrypt credentials.

Save as `kms-policy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "KeyAdministration",
      "Effect": "Allow",
      "Principal": {
        "AWS": [
          "arn:aws:iam::<account-id>:user/admin-user-1",
          "arn:aws:iam::<account-id>:user/admin-user-2"
        ]
      },
      "Action": [
        "kms:Create*",
        "kms:Describe*",
        "kms:Enable*",
        "kms:List*",
        "kms:Put*",
        "kms:Update*",
        "kms:Revoke*",
        "kms:Disable*",
        "kms:Delete*",
        "kms:ScheduleKeyDeletion",
        "kms:CancelKeyDeletion"
      ],
      "Resource": "*"
    },
    {
      "Sid": "LambdaDecryptAccess",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::<account-id>:role/your-lambda-execution-role"
      },
      "Action": [
        "kms:Decrypt",
        "kms:DescribeKey"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AuthorizedUsersDecryptAccess",
      "Effect": "Allow",
      "Principal": {
        "AWS": [
          "arn:aws:iam::<account-id>:user/dev-user-1",
          "arn:aws:iam::<account-id>:user/dev-user-2"
        ]
      },
      "Action": [
        "kms:Decrypt",
        "kms:DescribeKey"
      ],
      "Resource": "*"
    }
  ]
}
```

Apply the policy:

```bash
aws kms put-key-policy \
  --key-id alias/insightgen-postgres-credentials \
  --policy-name default \
  --policy file://kms-policy.json
```

**Access control summary:**

| Who | Can Read SSM? | Can Decrypt? |
|---|---|---|
| Lambda execution role | ✅ Yes | ✅ Yes |
| Authorized IAM users | ✅ Yes | ✅ Yes |
| KMS admins | ✅ Yes | ❌ No |
| Everyone else | ❌ No | ❌ No |

---

### Step 3 — Store Parameters as SecureString

All values stored under `/insightgen/postgres/` and encrypted using the dedicated KMS key:

```bash
aws ssm put-parameter \
  --name "/insightgen/postgres/host" \
  --value "172.x.x.x" \
  --type SecureString \
  --key-id alias/insightgen-postgres-credentials

aws ssm put-parameter \
  --name "/insightgen/postgres/port" \
  --value "5432" \
  --type SecureString \
  --key-id alias/insightgen-postgres-credentials

aws ssm put-parameter \
  --name "/insightgen/postgres/dbname" \
  --value "yourdb" \
  --type SecureString \
  --key-id alias/insightgen-postgres-credentials

aws ssm put-parameter \
  --name "/insightgen/postgres/username" \
  --value "youruser" \
  --type SecureString \
  --key-id alias/insightgen-postgres-credentials

aws ssm put-parameter \
  --name "/insightgen/postgres/password" \
  --value "yourpassword" \
  --type SecureString \
  --key-id alias/insightgen-postgres-credentials
```

---

### Step 4 — Lambda IAM Role Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "SSMReadAccess",
      "Effect": "Allow",
      "Action": [
        "ssm:GetParameter",
        "ssm:GetParameters"
      ],
      "Resource": "arn:aws:ssm:us-east-1:<account-id>:parameter/insightgen/postgres/*"
    },
    {
      "Sid": "KMSDecryptAccess",
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt",
        "kms:DescribeKey"
      ],
      "Resource": "arn:aws:kms:us-east-1:<account-id>:key/<KeyId>"
    }
  ]
}
```

---

### Step 5 — Lambda Code

Credentials are fetched from Parameter Store at cold start. No hardcoded values. `sslmode=require` encrypts all traffic without requiring a CA certificate file.

```python
import boto3
import psycopg2
import logging

logger = logging.getLogger()
logger.setLevel(logging.INFO)

SSM_REGION = "us-east-1"

def get_db_config():
    """Fetch all DB config from SSM Parameter Store at cold start."""
    logger.info("Fetching DB config from SSM Parameter Store")
    ssm    = boto3.client("ssm", region_name=SSM_REGION)
    params = ssm.get_parameters(
        Names=[
            "/insightgen/postgres/host",
            "/insightgen/postgres/port",
            "/insightgen/postgres/dbname",
            "/insightgen/postgres/username",
            "/insightgen/postgres/password"
        ],
        WithDecryption=True
    )
    config = {p["Name"].split("/")[-1]: p["Value"] for p in params["Parameters"]}
    logger.info("DB config loaded: host=%s port=%s dbname=%s username=%s",
        config["host"], config["port"], config["dbname"], config["username"]
        # password intentionally not logged
    )
    return config

# Runs once at cold start — reused across warm invocations
DB_CONFIG = get_db_config()

def get_connection():
    return psycopg2.connect(
        host=DB_CONFIG["host"],
        port=int(DB_CONFIG["port"]),
        dbname=DB_CONFIG["dbname"],
        user=DB_CONFIG["username"],
        password=DB_CONFIG["password"],
        sslmode="require",      # traffic encrypted, no CA cert validation
        connect_timeout=10
    )

def lambda_handler(event, context):
    """
    Lambda entrypoint.

    Test actions:
      ping   — verifies connectivity and confirms SSL is active
      query  — runs a SELECT query (pass 'sql' key in event)
    """
    action = event.get("action", "ping")

    try:
        if action == "ping":
            conn   = get_connection()
            cursor = conn.cursor()
            cursor.execute("""
                SELECT ssl, version, cipher, bits
                FROM pg_stat_ssl
                WHERE pid = pg_backend_pid()
            """)
            ssl_info = cursor.fetchone()
            cursor.execute("SELECT version()")
            pg_version = cursor.fetchone()[0]
            cursor.close()
            conn.close()
            return {
                "status":      "ok",
                "action":      "ping",
                "ssl":         ssl_info[0],
                "tls_version": ssl_info[1],
                "cipher":      ssl_info[2],
                "bits":        ssl_info[3],
                "pg_version":  pg_version
            }

        elif action == "query":
            sql = event.get("sql")
            if not sql:
                return {"status": "error", "message": "action=query requires a 'sql' field"}
            if not sql.strip().upper().startswith("SELECT"):
                return {"status": "error", "message": "Only SELECT queries allowed"}
            conn   = get_connection()
            cursor = conn.cursor()
            cursor.execute(sql)
            rows = cursor.fetchall()
            cols = [desc[0] for desc in cursor.description]
            cursor.close()
            conn.close()
            return {"status": "ok", "action": "query", "columns": cols, "rows": rows}

        else:
            return {"status": "error", "message": f"Unknown action '{action}'. Use: ping, query"}

    except psycopg2.OperationalError as e:
        logger.error("DB connection error: %s", str(e))
        return {"status": "error", "action": action, "message": str(e)}
    except Exception as e:
        logger.error("Unexpected error: %s", str(e))
        return {"status": "error", "action": action, "message": str(e)}
```

**Test events for Lambda Console — Test tab:**

Ping:
```json
{ "action": "ping" }
```

Expected ping response:
```json
{
  "status": "ok",
  "action": "ping",
  "ssl": true,
  "tls_version": "TLSv1.3",
  "cipher": "TLS_AES_256_GCM_SHA384",
  "bits": 256,
  "pg_version": "PostgreSQL 16.x ..."
}
```

Query:
```json
{
  "action": "query",
  "sql": "SELECT current_database(), current_user, now()"
}
```

---

### Step 6 — Verify Access Control

```bash
# Confirm Lambda role can decrypt
aws ssm get-parameter \
  --name "/insightgen/postgres/password" \
  --with-decryption \
  --profile lambda-role-profile
# Should return the decrypted value

# Confirm unauthorized users cannot decrypt
aws ssm get-parameter \
  --name "/insightgen/postgres/password" \
  --with-decryption \
  --profile unauthorized-user
# Should return: AccessDeniedException
```

**To rotate credentials — no Lambda redeploy needed:**

```bash
aws ssm put-parameter \
  --name "/insightgen/postgres/password" \
  --value "newpassword" \
  --type SecureString \
  --key-id alias/insightgen-postgres-credentials \
  --overwrite
```

---

## 5. Validate End-to-End Encryption

### Confirmed Result from Lambda Ping Test

```json
{
  "ssl": true,
  "tls_version": "TLSv1.3",
  "cipher": "TLS_AES_256_GCM_SHA384",
  "bits": 256
}
```

- `ssl: true` — connection is encrypted
- `TLSv1.3` — latest and most secure TLS version
- `TLS_AES_256_GCM_SHA384` — AES-256 bit encryption, same standard used by banks and major SaaS products

### Verify Active Connections from EC2

```bash
docker exec -it <container_name> psql -U postgres -c "
  SELECT pid, ssl, version, cipher, client_addr
  FROM pg_stat_ssl;"
# ssl=t on all Lambda connections confirms encrypted traffic
```

### sslmode Reference

| sslmode | Encrypts? | Validates CA? | Status |
|---|---|---|---|
| `disable` | ❌ | ❌ | Not used |
| `require` | ✅ | ❌ | **Current — working** |
| `verify-ca` | ✅ | ✅ | Pending — cert mismatch fix required |
| `verify-full` | ✅ | ✅ | Future consideration |

---

## 6. Architecture Summary

```
Lambda (Python / psycopg2)
  ├─ DB credentials  → SSM Parameter Store (/insightgen/postgres/*)
  │                    SecureString encrypted via dedicated KMS key
  │                    (alias/insightgen-postgres-credentials)
  │
  └─ DB connection   → sslmode=require (TLSv1.3 / AES-256-GCM-SHA384)
  │
  ▼
EC2 Security Group
  └─ Port 5432 — Lambda Security Group only
  │
  ▼
Docker → PostgreSQL Container
  ├─ ssl=on (postgresql.conf)
  ├─ TLSv1.2 minimum enforced
  ├─ hostssl + host rules (pg_hba.conf)
  └─ Certificates at /var/lib/postgresql/
  │
  ▼
EBS Volume
  └─ ⚠️ Encryption pending — migration scheduled
```

---

## 7. Pending Tasks

**SEC-005 — Enable KMS Encryption on EC2 EBS Data Volume**
- Schedule maintenance window — 5 to 15 minutes downtime required
- Follow migration steps in Section 3
- Verify `Encrypted=true` on new volume after swap
- Retain old volume for one week before deletion

**SEC-006 — Enforce Certificate-Based TLS Validation**
- Identify which certificate PostgreSQL is currently serving using `SHOW ssl_cert_file`
- Confirm PostgreSQL is using the correct certificates from `~/postgres-ssl/`
- Upgrade Lambda `sslmode` from `require` to `verify-ca`
- Revalidate using Lambda ping test — confirm `ssl=true` with cert validation passing

**SEC-007 — Remove Plain Connection Support**
- Once `verify-ca` is confirmed working, remove the `host` rule from `pg_hba.conf`
- Retain `hostssl` only to enforce SSL-only connections

**SEC-008 — Security Group Hardening**
- Confirm EC2 inbound port 5432 is restricted to Lambda security group ID only
- Remove any broad CIDR-based inbound rules

**SEC-009 — Enable Audit Logging**
- Enable VPC Flow Logs on Lambda and EC2 subnets
- Enable CloudTrail for API and connection activity audit trail

**SEC-010 — Certificate Expiry Monitoring**
- Document certificate expiry dates (current certs valid for 3650 days)
- Set calendar reminder well before expiry
- Document certificate regeneration and redeployment runbook

---

*Project: InsightGen — PostgreSQL on EC2 (Docker) + Python Lambda*
