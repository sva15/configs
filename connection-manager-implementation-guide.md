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

### 1.1 Create a Customer Managed KMS Key

Run this from any machine with appropriate AWS permissions (your laptop or a bastion).

```bash
# Create the CMK
aws kms create-key \
  --description "CMK for connection manager DB credentials" \
  --key-usage ENCRYPT_DECRYPT \
  --region ap-south-1 \
  --query "KeyMetadata.KeyId" \
  --output text
# Note the KeyId returned, e.g.: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

# Create a human-readable alias
aws kms create-alias \
  --alias-name alias/app-db-cmk \
  --target-key-id <KeyId-from-above> \
  --region ap-south-1
```

**Set the key policy** — only the EC2 instance role should be able to decrypt:

```bash
# First, get your EC2 instance's IAM role ARN
# (replace with your actual role ARN in the policy below)

aws kms put-key-policy \
  --key-id alias/app-db-cmk \
  --policy-name default \
  --region ap-south-1 \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "Enable IAM User Permissions",
        "Effect": "Allow",
        "Principal": {"AWS": "arn:aws:iam::<ACCOUNT_ID>:root"},
        "Action": "kms:*",
        "Resource": "*"
      },
      {
        "Sid": "Allow EC2 role to decrypt",
        "Effect": "Allow",
        "Principal": {"AWS": "arn:aws:iam::<ACCOUNT_ID>:role/<YOUR_EC2_ROLE_NAME>"},
        "Action": ["kms:Decrypt", "kms:DescribeKey"],
        "Resource": "*"
      },
      {
        "Sid": "Allow admin to encrypt during setup",
        "Effect": "Allow",
        "Principal": {"AWS": "arn:aws:iam::<ACCOUNT_ID>:user/<YOUR_ADMIN_USER>"},
        "Action": ["kms:Encrypt", "kms:Decrypt", "kms:DescribeKey"],
        "Resource": "*"
      }
    ]
  }'
```

> **Note:** Replace `<ACCOUNT_ID>`, `<YOUR_EC2_ROLE_NAME>`, and `<YOUR_ADMIN_USER>` with your actual values.

---

### 1.2 Create IAM Role for EC2 (Connection Manager)

If your EC2 instance does not already have an IAM role, create one and attach it.

Create a file `ec2-trust-policy.json`:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {"Service": "ec2.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }
  ]
}
```

```bash
# Create the role
aws iam create-role \
  --role-name ConnectionManagerEC2Role \
  --assume-role-policy-document file://ec2-trust-policy.json

# Attach an inline policy for SSM + KMS
aws iam put-role-policy \
  --role-name ConnectionManagerEC2Role \
  --policy-name ConnectionManagerPolicy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": ["ssm:GetParameter"],
        "Resource": "arn:aws:ssm:ap-south-1:<ACCOUNT_ID>:parameter/app/common-db/password"
      },
      {
        "Effect": "Allow",
        "Action": ["kms:Decrypt", "kms:DescribeKey"],
        "Resource": "arn:aws:kms:ap-south-1:<ACCOUNT_ID>:key/<CMK_KEY_ID>"
      }
    ]
  }'
```

---

### 1.3 Create IAM Role for Lambda

The Lambda role only needs network access to reach the EC2 instance. It does **not** need KMS or SSM permissions — those are handled entirely by the Connection Manager.

```bash
# Lambda trust policy
aws iam create-role \
  --role-name ConnectionManagerLambdaRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "lambda.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

# Attach basic execution + VPC networking
aws iam attach-role-policy \
  --role-name ConnectionManagerLambdaRole \
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
  -e POSTGRES_PASSWORD=postgres_root_pass \
  -v postgres_data:/var/lib/postgresql/data \
  postgres:16
```

### 2.2 Create Databases, Users, and Schema

```bash
# Exec into the PostgreSQL container
docker exec -it postgres psql -U postgres
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
docker exec -it postgres psql -U postgres -d common_db
```

```sql
-- Run as postgres superuser inside common_db

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
docker exec -it postgres psql -U postgres -d domain_db_a
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
docker exec -it postgres psql -U postgres -d domain_db_b
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

