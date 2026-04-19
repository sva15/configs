# Connection Manager Service — Implementation Guide

## Architecture Recap

```
Lambda (domain_id header)
    │
    ▼
Nginx (EC2 host)  ──/db-proxy/──▶  Connection Manager Container (port 8001)
                                         │
                              ┌──────────┴──────────┐
                              ▼                     ▼
                     AWS Parameter Store      Common DB (PostgreSQL)
                     (common DB password)     (domain_credentials table)
                                                    │
                                          KMS Decrypt password_encrypted
                                                    │
                                          ┌─────────┴─────────┐
                                          ▼                   ▼
                                    Domain DB A         Domain DB B
```

**Flow for every new domain_id:**
1. Lambda sends `POST /v1/credentials` with `domain_id`
2. Connection Manager checks its in-memory cache
3. If not cached → queries Common DB → KMS-decrypts the password → caches result
4. Returns decrypted credentials to Lambda
5. Lambda opens its own connection to the Domain DB

---

## Prerequisites Checklist

Before starting, confirm the following on your EC2 instance:

- [ ] Docker installed and running
- [ ] Nginx installed with write access to `/etc/nginx/conf.d/default.conf`
- [ ] AWS CLI configured (or EC2 instance has an IAM role attached)
- [ ] Python 3.12 available locally for testing
- [ ] PostgreSQL container already running (or you will start one below)
- [ ] Your EC2 instance is in a VPC and your Lambda will be in the same VPC

---

## Phase 1: AWS Infrastructure Setup

### 1.1 Existing KMS Key

The KMS Customer Managed Key already exists. Verify it is accessible before proceeding:

```bash
aws kms describe-key \
  --key-id alias/HCL-USER-KEY-insightgen-postgres-credentials \
  --region us-east-1 \
  --query "KeyMetadata.{KeyId:KeyId, Enabled:Enabled, Description:Description}"
```

Expected output confirms the key is `"Enabled": true`. Note the `KeyId` value — you will need it when running `kms:Encrypt` commands in Phase 2.

---

### 1.2 Update the KMS Key Policy

The key policy must be updated to add the EC2 instance profile so the Connection Manager container can decrypt both the SSM parameter (common DB password) and the KMS ciphertexts stored inside the `domain_credentials` table.

> **How to add more principals in future:** The policy has two clearly marked extension points — see the comments inside the JSON below. Adding a new service role follows the same pattern as the EC2 statement. Adding a new user session follows the same pattern as the `AllowAdminUserSessionFullKMSAccess` statement.

Apply the updated policy:

```bash
aws kms put-key-policy \
  --key-id alias/HCL-USER-KEY-insightgen-postgres-credentials \
  --policy-name default \
  --region us-east-1 \
  --policy '{
  "Version": "2012-10-17",
  "Id": "key-consolepolicy-restricted-shared-role",
  "Statement": [
    {
      "Sid": "EnableRootAccessAndPreventPermissionDelegation",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::<ACCOUNT_ID>:root"
      },
      "Action": "kms:*",
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:PrincipalType": "Account"
        }
      }
    },
    {
      "Sid": "AllowAdminUserSessionFullKMSAccess",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::<ACCOUNT_ID>:role/aws-reserved/sso.amazonaws.com/AWSReservedSSO_LZ-Account-Users-AWS_2cc08ead42206ada"
      },
      "Action": "kms:*",
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:userid": "AROAS6J7QKS2T7Q6KMSCW:john.singa@company.com"
        }
      }
    },
    {
      "Sid": "AllowDescribeKeyForApprovedUserSessions",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::<ACCOUNT_ID>:role/aws-reserved/sso.amazonaws.com/AWSReservedSSO_LZ-Account-Users-AWS_2cc08ead42206ada"
      },
      "Action": "kms:DescribeKey",
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:userid": "AROAS6J7QKS2T7Q6KMSCW:john.singa@company.com"
        }
      }
    },
    {
      "Sid": "AllowDecryptViaSSMForApprovedUserSessionsOnAllowedPath",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::<ACCOUNT_ID>:role/aws-reserved/sso.amazonaws.com/AWSReservedSSO_LZ-Account-Users-AWS_2cc08ead42206ada"
      },
      "Action": "kms:Decrypt",
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:userid": "AROAS6J7QKS2T7Q6KMSCW:john.singa@company.com"
        },
        "StringLike": {
          "kms:ViaService": "ssm.*.amazonaws.com",
          "kms:EncryptionContext:PARAMETER_ARN": "arn:aws:ssm:us-east-1:<ACCOUNT_ID>:parameter/insightgen/postgres/*"
        }
      }
    },
    {
      "Sid": "DenyDecryptForAllOtherSessionsOfSharedSSORole",
      "Effect": "Deny",
      "Principal": {
        "AWS": "*"
      },
      "Action": "kms:Decrypt",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:userid": "AROAS6J7QKS2T7Q6KMSCW:john.singa@company.com"
        },
        "ArnEquals": {
          "aws:PrincipalArn": "arn:aws:iam::<ACCOUNT_ID>:role/aws-reserved/sso.amazonaws.com/AWSReservedSSO_LZ-Account-Users-AWS_2cc08ead42206ada"
        }
      }
    },
    {
      "Sid": "AllowLambdaRoleDecryptViaSSMForInsightGenPostgres",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::<ACCOUNT_ID>:role/HCL-User-Role-InsightGen-Lambda"
      },
      "Action": "kms:Decrypt",
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "kms:ViaService": "ssm.us-east-1.amazonaws.com"
        },
        "StringLike": {
          "kms:EncryptionContext:PARAMETER_ARN": "arn:aws:ssm:us-east-1:<ACCOUNT_ID>:parameter/insightgen/postgres/*"
        }
      }
    },
    {
      "Sid": "AllowEC2RoleDecryptViaSSMForCommonDBPassword",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::<ACCOUNT_ID>:instance-profile/HCL-User-Role-InsightGen-EC2"
      },
      "Action": "kms:Decrypt",
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "kms:ViaService": "ssm.us-east-1.amazonaws.com"
        },
        "StringLike": {
          "kms:EncryptionContext:PARAMETER_ARN": "arn:aws:ssm:us-east-1:<ACCOUNT_ID>:parameter/insightgen/postgres/*"
        }
      }
    },
    {
      "Sid": "AllowEC2RoleDirectDecryptForDBTableCiphertexts",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::<ACCOUNT_ID>:instance-profile/HCL-User-Role-InsightGen-EC2"
      },
      "Action": ["kms:Decrypt", "kms:DescribeKey"],
      "Resource": "*"
    }
  ]
}'
```