Run these commands from **your local machine or bastion** (you need AWS credentials with `kms:Encrypt` permission):

```bash
# Encrypt domain_a password
echo -n "DomainAPass456!" > /tmp/pass_a.txt
CIPHER_A=$(aws kms encrypt \
  --key-id alias/app-db-cmk \
  --plaintext fileb:///tmp/pass_a.txt \
  --query CiphertextBlob \
  --output text \
  --region ap-south-1)
echo "Ciphertext A: $CIPHER_A"
rm /tmp/pass_a.txt

# Encrypt domain_b password
echo -n "DomainBPass789!" > /tmp/pass_b.txt
CIPHER_B=$(aws kms encrypt \
  --key-id alias/app-db-cmk \
  --plaintext fileb:///tmp/pass_b.txt \
  --query CiphertextBlob \
  --output text \
  --region ap-south-1)
echo "Ciphertext B: $CIPHER_B"
rm /tmp/pass_b.txt
```

> **Important:** Copy the ciphertext values (`$CIPHER_A` and `$CIPHER_B`). You will paste them into the SQL below.
> Delete the temp files immediately after — never let the plaintext touch disk for longer than needed.

---

### 2.4 Insert Encrypted Credentials into Common DB

The `host` field must be the **EC2 private IP** — this is the address Lambda (inside the VPC) will use to connect to the domain databases.

```bash
# Get your EC2 private IP
EC2_PRIVATE_IP=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)
echo $EC2_PRIVATE_IP
```

```bash
docker exec -it postgres psql -U postgres -d common_db
```

```sql
-- Replace the ciphertext values with the actual output from the KMS encrypt commands above.
-- Replace 10.0.1.55 with your actual EC2 private IP.

INSERT INTO domain_credentials (domain_id, host, port, db_name, db_user, password_encrypted)
VALUES
  (
    'tenant_a',
    '10.0.1.55',
    5432,
    'domain_db_a',
    'domain_a_user',
    'AQICAHxxx...PASTE_CIPHER_A_HERE...xxx'
  ),
  (
    'tenant_b',
    '10.0.1.55',
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

## Phase 3: Store Common DB Password in SSM Parameter Store

Now that the common DB is running, store its password as a SecureString in SSM:

```bash
aws ssm put-parameter \
  --name "/app/common-db/password" \
  --value "CommonDbPass123!" \
  --type SecureString \
  --key-id alias/app-db-cmk \
  --description "Password for the connection manager common database" \
  --region ap-south-1

# Verify it was stored (--with-decryption to confirm the CMK can read it)
aws ssm get-parameter \
  --name "/app/common-db/password" \
  --with-decryption \
  --region ap-south-1