> **Replace `<ACCOUNT_ID>`** with your AWS account ID in every statement above.

**Why the EC2 role needs two separate statements:**

The connection manager performs two distinct KMS operations that require different statement types:

| Operation | When | KMS call type | Statement that covers it |
|---|---|---|---|
| Fetch common DB password from SSM | Container startup (once) | Decrypt via SSM service | `AllowEC2RoleDecryptViaSSMForCommonDBPassword` |
| Decrypt domain passwords stored in `domain_credentials` table | First request per domain_id | Direct `kms:Decrypt` boto3 call — not via SSM | `AllowEC2RoleDirectDecryptForDBTableCiphertexts` |

The first operation goes through SSM (boto3 → SSM → KMS internally) so the `kms:ViaService` condition applies. The second operation is a direct boto3 `kms.decrypt()` call on the raw ciphertext blob, so there is no `kms:ViaService` context — a ViaService-only statement would silently fail for it.

**How to add more principals who can decrypt:**

- **New service role (e.g. another Lambda or container):** Add a new statement following the same pattern as `AllowEC2RoleDecryptViaSSMForCommonDBPassword` with the new role ARN. If it also needs direct KMS decrypt (not via SSM), add a second statement following `AllowEC2RoleDirectDecryptForDBTableCiphertexts`.
- **New user session (SSO user):** Add a new statement following the same pattern as `AllowAdminUserSessionFullKMSAccess`, scoped to the specific `aws:userid` value of that person's SSO session.

---

### 1.3 Add SSM Permission to the Existing EC2 Instance Profile

The EC2 instance profile `HCL-User-Role-InsightGen-EC2` already exists. You only need to attach an inline policy granting it permission to read the specific SSM parameter the connection manager will use:

```bash
aws iam put-role-policy \
  --role-name HCL-User-Role-InsightGen-EC2 \
  --policy-name ConnectionManagerSSMAccess \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": ["ssm:GetParameter"],
        "Resource": "arn:aws:ssm:us-east-1:<ACCOUNT_ID>:parameter/insightgen/postgres/common-db/password"
      }
    ]
  }'
```

> The KMS decrypt permission for this role is already handled in the key policy updated in step 1.2 — no separate IAM KMS statement is needed here. AWS evaluates both the key policy and the IAM policy; since the key policy explicitly allows the role, the IAM policy only needs to cover SSM access.

---

### 1.4 Existing Lambda Role

The Lambda role `HCL-User-Role-InsightGen-Lambda` already exists and its KMS decrypt permission is already present in the key policy. No changes are needed to this role.

Confirm it has VPC execution permissions (needed for Lambda to reach the EC2 private IP):

```bash
aws iam list-attached-role-policies \
  --role-name HCL-User-Role-InsightGen-Lambda \
  --query "AttachedPolicies[*].PolicyName"
```

If `AWSLambdaVPCAccessExecutionRole` is not listed, attach it:

```bash
aws iam attach-role-policy \
  --role-name HCL-User-Role-InsightGen-Lambda \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole
```

---

## Phase 2: PostgreSQL Database Setup (Manual)

All three databases live inside the **same PostgreSQL container**. You will exec into it and run `psql` commands.

### 2.1 Start PostgreSQL (skip if already running)

```bash
# Create a Docker network so containers can talk by name
docker network create app-network

# Start PostgreSQL — adjust password and port as needed
docker run -d \
  --name postgres \
  --network app-network \
  -p 5432:5432 \
  -e POSTGRES_USER=ifrs_user \
  -e POSTGRES_PASSWORD=postgres_root_pass \
  -v postgres_data:/var/lib/postgresql/data \
  postgres:16
```

### 2.2 Create Databases, Users, and Schema

```bash
# Exec into the PostgreSQL container
docker exec -it postgres psql -U ifrs_user
```

Run the following SQL inside `psql`:

```sql
-- ─────────────────────────────────────────────
-- Step 1: Create users with known passwords
-- (these are the passwords you will encrypt with KMS below)
-- ─────────────────────────────────────────────
CREATE USER common_user   WITH PASSWORD 'CommonDbPass123!';
CREATE USER domain_a_user WITH PASSWORD 'DomainAPass456!';
CREATE USER domain_b_user WITH PASSWORD 'DomainBPass789!';

-- ─────────────────────────────────────────────
-- Step 2: Create databases
-- ─────────────────────────────────────────────
CREATE DATABASE common_db   OWNER common_user;
CREATE DATABASE domain_db_a OWNER domain_a_user;
CREATE DATABASE domain_db_b OWNER domain_b_user;
```

Exit psql (`\q`) and reconnect to set up each database:

```bash
# Set up common_db schema
docker exec -it postgres psql -U ifrs_user -d common_db
```

```sql
-- Run as ifrs_user superuser inside common_db

CREATE TABLE domain_credentials (
    id                 SERIAL PRIMARY KEY,
    domain_id          VARCHAR(100) UNIQUE NOT NULL,
    host               VARCHAR(255) NOT NULL,
    port               INTEGER NOT NULL DEFAULT 5432,
    db_name            VARCHAR(100) NOT NULL,
    db_user            VARCHAR(100) NOT NULL,
    password_encrypted TEXT NOT NULL,       -- stores KMS ciphertext (base64)
    created_at         TIMESTAMPTZ DEFAULT NOW(),
    updated_at         TIMESTAMPTZ DEFAULT NOW()
);

GRANT SELECT ON domain_credentials TO common_user;
GRANT USAGE, SELECT ON SEQUENCE domain_credentials_id_seq TO common_user;
```

```bash
# Set up domain_db_a
docker exec -it postgres psql -U ifrs_user -d domain_db_a
```

```sql
CREATE TABLE tenant_data (
    id         SERIAL PRIMARY KEY,
    domain_id  VARCHAR(100) NOT NULL,
    data_key   VARCHAR(255) NOT NULL,
    data_value TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

GRANT ALL PRIVILEGES ON TABLE tenant_data TO domain_a_user;
GRANT USAGE, SELECT ON SEQUENCE tenant_data_id_seq TO domain_a_user;
```

```bash
# Set up domain_db_b
docker exec -it postgres psql -U ifrs_user -d domain_db_b
```

```sql
CREATE TABLE tenant_data (
    id         SERIAL PRIMARY KEY,
    domain_id  VARCHAR(100) NOT NULL,
    data_key   VARCHAR(255) NOT NULL,
    data_value TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

GRANT ALL PRIVILEGES ON TABLE tenant_data TO domain_b_user;
GRANT USAGE, SELECT ON SEQUENCE tenant_data_id_seq TO domain_b_user;
```

---

### 2.3 Encrypt Domain DB Passwords Using KMS

Run these commands from **your local machine or bastion** (you need AWS credentials with `kms:Encrypt` permission — your SSO session as `john.singa@company.com` has this via the `AllowAdminUserSessionFullKMSAccess` statement):

```bash
# Encrypt domain_a password
echo -n "DomainAPass456!" > /tmp/pass_a.txt
CIPHER_A=$(aws kms encrypt \
  --key-id alias/HCL-USER-KEY-insightgen-postgres-credentials \
  --plaintext fileb:///tmp/pass_a.txt \
  --query CiphertextBlob \
  --output text \
  --region us-east-1)
echo "Ciphertext A: $CIPHER_A"
rm /tmp/pass_a.txt

# Encrypt domain_b password
echo -n "DomainBPass789!" > /tmp/pass_b.txt
CIPHER_B=$(aws kms encrypt \
  --key-id alias/HCL-USER-KEY-insightgen-postgres-credentials \
  --plaintext fileb:///tmp/pass_b.txt \
  --query CiphertextBlob \
  --output text \
  --region us-east-1)
echo "Ciphertext B: $CIPHER_B"
rm /tmp/pass_b.txt
```

> **Important:** Copy the ciphertext values (`$CIPHER_A` and `$CIPHER_B`). You will paste them into the SQL below.
> Delete the temp files immediately after — never let the plaintext touch disk for longer than needed.

---

### 2.4 Insert Encrypted Credentials into Common DB

The `host` field is the EC2 private IP `10.132.191.157` — this is the address Lambda (inside the VPC) will use to connect to the domain databases.

```bash
docker exec -it postgres psql -U ifrs_user -d common_db
```

```sql
-- Replace the ciphertext values with the actual output from the KMS encrypt commands above.

INSERT INTO domain_credentials (domain_id, host, port, db_name, db_user, password_encrypted)
VALUES
  (
    'tenant_a',
    '10.132.191.157',
    5432,
    'domain_db_a',
    'domain_a_user',
    'AQICAHxxx...PASTE_CIPHER_A_HERE...xxx'
  ),
  (
    'tenant_b',
    '10.132.191.157',
    5432,
    'domain_db_b',
    'domain_b_user',
    'AQICAHyyy...PASTE_CIPHER_B_HERE...yyy'
  );

-- Verify
SELECT domain_id, host, db_name, db_user, LEFT(password_encrypted, 20) || '...' AS pwd_preview
FROM domain_credentials;
```

---

### 2.5 Insert Sample Data into Domain Tables

Insert test records so the Lambda function has real data to return when it connects to each domain DB.

```bash
docker exec -it postgres_db psql -U ifrs_user -d ifrs_dev
```

```sql
\c domain_db_a

INSERT INTO tenant_data (domain_id, data_key, data_value) VALUES
  ('tenant_a', 'company_name',   'Acme Corp'),
  ('tenant_a', 'fiscal_year',    '2024'),
  ('tenant_a', 'currency',       'USD'),
  ('tenant_a', 'reporting_date', '2024-12-31');

-- Verify
SELECT * FROM tenant_data;

\c domain_db_b

INSERT INTO tenant_data (domain_id, data_key, data_value) VALUES
  ('tenant_b', 'company_name',   'Globex Ltd'),
  ('tenant_b', 'fiscal_year',    '2024'),
  ('tenant_b', 'currency',       'EUR'),
  ('tenant_b', 'reporting_date', '2024-12-31');

-- Verify
SELECT * FROM tenant_data;

\q
```

---

## Phase 3: Store Common DB Password in SSM Parameter Store

### Existing parameters (do not modify)

The following parameters already exist and are used by other services. Leave them untouched:

| Parameter path | Value | Used by |
|---|---|---|
| `/insightgen/postgres/host` | `10.132.191.157` | Other services |
| `/insightgen/postgres/port` | `5432` | Other services |
| `/insightgen/postgres/username` | `ifrs_user` | Other services |
| `/insightgen/postgres/password` | ifrs_user master password | Other services — **do not use for connection manager** |

### Create a new dedicated parameter for the connection manager

Create a new parameter specifically for `common_user`'s password under the same path hierarchy. This falls within the `/insightgen/postgres/*` wildcard that the existing KMS policy already covers — no policy changes are needed:

```bash
aws ssm put-parameter \
  --name "/insightgen/postgres/common-db/password" \
  --value "CommonDbPass123!" \
  --type SecureString \
  --key-id alias/HCL-USER-KEY-insightgen-postgres-credentials \
  --description "Password for common_user on common_db — used by Connection Manager service only" \
  --region us-east-1

# Verify it was stored and can be decrypted
aws ssm get-parameter \
  --name "/insightgen/postgres/common-db/password" \
  --with-decryption \
  --region us-east-1
```