```

> After confirming SSM holds the value correctly, you should no longer need the plain text password anywhere. Do not store it in your shell history, scripts, or `.env` files on disk.

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
    aws_region: str            = os.getenv("AWS_REGION", "ap-south-1")
    ssm_parameter_name: str    = os.getenv("SSM_PARAMETER_NAME", "/app/common-db/password")
    common_db_host: str        = os.getenv("COMMON_DB_HOST", "postgres")   # Docker service name
    common_db_port: int        = int(os.getenv("COMMON_DB_PORT", "5432"))
    common_db_name: str        = os.getenv("COMMON_DB_NAME", "common_db")
    common_db_user: str        = os.getenv("COMMON_DB_USER", "common_user")
    common_pool_min: int       = int(os.getenv("COMMON_POOL_MIN", "2"))
    common_pool_max: int       = int(os.getenv("COMMON_POOL_MAX", "10"))
    api_key: str               = os.getenv("API_KEY", "")
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

            logger.info("Initialising ConnectionManager …")
            common_db_password = self._fetch_ssm_parameter()

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

            # { domain_id: {"host":…, "port":…, "db_name":…, "db_user":…, "password":…} }
            self._credential_cache: Dict[str, dict] = {}
            self._domain_lock = threading.Lock()

            self._initialized = True
            logger.info("ConnectionManager ready.")

    def _fetch_ssm_parameter(self) -> str:
        ssm = boto3.client("ssm", region_name=config.aws_region)
        response = ssm.get_parameter(
            Name=config.ssm_parameter_name,
            WithDecryption=True,
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
        if domain_id not in self._credential_cache:
            with self._domain_lock:
                # Double-checked locking — another thread may have filled it
                if domain_id not in self._credential_cache:
                    raw = self._query_common_db(domain_id)
                    raw["password"] = decrypt_password(raw.pop("password_encrypted"))
                    self._credential_cache[domain_id] = raw
                    logger.info("Credentials cached for domain_id=%s", domain_id)

        return self._credential_cache[domain_id]

    def health_check(self) -> bool:
        try:
            conn = self._common_pool.getconn()
            with conn.cursor() as cur:
                cur.execute("SELECT 1")
            self._common_pool.putconn(conn)
            return True
        except Exception as exc:
            logger.error("Health check failed: %s", exc)
            return False

    def invalidate_cache(self, domain_id: str) -> None:
        """Call this if domain credentials are rotated."""
        with self._domain_lock:
            self._credential_cache.pop(domain_id, None)
            logger.info("Cache invalidated for domain_id=%s", domain_id)

    # ──────────────────────────────────────────────────────────────
    # Internal helpers
    # ──────────────────────────────────────────────────────────────

    def _query_common_db(self, domain_id: str) -> dict:
        conn = self._common_pool.getconn()
        try:
            with conn.cursor() as cur:
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
                    raise ValueError(f"No credentials found for domain_id: {domain_id}")
                return {
                    "host":               row[0],
                    "port":               row[1],
                    "db_name":            row[2],
                    "db_user":            row[3],
                    "password_encrypted": row[4],
                }
        finally:
            self._common_pool.putconn(conn)
```

---

### 4.6 `app/main.py`

```python
import logging
from contextlib import asynccontextmanager

from fastapi import Depends, FastAPI, Header, HTTPException
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
    # teardown (if needed) goes here


app = FastAPI(title="Connection Manager", lifespan=lifespan)


# ──────────────────────────────────────────────────────────────────
# Auth dependency
# ──────────────────────────────────────────────────────────────────

def verify_api_key(x_api_key: str = Header(...)):
    if not config.api_key or x_api_key != config.api_key:
        raise HTTPException(status_code=401, detail="Invalid or missing API key")


# ──────────────────────────────────────────────────────────────────
# Routes
# ──────────────────────────────────────────────────────────────────

class CredentialRequest(BaseModel):
    domain_id: str


@app.get("/health")
def health():
    if not manager.health_check():
        raise HTTPException(status_code=503, detail="Common DB unreachable")
    return {"status": "ok"}


@app.post("/v1/credentials", dependencies=[Depends(verify_api_key)])
def get_credentials(req: CredentialRequest):
    """
    Returns decrypted DB credentials for the given domain_id.
    Credentials are served from in-memory cache after the first call.
    """
    try:
        creds = manager.get_domain_credentials(req.domain_id)
        return creds
    except ValueError as exc:
        raise HTTPException(status_code=404, detail=str(exc))
    except Exception as exc:
        logger.error("Unexpected error for domain_id=%s: %s", req.domain_id, exc)
        raise HTTPException(status_code=500, detail="Internal server error")


@app.delete("/v1/credentials/{domain_id}/cache", dependencies=[Depends(verify_api_key)])
def invalidate_cache(domain_id: str):
    """
    Force a re-fetch from common DB on next request.
    Call this after rotating a domain DB password.
    """
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
AWS_REGION=ap-south-1
SSM_PARAMETER_NAME=/app/common-db/password
COMMON_DB_HOST=postgres
COMMON_DB_PORT=5432
COMMON_DB_NAME=common_db
COMMON_DB_USER=common_user
COMMON_POOL_MIN=2
COMMON_POOL_MAX=10
API_KEY=replace-with-a-strong-random-string
LOG_LEVEL=INFO
```