> After confirming SSM holds the value correctly, the plain text password (`CommonDbPass123!`) should no longer exist anywhere — not in shell history, scripts, or `.env` files. Clear your terminal history if needed: `history -c`

### Final SSM parameter layout under `/insightgen/postgres/`

```
/insightgen/postgres/
├── host                        (existing)
├── port                        (existing)
├── username                    (existing)
├── password                    (existing — ifrs_user master, not used here)
└── common-db/
    └── password                (new — common_user password, connection manager only)
```

---

## Phase 4: Connection Manager Service

### 4.1 Project Structure

```
connection-manager/
├── app/
│   ├── __init__.py
│   ├── config.py
│   ├── kms_helper.py
│   ├── connection_manager.py
│   └── main.py
├── Dockerfile
├── requirements.txt
└── .env.example
```

---

### 4.2 `requirements.txt`

```
fastapi==0.115.0
uvicorn[standard]==0.30.6
psycopg2-binary==2.9.9
boto3==1.35.0
pydantic==2.7.0
```

---

### 4.3 `app/config.py`

```python
import os
from dataclasses import dataclass


@dataclass
class Config:
    aws_region: str            = os.getenv("AWS_REGION", "us-east-1")
    ssm_parameter_name: str    = os.getenv("SSM_PARAMETER_NAME", "/insightgen/postgres/common-db/password")
    common_db_host: str        = os.getenv("COMMON_DB_HOST", "10.132.191.157")
    common_db_port: int        = int(os.getenv("COMMON_DB_PORT", "5432"))
    common_db_name: str        = os.getenv("COMMON_DB_NAME", "common_db")
    common_db_user: str        = os.getenv("COMMON_DB_USER", "common_user")
    common_pool_min: int       = int(os.getenv("COMMON_POOL_MIN", "2"))
    common_pool_max: int       = int(os.getenv("COMMON_POOL_MAX", "10"))
    log_level: str             = os.getenv("LOG_LEVEL", "INFO")


config = Config()
```

---

### 4.4 `app/kms_helper.py`

```python
import base64
import logging
import boto3
from app.config import config

logger = logging.getLogger(__name__)

_kms_client = None


def _get_client():
    global _kms_client
    if _kms_client is None:
        _kms_client = boto3.client("kms", region_name=config.aws_region)
    return _kms_client


def decrypt_password(ciphertext_base64: str) -> str:
    """Decrypt a base64 KMS ciphertext and return the plaintext password."""
    try:
        response = _get_client().decrypt(
            CiphertextBlob=base64.b64decode(ciphertext_base64)
        )
        return response["Plaintext"].decode("utf-8")
    except Exception as exc:
        logger.error("KMS decryption failed: %s", exc)
        raise RuntimeError("Failed to decrypt credential") from exc
```

---

### 4.5 `app/connection_manager.py`

```python
import threading
import logging
import time
import boto3
import psycopg2
import psycopg2.pool
from typing import Dict, Optional
from app.config import config
from app.kms_helper import decrypt_password

logger = logging.getLogger(__name__)


class ConnectionManager:
    """
    Singleton that:
      - Fetches the common DB password from SSM once at startup.
      - Maintains a persistent ThreadedConnectionPool to common_db.
      - Caches decrypted domain credentials in memory after first fetch.

    IMPORTANT: Run uvicorn with --workers 1.
    Multiple workers = multiple processes = multiple instances = no shared cache.
    """

    _instance: Optional["ConnectionManager"] = None
    _class_lock = threading.Lock()

    def __new__(cls) -> "ConnectionManager":
        if cls._instance is None:
            with cls._class_lock:
                if cls._instance is None:
                    obj = super().__new__(cls)
                    obj._initialized = False
                    cls._instance = obj
        return cls._instance

    # ──────────────────────────────────────────────────────────────
    # Startup
    # ──────────────────────────────────────────────────────────────

    def initialize(self) -> None:
        """Called once during application startup (FastAPI lifespan)."""
        if self._initialized:
            return

        with self._class_lock:
            if self._initialized:
                return

            logger.info("━━━ ConnectionManager initialising ━━━")
            logger.info("AWS region       : %s", config.aws_region)
            logger.info("SSM parameter    : %s", config.ssm_parameter_name)
            logger.info("Common DB host   : %s:%s", config.common_db_host, config.common_db_port)
            logger.info("Common DB name   : %s", config.common_db_name)
            logger.info("Common DB user   : %s", config.common_db_user)
            logger.info("Pool size        : min=%s max=%s", config.common_pool_min, config.common_pool_max)

            t0 = time.monotonic()
            logger.info("Fetching common DB password from SSM …")
            common_db_password = self._fetch_ssm_parameter()
            logger.info("SSM fetch completed in %.3fs", time.monotonic() - t0)

            logger.info("Creating connection pool to common DB …")
            t1 = time.monotonic()
            self._common_pool = psycopg2.pool.ThreadedConnectionPool(
                minconn=config.common_pool_min,
                maxconn=config.common_pool_max,
                host=config.common_db_host,
                port=config.common_db_port,
                dbname=config.common_db_name,
                user=config.common_db_user,
                password=common_db_password,
                connect_timeout=5,
            )
            logger.info("Common DB pool ready in %.3fs", time.monotonic() - t1)

            self._credential_cache: Dict[str, dict] = {}
            self._domain_lock = threading.Lock()

            self._initialized = True
            logger.info("━━━ ConnectionManager ready ━━━")

    def _fetch_ssm_parameter(self) -> str:
        ssm = boto3.client("ssm", region_name=config.aws_region)
        response = ssm.get_parameter(
            Name=config.ssm_parameter_name,
            WithDecryption=True,
        )
        logger.info(
            "SSM parameter retrieved: name=%s version=%s",
            config.ssm_parameter_name,
            response["Parameter"]["Version"],
        )
        return response["Parameter"]["Value"]

    # ──────────────────────────────────────────────────────────────
    # Public API
    # ──────────────────────────────────────────────────────────────

    def get_domain_credentials(self, domain_id: str) -> dict:
        """
        Return decrypted credentials for domain_id.
        First call queries common_db and decrypts via KMS.
        Subsequent calls return from the in-memory cache.
        """
        logger.info("[%s] Credential request received", domain_id)

        if domain_id in self._credential_cache:
            logger.info("[%s] Cache HIT — returning cached credentials", domain_id)
            creds = self._credential_cache[domain_id]
            logger.info(
                "[%s] Cached target: host=%s port=%s db=%s user=%s",
                domain_id, creds["host"], creds["port"], creds["db_name"], creds["db_user"],
            )
            return creds

        logger.info("[%s] Cache MISS — fetching from common DB", domain_id)
        with self._domain_lock:
            if domain_id in self._credential_cache:
                logger.info("[%s] Cache HIT after lock (another thread populated it)", domain_id)
                return self._credential_cache[domain_id]

            t0 = time.monotonic()
            raw = self._query_common_db(domain_id)
            logger.info("[%s] Common DB query completed in %.3fs", domain_id, time.monotonic() - t0)
            logger.info(
                "[%s] Raw record: host=%s port=%s db=%s user=%s",
                domain_id, raw["host"], raw["port"], raw["db_name"], raw["db_user"],
            )

            logger.info("[%s] Decrypting password via KMS …", domain_id)
            t1 = time.monotonic()
            raw["password"] = decrypt_password(raw.pop("password_encrypted"))
            logger.info("[%s] KMS decrypt completed in %.3fs", domain_id, time.monotonic() - t1)

            self._credential_cache[domain_id] = raw
            logger.info(
                "[%s] Credentials cached. Cache now holds %d domain(s): %s",
                domain_id,
                len(self._credential_cache),
                list(self._credential_cache.keys()),
            )

        return self._credential_cache[domain_id]

    def health_check(self) -> bool:
        logger.debug("Running health check against common DB …")
        try:
            conn = self._common_pool.getconn()
            with conn.cursor() as cur:
                cur.execute("SELECT 1")
            self._common_pool.putconn(conn)
            logger.debug("Health check passed")
            return True
        except Exception as exc:
            logger.error("Health check FAILED: %s", exc)
            return False

    def invalidate_cache(self, domain_id: str) -> None:
        """Call this if domain credentials are rotated."""
        with self._domain_lock:
            existed = domain_id in self._credential_cache
            self._credential_cache.pop(domain_id, None)
            if existed:
                logger.info(
                    "[%s] Cache entry invalidated. Cache now holds %d domain(s): %s",
                    domain_id,
                    len(self._credential_cache),
                    list(self._credential_cache.keys()),
                )
            else:
                logger.warning("[%s] Cache invalidation requested but entry was not cached", domain_id)

    # ──────────────────────────────────────────────────────────────
    # Internal helpers
    # ──────────────────────────────────────────────────────────────

    def _query_common_db(self, domain_id: str) -> dict:
        logger.info("[%s] Acquiring connection from common pool …", domain_id)
        conn = self._common_pool.getconn()
        logger.info("[%s] Connection acquired", domain_id)
        try:
            with conn.cursor() as cur:
                logger.info("[%s] Executing SELECT on domain_credentials …", domain_id)
                cur.execute(
                    """
                    SELECT host, port, db_name, db_user, password_encrypted
                    FROM   domain_credentials
                    WHERE  domain_id = %s
                    """,
                    (domain_id,),
                )
                row = cur.fetchone()
                if row is None:
                    logger.error("[%s] No row found in domain_credentials", domain_id)
                    raise ValueError(f"No credentials found for domain_id: {domain_id}")
                logger.info("[%s] Row found: host=%s port=%s db=%s user=%s", domain_id, row[0], row[1], row[2], row[3])
                return {
                    "host":               row[0],
                    "port":               row[1],
                    "db_name":            row[2],
                    "db_user":            row[3],
                    "password_encrypted": row[4],
                }
        finally:
            self._common_pool.putconn(conn)
            logger.info("[%s] Connection returned to common pool", domain_id)
```

---

### 4.6 `app/main.py`

```python
import logging
import time
import uuid
from contextlib import asynccontextmanager

from fastapi import FastAPI, HTTPException, Request, Response
from pydantic import BaseModel

from app.config import config
from app.connection_manager import ConnectionManager

logging.basicConfig(
    level=config.log_level,
    format="%(asctime)s  %(levelname)-8s  %(name)s  %(message)s",
)
logger = logging.getLogger(__name__)

manager = ConnectionManager()


@asynccontextmanager
async def lifespan(app: FastAPI):
    manager.initialize()      # runs once at startup
    yield


app = FastAPI(title="Connection Manager", lifespan=lifespan)


# ──────────────────────────────────────────────────────────────────
# Request / response logging middleware
# ──────────────────────────────────────────────────────────────────

@app.middleware("http")
async def log_requests(request: Request, call_next):
    request_id = str(uuid.uuid4())[:8]
    t0 = time.monotonic()

    logger.info(
        "▶ [%s] %s %s  client=%s",
        request_id, request.method, request.url.path, request.client.host,
    )

    try:
        body = await request.body()
        if body:
            logger.info("[%s] Request body : %s", request_id, body.decode("utf-8", errors="replace"))
    except Exception:
        pass

    response: Response = await call_next(request)

    elapsed = (time.monotonic() - t0) * 1000
    logger.info(
        "◀ [%s] %s %s  status=%s  %.1fms",
        request_id, request.method, request.url.path, response.status_code, elapsed,
    )
    response.headers["X-Request-ID"] = request_id
    return response


# ──────────────────────────────────────────────────────────────────
# Routes
# ──────────────────────────────────────────────────────────────────

class CredentialRequest(BaseModel):
    domain_id: str


@app.get("/health")
def health():
    logger.debug("Health check requested")
    if not manager.health_check():
        raise HTTPException(status_code=503, detail="Common DB unreachable")
    return {"status": "ok"}


@app.post("/v1/credentials")
def get_credentials(req: CredentialRequest):
    """
    Returns decrypted DB credentials for the given domain_id.
    Credentials are served from in-memory cache after the first call.
    """
    logger.info("POST /v1/credentials  domain_id=%s", req.domain_id)
    try:
        creds = manager.get_domain_credentials(req.domain_id)
        logger.info(
            "Returning credentials for domain_id=%s  host=%s db=%s user=%s",
            req.domain_id, creds["host"], creds["db_name"], creds["db_user"],
        )
        return creds
    except ValueError as exc:
        logger.warning("domain_id=%s not found: %s", req.domain_id, exc)
        raise HTTPException(status_code=404, detail=str(exc))
    except Exception as exc:
        logger.error("Unexpected error for domain_id=%s: %s", req.domain_id, exc, exc_info=True)
        raise HTTPException(status_code=500, detail="Internal server error")


@app.delete("/v1/credentials/{domain_id}/cache")
def invalidate_cache(domain_id: str):
    """
    Force a re-fetch from common DB on next request.
    Call this after rotating a domain DB password.
    """
    logger.info("Cache invalidation requested for domain_id=%s", domain_id)
    manager.invalidate_cache(domain_id)
    return {"status": "invalidated", "domain_id": domain_id}
```