> Generate a strong API key: `python3 -c "import secrets; print(secrets.token_urlsafe(32))"`

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
  -e AWS_REGION=ap-south-1 \
  -e SSM_PARAMETER_NAME=/app/common-db/password \
  -e COMMON_DB_HOST=postgres \
  -e COMMON_DB_PORT=5432 \
  -e COMMON_DB_NAME=common_db \
  -e COMMON_DB_USER=common_user \
  -e API_KEY=<your-generated-api-key> \
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

Add the following `location` block inside your existing `server {}` block in `/etc/nginx/conf.d/default.conf`:

```nginx
location /db-proxy/ {
    # Strip the /db-proxy prefix before forwarding.
    # The trailing slash on proxy_pass is important — it rewrites the path.
    proxy_pass          http://127.0.0.1:8001/;

    proxy_http_version  1.1;
    proxy_set_header    Host              $host;
    proxy_set_header    X-Real-IP         $remote_addr;
    proxy_set_header    X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header    X-Forwarded-Proto $scheme;

    proxy_read_timeout    30s;
    proxy_connect_timeout 5s;
    proxy_send_timeout    10s;

    # Restrict access to VPC CIDR only.
    # Adjust the CIDR to match your VPC range.
    allow 10.0.0.0/8;
    allow 172.16.0.0/12;
    allow 127.0.0.1;
    deny  all;
}
```

After adding the block:

```bash
# Test the config
sudo nginx -t

# Reload without dropping connections
sudo nginx -s reload
```

**URL mapping after this config:**

| Nginx URL | Forwards to |
|---|---|
| `POST /db-proxy/v1/credentials` | `POST http://127.0.0.1:8001/v1/credentials` |
| `GET /db-proxy/health` | `GET http://127.0.0.1:8001/health` |
| `DELETE /db-proxy/v1/credentials/{id}/cache` | `DELETE http://127.0.0.1:8001/v1/credentials/{id}/cache` |

---

## Phase 6: Lambda Function (Python 3.12)

### 6.1 Lambda Code — `lambda_function.py`

```python
"""
Lambda function that:
  1. Calls the Connection Manager to get domain DB credentials
  2. Opens its own psycopg2 connection to the domain DB
  3. Runs a query and returns results

NOTE: Install psycopg2-binary as a Lambda layer or package it with this function.
"""

import json
import logging
import os
import urllib.error
import urllib.request

import psycopg2

logger = logging.getLogger()
logger.setLevel(logging.INFO)

# Set in Lambda environment variables
CONNECTION_MANAGER_URL = os.environ["CONNECTION_MANAGER_URL"]
# e.g. http://10.0.1.55/db-proxy   (EC2 private IP + nginx path, no trailing slash)

API_KEY = os.environ["CONNECTION_MANAGER_API_KEY"]


def _get_domain_credentials(domain_id: str) -> dict:
    """Call the Connection Manager service and return decrypted credentials."""
    url = f"{CONNECTION_MANAGER_URL}/v1/credentials"
    payload = json.dumps({"domain_id": domain_id}).encode("utf-8")

    req = urllib.request.Request(
        url,
        data=payload,
        headers={
            "Content-Type": "application/json",
            "X-API-Key": API_KEY,
        },
        method="POST",
    )

    try:
        with urllib.request.urlopen(req, timeout=5) as resp:
            return json.loads(resp.read())
    except urllib.error.HTTPError as exc:
        body = exc.read().decode()
        logger.error("Connection Manager returned %s: %s", exc.code, body)
        raise RuntimeError(f"Credential fetch failed ({exc.code}): {body}") from exc


def lambda_handler(event, context):
    # domain_id can come from the event body or headers (API Gateway)
    domain_id = (
        event.get("domain_id")
        or (event.get("headers") or {}).get("domain_id")
        or (event.get("headers") or {}).get("x-domain-id")
    )

    if not domain_id:
        return {
            "statusCode": 400,
            "body": json.dumps({"error": "domain_id is required"}),
        }

    try:
        creds = _get_domain_credentials(domain_id)
        logger.info("Connecting to domain DB for domain_id=%s", domain_id)

        with psycopg2.connect(
            host=creds["host"],
            port=creds["port"],
            dbname=creds["db_name"],
            user=creds["db_user"],
            password=creds["password"],
            connect_timeout=5,
        ) as conn:
            with conn.cursor() as cur:
                # Replace with your actual query
                cur.execute(
                    "SELECT id, data_key, data_value FROM tenant_data LIMIT 10"
                )
                rows = cur.fetchall()
                columns = [desc[0] for desc in cur.description]
                result = [dict(zip(columns, row)) for row in rows]

        return {
            "statusCode": 200,
            "body": json.dumps({"domain_id": domain_id, "rows": result}),
        }

    except RuntimeError as exc:
        return {"statusCode": 502, "body": json.dumps({"error": str(exc)})}
    except psycopg2.OperationalError as exc:
        logger.error("Domain DB connection failed: %s", exc)
        return {"statusCode": 503, "body": json.dumps({"error": "Domain DB unreachable"})}
    except Exception as exc:
        logger.error("Unexpected error: %s", exc)
        return {"statusCode": 500, "body": json.dumps({"error": "Internal error"})}
```