---

### 4.7 `app/__init__.py`

```python
# intentionally empty
```

---

### 4.8 `.env.example`

```env
AWS_REGION=us-east-1
SSM_PARAMETER_NAME=/insightgen/postgres/common-db/password
COMMON_DB_HOST=10.132.191.157
COMMON_DB_PORT=5432
COMMON_DB_NAME=common_db
COMMON_DB_USER=common_user
COMMON_POOL_MIN=2
COMMON_POOL_MAX=10
LOG_LEVEL=INFO
```

---

### 4.9 `Dockerfile`

```dockerfile
FROM python:3.12-slim

# Install libpq for psycopg2
RUN apt-get update \
    && apt-get install -y --no-install-recommends libpq-dev gcc \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app/ ./app/

EXPOSE 8001

# --workers 1 is mandatory for the singleton to work correctly.
# Multiple workers = multiple processes = separate memory spaces = no shared cache.
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8001", "--workers", "1"]
```

---

### 4.10 Build and Run the Container on EC2

```bash
# SSH into your EC2 instance
# Navigate to the project directory

# Build the image
docker build -t connection-manager:latest .

# Run the container
# The container joins the same Docker network as PostgreSQL
# The EC2 IAM role provides AWS credentials automatically — no keys needed
docker run -d \
  --name connection-manager \
  --network app-network \
  --restart unless-stopped \
  -p 8001:8001 \
  -e AWS_REGION=us-east-1 \
  -e SSM_PARAMETER_NAME=/insightgen/postgres/common-db/password \
  -e COMMON_DB_HOST=10.132.191.157 \
  -e COMMON_DB_PORT=5432 \
  -e COMMON_DB_NAME=common_db \
  -e COMMON_DB_USER=common_user \
  -e LOG_LEVEL=INFO \
  connection-manager:latest

# Verify startup
docker logs connection-manager

# Smoke test the health endpoint
curl http://localhost:8001/health
# Expected: {"status":"ok"}
```

---

## Phase 5: Nginx Location Config

> **nginx runs as a Docker container on this server.** All nginx commands use `docker exec nginx nginx ...` rather than `sudo nginx ...`. The `proxy_pass` directive must use the EC2 private IP so that traffic exits the nginx container, hits the EC2 host network interface, and is forwarded by Docker's port mapping into the connection-manager container. Using `127.0.0.1` here would point to the nginx container's own loopback and always result in a 502.

Add the following `location` block inside your existing `server {}` block:

```nginx
location /db-proxy/ {
    # Use EC2 private IP + exposed container port.
    # 127.0.0.1 does NOT work here — nginx is a container; that address
    # is its own loopback, not the EC2 host. The private IP exits the container
    # and Docker's -p 8001:8001 mapping forwards it to connection-manager.
    proxy_pass          http://10.132.191.157:8001/;

    proxy_http_version  1.1;
    proxy_set_header    Host              $host;
    proxy_set_header    X-Real-IP         $remote_addr;
    proxy_set_header    X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header    X-Forwarded-Proto $scheme;

    proxy_read_timeout    30s;
    proxy_connect_timeout 5s;
    proxy_send_timeout    10s;

    # Restrict access to VPC CIDR only.
    allow 10.0.0.0/8;
    allow 172.16.0.0/12;
    allow 127.0.0.1;
    deny  all;
}
```

After adding the block:

```bash
# Test the config
docker exec nginx nginx -t

# Reload without dropping connections
docker exec nginx nginx -s reload
```

**URL mapping after this config:**

| Nginx URL | Forwards to |
|---|---|
| `POST /db-proxy/v1/credentials` | `POST http://10.132.191.157:8001/v1/credentials` |
| `GET /db-proxy/health` | `GET http://10.132.191.157:8001/health` |
| `DELETE /db-proxy/v1/credentials/{id}/cache` | `DELETE http://10.132.191.157:8001/v1/credentials/{id}/cache` |

---

## Phase 6: Lambda Function (Python 3.12)

### 6.1 Lambda Code — `lambda_function.py`

```python
"""
Lambda function that:
  1. Calls the Connection Manager to get domain DB credentials
  2. Opens its own psycopg2 connection to the domain DB
  3. Queries tenant_data and returns the rows

Requires the psycopg2 Lambda layer to be attached to this function.
"""

import json
import logging
import os
import time
import urllib.error
import urllib.request

import psycopg2

# ── Logging setup ─────────────────────────────────────────────────
logger = logging.getLogger()
logger.setLevel(logging.INFO)

# Avoid duplicate log lines if handler is already attached (Lambda reuse)
if not logger.handlers:
    handler = logging.StreamHandler()
    handler.setFormatter(logging.Formatter(
        "%(asctime)s  %(levelname)-8s  %(message)s"
    ))
    logger.addHandler(handler)

# ── Environment variables ─────────────────────────────────────────
CONNECTION_MANAGER_URL = os.environ["CONNECTION_MANAGER_URL"]
# e.g. http://10.132.191.157/db-proxy   (EC2 private IP + nginx path, no trailing slash)


# ──────────────────────────────────────────────────────────────────
# Helpers
# ──────────────────────────────────────────────────────────────────

def _get_domain_credentials(domain_id: str) -> dict:
    """Call the Connection Manager service and return decrypted credentials."""
    url = f"{CONNECTION_MANAGER_URL}/v1/credentials"
    payload = json.dumps({"domain_id": domain_id}).encode("utf-8")

    logger.info("── Credential fetch ──────────────────────────────────")
    logger.info("URL           : %s", url)
    logger.info("Request body  : %s", payload.decode())

    req = urllib.request.Request(
        url,
        data=payload,
        headers={"Content-Type": "application/json"},
        method="POST",
    )

    t0 = time.monotonic()
    try:
        with urllib.request.urlopen(req, timeout=5) as resp:
            raw = resp.read()
            elapsed = (time.monotonic() - t0) * 1000
            logger.info("HTTP status   : %s", resp.status)
            logger.info("Response time : %.1fms", elapsed)
            creds = json.loads(raw)
            logger.info(
                "Credentials received: host=%s port=%s db=%s user=%s",
                creds.get("host"), creds.get("port"),
                creds.get("db_name"), creds.get("db_user"),
            )
            # Never log the password — log only that it was received
            logger.info("Password field present: %s", "password" in creds)
            return creds
    except urllib.error.HTTPError as exc:
        elapsed = (time.monotonic() - t0) * 1000
        body = exc.read().decode()
        logger.error("HTTP %s from Connection Manager after %.1fms", exc.code, elapsed)
        logger.error("Error body    : %s", body)
        raise RuntimeError(f"Credential fetch failed ({exc.code}): {body}") from exc
    except Exception as exc:
        elapsed = (time.monotonic() - t0) * 1000
        logger.error("Connection Manager unreachable after %.1fms: %s", elapsed, exc)
        raise


def _query_domain_db(creds: dict, domain_id: str) -> list:
    """Open a direct connection to the domain DB and query tenant_data."""
    logger.info("── Domain DB connection ──────────────────────────────")
    logger.info("Host          : %s", creds["host"])
    logger.info("Port          : %s", creds["port"])
    logger.info("Database      : %s", creds["db_name"])
    logger.info("User          : %s", creds["db_user"])

    t0 = time.monotonic()
    try:
        conn = psycopg2.connect(
            host=creds["host"],
            port=creds["port"],
            dbname=creds["db_name"],
            user=creds["db_user"],
            password=creds["password"],
            connect_timeout=5,
        )
        logger.info("Connected to domain DB in %.1fms", (time.monotonic() - t0) * 1000)

        with conn:
            with conn.cursor() as cur:
                query = "SELECT id, domain_id, data_key, data_value, created_at FROM tenant_data ORDER BY id"
                logger.info("Executing query: %s", query)
                t1 = time.monotonic()
                cur.execute(query)
                rows = cur.fetchall()
                columns = [desc[0] for desc in cur.description]
                logger.info(
                    "Query returned %d row(s) in %.1fms",
                    len(rows), (time.monotonic() - t1) * 1000,
                )
                result = [dict(zip(columns, row)) for row in rows]

        conn.close()
        logger.info("Domain DB connection closed")
        return result

    except psycopg2.OperationalError as exc:
        logger.error(
            "Cannot connect to domain DB %s@%s:%s/%s — %s",
            creds["db_user"], creds["host"], creds["port"], creds["db_name"], exc,
        )
        raise


# ──────────────────────────────────────────────────────────────────
# Handler
# ──────────────────────────────────────────────────────────────────

def lambda_handler(event, context):
    invocation_start = time.monotonic()

    logger.info("══════════════════════════════════════════════════════")
    logger.info("Lambda invoked")
    logger.info("Function      : %s", context.function_name)
    logger.info("Request ID    : %s", context.aws_request_id)
    logger.info("Remaining ms  : %s", context.get_remaining_time_in_millis())
    logger.info("Event         : %s", json.dumps(event))
    logger.info("CONNECTION_MANAGER_URL: %s", CONNECTION_MANAGER_URL)

    # ── Resolve domain_id ────────────────────────────────────────
    domain_id = (
        event.get("domain_id")
        or (event.get("headers") or {}).get("domain_id")
        or (event.get("headers") or {}).get("x-domain-id")
    )
    logger.info("Resolved domain_id: %s", domain_id)

    if not domain_id:
        logger.warning("No domain_id found in event — returning 400")
        return {
            "statusCode": 400,
            "body": json.dumps({"error": "domain_id is required"}),
        }

    # ── Step 1: get credentials ──────────────────────────────────
    try:
        creds = _get_domain_credentials(domain_id)
    except RuntimeError as exc:
        logger.error("Credential fetch failed: %s", exc)
        return {"statusCode": 502, "body": json.dumps({"error": str(exc)})}
    except Exception as exc:
        logger.error("Unexpected error during credential fetch: %s", exc, exc_info=True)
        return {"statusCode": 500, "body": json.dumps({"error": "Internal error"})}

    # ── Step 2: query domain DB ──────────────────────────────────
    try:
        rows = _query_domain_db(creds, domain_id)
    except psycopg2.OperationalError as exc:
        logger.error("Domain DB connection failed: %s", exc)
        return {"statusCode": 503, "body": json.dumps({"error": "Domain DB unreachable"})}
    except Exception as exc:
        logger.error("Unexpected error during DB query: %s", exc, exc_info=True)
        return {"statusCode": 500, "body": json.dumps({"error": "Internal error"})}

    # ── Response ─────────────────────────────────────────────────
    total_ms = (time.monotonic() - invocation_start) * 1000
    logger.info("── Summary ───────────────────────────────────────────")
    logger.info("domain_id     : %s", domain_id)
    logger.info("Rows returned : %d", len(rows))
    logger.info("Total time    : %.1fms", total_ms)
    logger.info("══════════════════════════════════════════════════════")

    # Serialize datetime objects for JSON
    def serialise(obj):
        return str(obj) if hasattr(obj, "isoformat") else obj

    serialisable_rows = [
        {k: serialise(v) for k, v in row.items()} for row in rows
    ]

    return {
        "statusCode": 200,
        "body": json.dumps({
            "domain_id":   domain_id,
            "row_count":   len(rows),
            "rows":        serialisable_rows,
        }),
    }
```