---

### 6.2 Package psycopg2 with the Lambda

psycopg2 has native C extensions and cannot be `pip install`-ed on your laptop and uploaded directly. Use the pre-compiled binary:

```bash
# Create a deployment package
mkdir lambda_package && cd lambda_package
pip install psycopg2-binary --target . --platform manylinux2014_x86_64 \
    --implementation cp --python-version 3.12 --only-binary :all:
cp ../lambda_function.py .
zip -r ../lambda_deployment.zip .
cd ..
```

```bash
# Deploy to Lambda
aws lambda create-function \
  --function-name DomainDBConnector \
  --runtime python3.12 \
  --role arn:aws:iam::<ACCOUNT_ID>:role/ConnectionManagerLambdaRole \
  --handler lambda_function.lambda_handler \
  --zip-file fileb://lambda_deployment.zip \
  --timeout 30 \
  --memory-size 256 \
  --environment Variables="{
    CONNECTION_MANAGER_URL=http://10.0.1.55/db-proxy,
    CONNECTION_MANAGER_API_KEY=<your-api-key>
  }" \
  --region ap-south-1
```

---

### 6.3 Lambda VPC Configuration

Lambda must be in the same VPC as the EC2 instance to reach it via private IP:

```bash
aws lambda update-function-configuration \
  --function-name DomainDBConnector \
  --vpc-config SubnetIds=<SUBNET_ID_1>,<SUBNET_ID_2>,SecurityGroupIds=<LAMBDA_SG_ID> \
  --region ap-south-1
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
  -H "X-API-Key: <your-api-key>" \
  -d '{"domain_id": "tenant_a"}' | python3 -m json.tool

# Expected:
# {
#   "host": "10.0.1.55",
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
  -H "X-API-Key: <your-api-key>" \
  -d '{"domain_id": "tenant_b"}' | python3 -m json.tool
```

### 7.3 Invoke the Lambda manually

```bash
aws lambda invoke \
  --function-name DomainDBConnector \
  --payload '{"domain_id": "tenant_a"}' \
  --cli-binary-format raw-in-base64-out \
  response.json \
  --region ap-south-1

cat response.json | python3 -m json.tool
```

---

## Security Notes

| Risk | Mitigation applied |
|---|---|
| Common DB password in plain text | Stored only in SSM SecureString, encrypted with KMS CMK. Fetched in memory at startup, never written to disk. |
| Domain DB passwords in common DB | Stored as KMS ciphertext. Plain text only exists in process memory after decryption. |
| Credential endpoint exposed | Protected by `X-API-Key` header. Nginx `deny all` outside VPC CIDRs blocks external access. |
| Lambda returning credentials over wire | All traffic is internal VPC. Add TLS on nginx (`ssl_certificate`) for defence in depth. |
| KMS key misuse | CMK key policy restricts `kms:Decrypt` to the EC2 IAM role only. |
| Credential cache stale after password rotation | Call `DELETE /v1/credentials/{domain_id}/cache` to force a fresh fetch. |
| Multiple uvicorn workers breaking singleton | Dockerfile hard-codes `--workers 1`. Document this constraint clearly for anyone modifying the Dockerfile. |