---

### 6.2 Attach the Existing psycopg2 Lambda Layer

Since you already have a psycopg2 Lambda layer, attach it when creating the function. First, find the layer ARN:

```bash
# List your existing layers to find the psycopg2 one
aws lambda list-layers \
  --region us-east-1 \
  --query "Layers[*].{Name:LayerName, ARN:LatestMatchingVersion.LayerVersionArn}" \
  --output table
```

Package only the function code (no dependencies needed):

```bash
zip lambda_deployment.zip lambda_function.py
```

Deploy the function with the layer ARN from the list above:

```bash
aws lambda create-function \
  --function-name DomainDBConnector \
  --runtime python3.12 \
  --role arn:aws:iam::<ACCOUNT_ID>:role/HCL-User-Role-InsightGen-Lambda \
  --handler lambda_function.lambda_handler \
  --zip-file fileb://lambda_deployment.zip \
  --timeout 30 \
  --memory-size 256 \
  --layers arn:aws:lambda:us-east-1:<ACCOUNT_ID>:layer:<YOUR_PSYCOPG2_LAYER_NAME>:<VERSION> \
  --environment Variables="{CONNECTION_MANAGER_URL=http://10.132.191.157/db-proxy}" \
  --region us-east-1
```

---

### 6.3 Lambda VPC Configuration

Lambda must be in the same VPC as the EC2 instance to reach it via private IP:

```bash
aws lambda update-function-configuration \
  --function-name DomainDBConnector \
  --vpc-config SubnetIds=<SUBNET_ID_1>,<SUBNET_ID_2>,SecurityGroupIds=<LAMBDA_SG_ID> \
  --region us-east-1
```

**Security group rules required:**

| Resource | Rule type | Port | Source / Destination |
|---|---|---|---|
| EC2 security group | Inbound | 80 (nginx) | Lambda security group |
| EC2 security group | Inbound | 5432 (postgres) | Lambda security group |
| Lambda security group | Outbound | 80 | EC2 security group |
| Lambda security group | Outbound | 5432 | EC2 security group |

---

## Phase 7: End-to-End Test

### 7.1 Test the Connection Manager directly from EC2

```bash
# From the EC2 instance
curl -s -X POST http://localhost:8001/v1/credentials \
  -H "Content-Type: application/json" \
  -d '{"domain_id": "tenant_a"}' | python3 -m json.tool

# Expected:
# {
#   "host": "10.132.191.157",
#   "port": 5432,
#   "db_name": "domain_db_a",
#   "db_user": "domain_a_user",
#   "password": "DomainAPass456!"
# }
```

### 7.2 Test through Nginx

```bash
curl -s -X POST http://localhost/db-proxy/v1/credentials \
  -H "Content-Type: application/json" \
  -d '{"domain_id": "tenant_b"}' | python3 -m json.tool
```

### 7.3 Test the Lambda from the AWS Console

1. Open the [AWS Lambda console](https://console.aws.amazon.com/lambda) and select the `DomainDBConnector` function.
2. Click the **Test** tab.
3. Click **Create new event** (or select an existing test event from the dropdown).
4. Set **Event name** to `TestTenantA`.
5. Replace the event JSON with:

```json
{
  "domain_id": "tenant_a"
}
```

6. Click **Save** then click **Test**.
7. Expand the **Execution result** panel. A successful response looks like:

```json
{
  "statusCode": 200,
  "body": "{\"domain_id\": \"tenant_a\", \"rows\": []}"
}
```

8. Repeat with `"domain_id": "tenant_b"` to verify the second domain.

**Reading the logs:** Scroll down to the **Log output** section in the Execution result panel. If the connection fails, the error from psycopg2 or the Connection Manager will appear there. You can also open **CloudWatch → Log groups → /aws/lambda/DomainDBConnector** for the full log stream.

---

## Security Notes

| Risk | Mitigation applied |
|---|---|
| Common DB password in plain text | Stored only in SSM SecureString, encrypted with KMS CMK. Fetched in memory at startup, never written to disk. |
| Domain DB passwords in common DB | Stored as KMS ciphertext. Plain text only exists in process memory after decryption. |
| Credential endpoint reachable only from VPC | Nginx `deny all` outside VPC CIDRs blocks any access from outside the private network. |
| Lambda returning credentials over wire | All traffic is internal VPC. Add TLS on nginx (`ssl_certificate`) for defence in depth. |
| KMS key misuse | Key policy `HCL-USER-KEY-insightgen-postgres-credentials` restricts decrypt to EC2 instance profile and Lambda role only. EC2 has two scoped statements: one via SSM (for the SSM parameter), one direct (for DB table ciphertexts). |
| Credential cache stale after password rotation | Call `DELETE /v1/credentials/{domain_id}/cache` to force a fresh fetch. |
| Multiple uvicorn workers breaking singleton | Dockerfile hard-codes `--workers 1`. Document this constraint clearly for anyone modifying the Dockerfile. |
