# Secure PostgreSQL Architecture on AWS EC2
### Production-Grade Design for Sensitive Data Handling

> **Scope:** PostgreSQL in Docker on EC2 · AWS Lambda (Python 3.12) · AWS KMS · Parameter Store · Secrets Manager  
> **Goal:** Password hashing, field-level encryption, secure key management, and a fully testable implementation.

---

## Table of Contents

1. [Core Concepts](#1-core-concepts)
2. [System Design](#2-system-design)
3. [AWS-Based Architecture](#3-aws-based-architecture)
4. [Architecture Diagram](#4-architecture-diagram)
5. [Implementation Guide](#5-implementation-guide)
6. [Testing & Validation](#6-testing--validation)

---

## 1. Core Concepts

### 1.1 Hashing vs Encryption — When to Use Each

Understanding the difference between hashing and encryption is foundational. They are **not interchangeable**.

| Property | Hashing | Encryption |
|---|---|---|
| **Reversible?** | ❌ No (one-way) | ✅ Yes (with key) |
| **Use case** | Passwords, integrity checks | API keys, PII, sensitive fields |
| **Key required?** | No | Yes |
| **Output** | Fixed-length digest | Variable-length ciphertext |
| **Examples** | bcrypt, Argon2, SHA-256 | AES-256-GCM, RSA |

**Golden Rule:**
- **Passwords → always hash.** You never need to know the original password — you only need to verify it. Hashing makes this irreversible by design.
- **Sensitive fields that must be read back → encrypt.** Examples: national ID numbers, API keys, credit card numbers, medical records.

---

### 1.2 Password Hashing Algorithms

#### bcrypt
The most widely deployed password hashing function. Deliberately slow and includes a built-in salt.

```
$2b$12$<22-char-salt><31-char-hash>
```

- **Cost factor (work factor):** Controls iteration count. `12` is a common production value (~250ms per hash on modern hardware). Increase as hardware gets faster.
- **Max input:** 72 bytes (passwords longer than this are silently truncated — use pre-hashing if needed).
- **Python library:** `bcrypt`

#### Argon2
Winner of the Password Hashing Competition (2015). The modern recommendation.

- **Three variants:**
  - `argon2id` — **recommended** (hybrid of argon2i and argon2d, resistant to both side-channel and GPU attacks)
  - `argon2i` — side-channel resistant
  - `argon2d` — GPU resistant
- **Parameters:** `time_cost` (iterations), `memory_cost` (RAM in KB), `parallelism` (threads)
- **Python library:** `argon2-cffi`

**Recommendation:** Use **Argon2id** for new systems. Use bcrypt only when integrating with legacy systems or when library support for Argon2 is limited.

---

### 1.3 Is PostgreSQL Alone Sufficient?

No. PostgreSQL provides valuable built-in capabilities, but alone it is **not sufficient** for a production-grade sensitive data architecture.

| Layer | What PostgreSQL Provides | What it Lacks |
|---|---|---|
| **Storage** | `pgcrypto` extension for symmetric/asymmetric encryption | Secure key management, key rotation, audit trails |
| **Access** | Row-level security (RLS), column-level permissions | Integration with IAM/SSO, centralized secret management |
| **Audit** | `pgaudit` extension | Centralised logging, alerting, compliance reporting |
| **Transport** | TLS for connections | Application-level encryption at rest |

**What you need beyond PostgreSQL:**

- **AWS KMS** — Managed cryptographic key lifecycle (creation, rotation, deletion, access policy)
- **AWS Secrets Manager / Parameter Store** — Secure runtime retrieval of keys by the application
- **Application-layer encryption** — Encrypt before writing to DB; decrypt after reading
- **IAM roles** — Least-privilege access to KMS and secrets

---

### 1.4 How Encryption/Decryption Works in Real Systems

#### Write Path (Encrypting before storage)

```
User Input → Application Layer
  → Retrieve encryption key (from KMS/Parameter Store)
  → Encrypt field value (AES-256-GCM)
  → Store ciphertext + IV + auth tag in PostgreSQL
```

#### Read Path (Decrypting after retrieval)

```
PostgreSQL → Ciphertext + IV
  → Application Layer
  → Retrieve encryption key (from KMS/Parameter Store)
  → Decrypt field value
  → Return plaintext to authorised caller
```

**Critical principles:**
- **Encryption happens in the application layer, not in the database.** The database stores opaque ciphertext and never sees the key.
- **Never store encryption keys in the same system as the data they protect.**
- **Use authenticated encryption** (AES-256-GCM) — it provides both confidentiality *and* integrity (tamper detection). Never use raw AES-ECB or AES-CBC without a MAC.
- **IV (Initialization Vector) must be unique per encryption operation.** Reusing an IV with the same key breaks security.

---

## 2. System Design

### 2.1 Layered Security Model

```
┌─────────────────────────────────────────────────────────┐
│  APPLICATION LAYER  (Python Lambda / App Server)         │
│  • Password hashing (Argon2id)                           │
│  • Field encryption/decryption (AES-256-GCM)             │
│  • Key retrieval from AWS KMS / Parameter Store          │
│  • IAM-authenticated API calls only                      │
├─────────────────────────────────────────────────────────┤
│  DATABASE LAYER  (PostgreSQL in Docker on EC2)           │
│  • Stores only: hashed passwords, ciphertext             │
│  • TLS enforced for all connections                      │
│  • Encrypted EBS volume                                  │
│  • Least-privilege DB roles per service                  │
├─────────────────────────────────────────────────────────┤
│  KEY MANAGEMENT LAYER  (AWS KMS + Parameter Store)       │
│  • Encryption keys never leave KMS in plaintext          │
│  • IAM policies restrict key usage to specific roles     │
│  • CloudTrail logs every key usage event                 │
└─────────────────────────────────────────────────────────┘
```

---

### 2.2 Field Classification

Before designing the schema, classify each field by its security requirement:

| Field | Classification | Strategy |
|---|---|---|
| `password` | Secret, never read back | **Hash (Argon2id)** |
| `api_key` | Secret, must be read back | **Encrypt (AES-256-GCM via KMS)** |
| `national_id` | PII, must be read back | **Encrypt (AES-256-GCM via KMS)** |
| `email` | PII, queryable | Encrypt + maintain indexed hash for lookups |
| `username` | Non-sensitive | Plaintext |
| `created_at` | Non-sensitive | Plaintext |

---

### 2.3 Application Layer vs Database Layer Responsibilities

#### Application Layer Owns:
- **All cryptographic operations** — hashing, encryption, decryption
- **Key retrieval** — calls to AWS KMS or Parameter Store
- **Access control** — validates that the caller is authorised before decrypting
- **Input validation** — sanitises and validates before writing to DB

#### Database Layer Owns:
- **Durable, encrypted-at-rest storage** (EBS encryption + TLS in transit)
- **Referential integrity and schema constraints**
- **Row-level security** — database roles see only what they are permitted to see
- **Audit logging** via `pgaudit`

#### Why NOT use `pgcrypto` for primary encryption?

`pgcrypto` is a PostgreSQL extension that performs encryption inside the DB process. **Avoid it as your primary strategy** for these reasons:
- The key must be passed as a function argument in the SQL query — it appears in `pg_stat_activity`, query logs, and any monitoring system
- There is no integration with KMS or key rotation
- A compromised DB superuser or log file exposes both the data and the key

Use `pgcrypto` only for supplementary operations (e.g., generating UUIDs, internal digests).

---

### 2.4 Key Management Strategy

Use **envelope encryption** — the AWS-native pattern:

```
Data Encryption Key (DEK)  ←  what actually encrypts your data
         ↓
   Encrypted by KMS Customer Master Key (CMK)
         ↓
   Encrypted DEK stored alongside ciphertext in DB
         ↓
   To decrypt: send encrypted DEK to KMS → KMS returns plaintext DEK → decrypt data
```

**Why envelope encryption?**
- KMS never exposes the CMK — all operations happen inside KMS hardware
- DEK rotation is possible without re-encrypting all data (just re-wrap the DEK)
- Enables fine-grained IAM control at the CMK level
- Scales: each row can have its own DEK without per-row KMS API calls at write time

---

## 3. AWS-Based Architecture

### 3.1 AWS Services and Their Roles

#### AWS KMS (Key Management Service)
- Hosts the **Customer Master Key (CMK)** — a logical key that never leaves the KMS hardware
- Used to **GenerateDataKey** (returns a plaintext DEK + encrypted DEK)
- Used to **Decrypt** an encrypted DEK back to plaintext
- All KMS usage is logged in **CloudTrail**
- Supports automatic **annual key rotation** (enable via key policy)

#### AWS Systems Manager Parameter Store (SecureString)
- Stores configuration secrets encrypted with a KMS key: DB connection strings, database passwords, service credentials
- **Free tier available** for standard parameters (up to 4KB)
- Best for: static or infrequently rotated values, configurations the application reads at startup
- Access via `ssm:GetParameter` API call with IAM authentication

#### AWS Secrets Manager
- Stores and **automatically rotates** secrets (DB passwords, API keys)
- Native integration with RDS for automatic password rotation via Lambda
- **Cost:** ~$0.40/secret/month + $0.05 per 10,000 API calls
- Best for: DB passwords that need rotation, short-lived credentials
- Access via `secretsmanager:GetSecretValue` API call

#### Recommended allocation for InsightGen:
| Secret | Store |
|---|---|
| PostgreSQL master password | **Secrets Manager** (supports rotation) |
| DB connection string | **Parameter Store SecureString** |
| Encryption DEK (encrypted form) | **Parameter Store SecureString** |
| Application API keys | **Secrets Manager** |
| KMS Key ARN | **Parameter Store String** (not sensitive) |

---

### 3.2 Where Encryption and Decryption Happen

```
NEVER in PostgreSQL ──────────────────────────────────────────────────┐
                                                                       │
ALWAYS in the application layer (Lambda Python 3.12 function):        │
  1. Retrieve encrypted DEK from Parameter Store                       │
  2. Call KMS Decrypt API → plaintext DEK                             │
  3. Use plaintext DEK with AES-256-GCM to encrypt/decrypt the field  │
  4. Discard plaintext DEK from memory after the operation             │
                                                                       │
PostgreSQL only ever sees:  ciphertext + IV (stored as bytea/text)  ──┘
```

---

### 3.3 IAM Roles and Policies

#### EC2 Instance Role (for application server / Docker host)
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSSMRead",
      "Effect": "Allow",
      "Action": [
        "ssm:GetParameter",
        "ssm:GetParameters"
      ],
      "Resource": "arn:aws:ssm:REGION:ACCOUNT_ID:parameter/insightgen/*"
    },
    {
      "Sid": "AllowKMSDecrypt",
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt",
        "kms:DescribeKey"
      ],
      "Resource": "arn:aws:kms:REGION:ACCOUNT_ID:key/KEY_ID"
    }
  ]
}
```

#### Lambda Execution Role
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSecretsRead",
      "Effect": "Allow",
      "Action": "secretsmanager:GetSecretValue",
      "Resource": "arn:aws:secretsmanager:REGION:ACCOUNT_ID:secret:insightgen/*"
    },
    {
      "Sid": "AllowKMSDecryptOnly",
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt",
        "kms:GenerateDataKey"
      ],
      "Resource": "arn:aws:kms:REGION:ACCOUNT_ID:key/KEY_ID"
    },
    {
      "Sid": "AllowVPCAccess",
      "Effect": "Allow",
      "Action": [
        "ec2:CreateNetworkInterface",
        "ec2:DescribeNetworkInterfaces",
        "ec2:DeleteNetworkInterface"
      ],
      "Resource": "*"
    }
  ]
}
```

#### KMS Key Policy (controls who can use the CMK itself)
```json
{
  "Statement": [
    {
      "Sid": "AllowKeyAdministration",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT_ID:role/KeyAdminRole"
      },
      "Action": ["kms:Create*", "kms:Describe*", "kms:Enable*", "kms:List*",
                 "kms:Put*", "kms:Update*", "kms:Revoke*", "kms:Disable*",
                 "kms:Get*", "kms:Delete*", "kms:ScheduleKeyDeletion", "kms:CancelKeyDeletion"],
      "Resource": "*"
    },
    {
      "Sid": "AllowLambdaKeyUsage",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT_ID:role/insightgen-lambda-role"
      },
      "Action": ["kms:Decrypt", "kms:GenerateDataKey"],
      "Resource": "*"
    },
    {
      "Sid": "AllowEC2KeyUsage",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT_ID:role/insightgen-ec2-role"
      },
      "Action": ["kms:Decrypt"],
      "Resource": "*"
    }
  ]
}
```

---

### 3.4 How EC2 and Lambda Access Keys Securely

**EC2 (Docker host running PostgreSQL):**
- Attach an **IAM Instance Profile** to the EC2 instance
- The instance profile binds the IAM role to the EC2 instance
- The application running in Docker calls the **EC2 Instance Metadata Service (IMDS v2)** to get temporary STS credentials automatically
- No static credentials are stored anywhere on disk

**Lambda (Python 3.12 function):**
- Assign an **IAM Execution Role** to the Lambda function
- Lambda automatically injects temporary credentials via environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`)
- The AWS SDK (`boto3`) picks these up automatically — no manual credential handling needed
- Lambda in a VPC can connect to PostgreSQL on EC2 via private subnet

---

## 4. Architecture Diagram

```mermaid
flowchart TD
    subgraph USER["👤 User / Client"]
        C[Client Application]
    end

    subgraph LAMBDA["☁️ AWS Lambda (Python 3.12) — Private VPC Subnet"]
        L[Lambda Handler\nauthenticate_user\nencrypt_field\ndecrypt_field]
    end

    subgraph EC2["🖥️ AWS EC2 Instance (Private Subnet)"]
        subgraph DOCKER["Docker Containers"]
            PG[(PostgreSQL\nport 5432)]
            NG[Nginx\nReverse Proxy]
            KC[Keycloak\nAuth Server]
            KONG[Kong\nAPI Gateway]
        end
    end

    subgraph AWS_SERVICES["🔐 AWS Security Services"]
        KMS[AWS KMS\nCustomer Master Key\nKey Rotation Enabled]
        SSM[Parameter Store\nSecureString\nDB Config & Encrypted DEK]
        SM[Secrets Manager\nDB Password\nAuto-Rotation]
    end

    subgraph IAM["🛡️ IAM"]
        LR[Lambda Execution Role\nkms:Decrypt\nkms:GenerateDataKey\nsecretsmanager:GetSecretValue]
        ER[EC2 Instance Role\nkms:Decrypt\nssm:GetParameter]
    end

    subgraph CLOUDTRAIL["📋 Audit"]
        CT[CloudTrail\nAll KMS API Calls\nAll SSM Accesses]
    end

    %% User to Lambda
    C -->|HTTPS API Request| KONG
    KONG -->|Route /auth /data| L

    %% Lambda auth flow
    L -->|1. Assume role via IMDS| LR
    LR -->|2. GetSecretValue| SM
    SM -->|3. DB credentials| L
    L -->|4. GetParameter encrypted DEK| SSM
    SSM -->|5. Encrypted DEK| L
    L -->|6. kms:Decrypt DEK| KMS
    KMS -->|7. Plaintext DEK| L
    L -->|8. AES-256-GCM encrypt/decrypt field| L
    L -->|9. TLS connection + hashed/encrypted data| PG

    %% EC2 access to secrets
    EC2 -->|Startup: fetch DB config| SSM
    EC2 -->|Startup: fetch DB password| SM
    ER -.->|Bound to EC2 Instance Profile| EC2

    %% Audit
    KMS -->|Log all key operations| CT
    SSM -->|Log all parameter reads| CT
    SM -->|Log all secret reads| CT

    %% Styling
    classDef aws fill:#FF9900,stroke:#232F3E,color:#000
    classDef lambda fill:#FF9900,stroke:#232F3E,color:#000
    classDef ec2 fill:#2496ED,stroke:#1a6fa8,color:#fff
    classDef secure fill:#d4edda,stroke:#28a745,color:#000
    classDef audit fill:#fff3cd,stroke:#856404,color:#000

    class KMS,SSM,SM aws
    class L lambda
    class PG,NG,KC,KONG ec2
    class LR,ER secure
    class CT audit
```

### Data Flow Summary

| Step | Operation | Where |
|---|---|---|
| **Signup** | Hash password (Argon2id) | Lambda |
| **Signup** | Encrypt sensitive fields (AES-256-GCM) | Lambda |
| **Signup** | Store ciphertext + hash in PostgreSQL | Lambda → EC2 |
| **Login** | Retrieve hash from DB | Lambda → EC2 |
| **Login** | Verify password against hash | Lambda |
| **Read PII** | Retrieve ciphertext from DB | Lambda → EC2 |
| **Read PII** | Decrypt via KMS + DEK | Lambda |
| **Key retrieval** | GetParameter + kms:Decrypt | Lambda → SSM → KMS |

---

## 5. Implementation Guide

### 5.1 Prerequisites and Dependencies

```bash
# Python 3.12 dependencies (requirements.txt for Lambda layer)
bcrypt==4.1.2
argon2-cffi==23.1.0
cryptography==42.0.5
boto3==1.34.0
psycopg2-binary==2.9.9
```

---

### 5.2 PostgreSQL Schema Setup

Connect to your PostgreSQL container and run the following:

```sql
-- ============================================================
-- Enable required extensions
-- ============================================================
CREATE EXTENSION IF NOT EXISTS pgcrypto;  -- for gen_random_uuid()
CREATE EXTENSION IF NOT EXISTS pgaudit;   -- for audit logging

-- ============================================================
-- Create application database and user
-- ============================================================
CREATE DATABASE insightgen_db;
\c insightgen_db;

-- Create a least-privilege application user
-- This user can only read/write data, not alter schema
CREATE ROLE app_user WITH LOGIN PASSWORD 'REPLACE_WITH_SECRETS_MANAGER_VALUE';
GRANT CONNECT ON DATABASE insightgen_db TO app_user;

-- ============================================================
-- Users table
-- Stores hashed passwords and encrypted PII
-- ============================================================
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username        VARCHAR(50)  NOT NULL UNIQUE,
    email_hash      VARCHAR(64)  NOT NULL UNIQUE,  -- SHA-256 of normalised email for lookup
    email_encrypted TEXT         NOT NULL,          -- AES-256-GCM ciphertext (base64)
    email_iv        VARCHAR(32)  NOT NULL,          -- IV for email encryption (hex)
    password_hash   TEXT         NOT NULL,          -- Argon2id hash string
    created_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    is_active       BOOLEAN      NOT NULL DEFAULT TRUE
);

-- ============================================================
-- API keys table
-- Encrypted API keys that must be decryptable on read
-- ============================================================
CREATE TABLE api_keys (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    key_name        VARCHAR(100) NOT NULL,
    key_encrypted   TEXT NOT NULL,   -- AES-256-GCM ciphertext (base64)
    key_iv          VARCHAR(32) NOT NULL,
    key_prefix      VARCHAR(10) NOT NULL,  -- plaintext prefix for display, e.g. "sk_live_..."
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at      TIMESTAMPTZ,
    is_revoked      BOOLEAN NOT NULL DEFAULT FALSE
);

-- ============================================================
-- Sensitive profile data
-- PII that must be stored encrypted and read back
-- ============================================================
CREATE TABLE user_profiles (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL UNIQUE REFERENCES users(id) ON DELETE CASCADE,
    national_id_encrypted TEXT,       -- AES-256-GCM ciphertext
    national_id_iv      VARCHAR(32),
    phone_encrypted     TEXT,
    phone_iv            VARCHAR(32),
    full_name           VARCHAR(200), -- Non-sensitive, plaintext
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================================
-- Audit log (application-level, supplements pgaudit)
-- ============================================================
CREATE TABLE audit_log (
    id          BIGSERIAL PRIMARY KEY,
    user_id     UUID REFERENCES users(id),
    action      VARCHAR(50) NOT NULL,   -- e.g. 'LOGIN', 'READ_PROFILE', 'UPDATE_API_KEY'
    resource    VARCHAR(100),
    ip_address  INET,
    success     BOOLEAN NOT NULL,
    details     JSONB,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================================
-- Indexes
-- ============================================================
CREATE INDEX idx_users_username   ON users(username);
CREATE INDEX idx_users_email_hash ON users(email_hash);
CREATE INDEX idx_api_keys_user_id ON api_keys(user_id);
CREATE INDEX idx_audit_log_user   ON audit_log(user_id, created_at DESC);

-- ============================================================
-- Row-Level Security
-- app_user can only see rows it owns
-- (Extend with user_id column policies as needed)
-- ============================================================
ALTER TABLE api_keys ENABLE ROW LEVEL SECURITY;

-- Grant minimum necessary privileges to app_user
GRANT SELECT, INSERT, UPDATE, DELETE ON users         TO app_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON api_keys      TO app_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON user_profiles TO app_user;
GRANT INSERT ON audit_log TO app_user;
GRANT USAGE, SELECT ON SEQUENCE audit_log_id_seq TO app_user;
```

---

### 5.3 AWS KMS and Parameter Store Setup

```bash
# ============================================================
# 1. Create a Customer Managed Key in KMS
# ============================================================
aws kms create-key \
  --description "InsightGen data encryption key" \
  --key-usage ENCRYPT_DECRYPT \
  --key-spec SYMMETRIC_DEFAULT \
  --enable-key-rotation \
  --tags TagKey=Project,TagValue=InsightGen TagKey=Environment,TagValue=Production

# Note the KeyId and KeyArn from the output
export KMS_KEY_ID="<your-key-id>"
export KMS_KEY_ARN="<your-key-arn>"

# Create an alias for easier reference
aws kms create-alias \
  --alias-name alias/insightgen-data-key \
  --target-key-id $KMS_KEY_ID

# ============================================================
# 2. Generate a Data Encryption Key (DEK)
#    KMS generates a random AES-256 key, returns:
#    - Plaintext DEK (use once, then discard)
#    - Encrypted DEK (store this in Parameter Store)
# ============================================================
aws kms generate-data-key \
  --key-id alias/insightgen-data-key \
  --key-spec AES_256 \
  --query 'CiphertextBlob' \
  --output text > /tmp/encrypted_dek.b64

# ============================================================
# 3. Store the ENCRYPTED DEK in Parameter Store
#    Never store the plaintext DEK anywhere
# ============================================================
ENCRYPTED_DEK=$(cat /tmp/encrypted_dek.b64)

aws ssm put-parameter \
  --name "/insightgen/prod/encrypted-dek" \
  --value "$ENCRYPTED_DEK" \
  --type "SecureString" \
  --key-id "alias/insightgen-data-key" \
  --description "AES-256 DEK encrypted by KMS CMK" \
  --tags Key=Project,Value=InsightGen

# ============================================================
# 4. Store PostgreSQL connection info in Parameter Store
# ============================================================
aws ssm put-parameter \
  --name "/insightgen/prod/db-host" \
  --value "10.0.1.45" \
  --type "SecureString" \
  --key-id "alias/insightgen-data-key"

aws ssm put-parameter \
  --name "/insightgen/prod/db-name" \
  --value "insightgen_db" \
  --type "SecureString" \
  --key-id "alias/insightgen-data-key"

# ============================================================
# 5. Store PostgreSQL password in Secrets Manager
#    (supports automatic rotation)
# ============================================================
aws secretsmanager create-secret \
  --name "insightgen/prod/db-credentials" \
  --description "PostgreSQL credentials for InsightGen" \
  --kms-key-id "alias/insightgen-data-key" \
  --secret-string '{
    "username": "app_user",
    "password": "STRONG_RANDOM_PASSWORD_HERE",
    "host": "10.0.1.45",
    "port": 5432,
    "dbname": "insightgen_db"
  }'

# ============================================================
# Clean up plaintext DEK from local machine
# ============================================================
rm /tmp/encrypted_dek.b64
```

---

### 5.4 Python Cryptography Module (crypto.py)

This module handles all encryption and decryption. Deploy as a Lambda Layer.

```python
# crypto.py — Application-layer cryptography module
# Deploy as a Lambda Layer shared by all Lambda functions

import base64
import hashlib
import os
import secrets
import logging
from typing import Tuple, Optional

import boto3
import argon2
from argon2 import PasswordHasher
from argon2.exceptions import VerifyMismatchError, VerificationError, InvalidHashError
from cryptography.hazmat.primitives.ciphers.aead import AESGCM

logger = logging.getLogger(__name__)
logger.setLevel(logging.INFO)


# ============================================================
# KMS Client (reused across Lambda invocations via module scope)
# ============================================================
_kms_client = None
_ssm_client = None

def get_kms_client():
    global _kms_client
    if _kms_client is None:
        _kms_client = boto3.client('kms', region_name=os.environ.get('AWS_REGION', 'ap-southeast-1'))
    return _kms_client

def get_ssm_client():
    global _ssm_client
    if _ssm_client is None:
        _ssm_client = boto3.client('ssm', region_name=os.environ.get('AWS_REGION', 'ap-southeast-1'))
    return _ssm_client


# ============================================================
# DEK Management — Envelope Encryption Pattern
# ============================================================

_plaintext_dek_cache: Optional[bytes] = None  # Module-level cache (Lambda container lifetime)

def get_plaintext_dek() -> bytes:
    """
    Retrieve the plaintext Data Encryption Key by:
    1. Fetching the encrypted DEK from Parameter Store
    2. Calling KMS Decrypt to unwrap it

    The plaintext DEK is cached for the lifetime of the Lambda container
    to avoid redundant KMS API calls on every invocation.
    """
    global _plaintext_dek_cache
    if _plaintext_dek_cache is not None:
        return _plaintext_dek_cache

    # Step 1: Get encrypted DEK from Parameter Store
    ssm = get_ssm_client()
    param_name = os.environ.get('ENCRYPTED_DEK_PARAM', '/insightgen/prod/encrypted-dek')

    response = ssm.get_parameter(Name=param_name, WithDecryption=True)
    encrypted_dek_b64 = response['Parameter']['Value']
    encrypted_dek_bytes = base64.b64decode(encrypted_dek_b64)

    # Step 2: Decrypt DEK via KMS
    kms = get_kms_client()
    kms_response = kms.decrypt(
        CiphertextBlob=encrypted_dek_bytes,
        KeyId=os.environ.get('KMS_KEY_ARN', 'alias/insightgen-data-key')
    )

    plaintext_dek = kms_response['Plaintext']  # 32 bytes for AES-256
    _plaintext_dek_cache = plaintext_dek
    logger.info("DEK successfully retrieved and cached")
    return plaintext_dek


# ============================================================
# AES-256-GCM Field Encryption
# ============================================================

def encrypt_field(plaintext: str) -> Tuple[str, str]:
    """
    Encrypt a sensitive string field using AES-256-GCM.

    Returns:
        Tuple of (ciphertext_b64, iv_hex)
        - ciphertext_b64: base64-encoded ciphertext + auth tag (GCM appends 16-byte tag)
        - iv_hex: hex-encoded 12-byte IV (must be stored with the ciphertext)

    Security properties:
        - AES-256-GCM provides authenticated encryption (confidentiality + integrity)
        - A fresh random IV is generated per encryption — NEVER reuse an IV with the same key
        - The 16-byte GCM auth tag detects any tampering with the ciphertext
    """
    dek = get_plaintext_dek()
    iv = secrets.token_bytes(12)  # 96-bit IV — recommended for AES-GCM
    aesgcm = AESGCM(dek)
    ciphertext = aesgcm.encrypt(iv, plaintext.encode('utf-8'), None)
    return base64.b64encode(ciphertext).decode('utf-8'), iv.hex()


def decrypt_field(ciphertext_b64: str, iv_hex: str) -> str:
    """
    Decrypt a field encrypted by encrypt_field().

    Raises cryptography.exceptions.InvalidTag if the ciphertext
    has been tampered with or the wrong key is used.
    """
    dek = get_plaintext_dek()
    iv = bytes.fromhex(iv_hex)
    ciphertext = base64.b64decode(ciphertext_b64)
    aesgcm = AESGCM(dek)
    plaintext_bytes = aesgcm.decrypt(iv, ciphertext, None)
    return plaintext_bytes.decode('utf-8')


# ============================================================
# Password Hashing — Argon2id
# ============================================================

# Production Argon2id parameters
# time_cost=3, memory_cost=65536 (64MB), parallelism=2
# Adjust memory_cost based on available Lambda memory
_ph = PasswordHasher(
    time_cost=3,
    memory_cost=65536,
    parallelism=2,
    hash_len=32,
    salt_len=16,
    encoding='utf-8'
)


def hash_password(password: str) -> str:
    """
    Hash a password using Argon2id.

    The returned hash string includes all parameters and salt:
    $argon2id$v=19$m=65536,t=3,p=2$<salt>$<hash>

    This self-describing format means you never need to store
    the salt separately — it is embedded in the hash string.
    """
    if len(password) < 8:
        raise ValueError("Password must be at least 8 characters")
    return _ph.hash(password)


def verify_password(password: str, password_hash: str) -> bool:
    """
    Verify a password against its Argon2id hash.

    Returns True if the password matches, False otherwise.
    Never raises an exception for invalid passwords — always returns bool.

    Also performs automatic rehash detection: if the stored hash
    was created with weaker parameters, argon2-cffi flags it for upgrade.
    """
    try:
        _ph.verify(password_hash, password)
        return True
    except VerifyMismatchError:
        return False
    except (VerificationError, InvalidHashError) as e:
        logger.warning(f"Password verification error (not a mismatch): {e}")
        return False


def needs_rehash(password_hash: str) -> bool:
    """
    Check if a stored hash needs upgrading to current parameters.
    Call this after a successful login and update the hash if True.
    """
    return _ph.check_needs_rehash(password_hash)


# ============================================================
# Email normalisation and indexable hash
# ============================================================

def email_search_hash(email: str) -> str:
    """
    Create a deterministic SHA-256 hash of a normalised email address.
    This allows database lookups by email without decrypting stored values.

    The email is normalised (lowercased, stripped) before hashing to
    ensure consistent lookups regardless of input case.
    """
    normalised = email.strip().lower()
    return hashlib.sha256(normalised.encode('utf-8')).hexdigest()
```

---

### 5.5 AWS Lambda Function (Python 3.12)

```python
# lambda_handler.py — Main Lambda entry point
# Handles: user signup, login, and secure data retrieval

import json
import logging
import os
import boto3
import psycopg2
from psycopg2.extras import RealDictCursor
from typing import Any, Dict

from crypto import (
    hash_password, verify_password, needs_rehash,
    encrypt_field, decrypt_field, email_search_hash
)

logger = logging.getLogger(__name__)
logger.setLevel(logging.INFO)


# ============================================================
# Database Connection (with Secrets Manager)
# ============================================================

_db_conn = None

def get_db_connection():
    """
    Get a PostgreSQL connection using credentials from Secrets Manager.
    Connection is reused across warm Lambda invocations.
    """
    global _db_conn
    if _db_conn is not None and not _db_conn.closed:
        return _db_conn

    sm = boto3.client('secretsmanager', region_name=os.environ.get('AWS_REGION'))
    secret_arn = os.environ['DB_SECRET_ARN']

    response = sm.get_secret_value(SecretId=secret_arn)
    creds = json.loads(response['SecretString'])

    _db_conn = psycopg2.connect(
        host=creds['host'],
        port=creds['port'],
        dbname=creds['dbname'],
        user=creds['username'],
        password=creds['password'],
        sslmode='require',          # Enforce TLS — never allow plaintext connections
        sslrootcert='/var/task/certs/rds-ca-bundle.pem',  # CA cert for verification
        connect_timeout=5
    )
    _db_conn.autocommit = False
    logger.info("Database connection established")
    return _db_conn


# ============================================================
# Signup Handler
# ============================================================

def handle_signup(body: Dict[str, Any]) -> Dict[str, Any]:
    """
    Register a new user.
    - Hashes the password with Argon2id (irreversible)
    - Encrypts the email and national ID with AES-256-GCM (reversible)
    - Stores a SHA-256 email hash for searchable lookups
    """
    username = body.get('username', '').strip()
    password = body.get('password', '')
    email    = body.get('email', '').strip().lower()
    national_id = body.get('national_id', '')

    # Input validation
    if not all([username, password, email]):
        return {'statusCode': 400, 'body': 'username, password, email are required'}
    if len(username) < 3 or len(username) > 50:
        return {'statusCode': 400, 'body': 'username must be 3-50 characters'}

    conn = get_db_connection()
    try:
        # Hash password — one-way, cannot be reversed
        pwd_hash = hash_password(password)

        # Encrypt email — reversible, needed for communication
        email_enc, email_iv = encrypt_field(email)

        # Index hash for lookups without decrypting
        email_h = email_search_hash(email)

        with conn.cursor(cursor_factory=RealDictCursor) as cur:
            # Check uniqueness on the hash (plaintext never queried)
            cur.execute("SELECT id FROM users WHERE email_hash = %s OR username = %s",
                        (email_h, username))
            if cur.fetchone():
                return {'statusCode': 409, 'body': 'Username or email already exists'}

            # Insert user record
            cur.execute("""
                INSERT INTO users (username, email_hash, email_encrypted, email_iv, password_hash)
                VALUES (%s, %s, %s, %s, %s)
                RETURNING id
            """, (username, email_h, email_enc, email_iv, pwd_hash))
            user_id = cur.fetchone()['id']

            # Insert profile with encrypted PII if provided
            if national_id:
                nid_enc, nid_iv = encrypt_field(national_id)
                cur.execute("""
                    INSERT INTO user_profiles (user_id, national_id_encrypted, national_id_iv)
                    VALUES (%s, %s, %s)
                """, (str(user_id), nid_enc, nid_iv))

            # Audit log
            cur.execute("""
                INSERT INTO audit_log (user_id, action, success)
                VALUES (%s, 'SIGNUP', TRUE)
            """, (str(user_id),))

        conn.commit()
        logger.info(f"User created successfully: {username}")
        return {'statusCode': 201, 'body': json.dumps({'user_id': str(user_id)})}

    except Exception as e:
        conn.rollback()
        logger.error(f"Signup failed: {e}", exc_info=True)
        return {'statusCode': 500, 'body': 'Internal server error'}


# ============================================================
# Login Handler
# ============================================================

def handle_login(body: Dict[str, Any]) -> Dict[str, Any]:
    """
    Authenticate a user.
    - Retrieves stored Argon2id hash from database
    - Verifies submitted password against hash (timing-safe)
    - Performs automatic hash upgrade if parameters have changed
    """
    username = body.get('username', '').strip()
    password = body.get('password', '')

    if not username or not password:
        return {'statusCode': 400, 'body': 'username and password are required'}

    conn = get_db_connection()
    try:
        with conn.cursor(cursor_factory=RealDictCursor) as cur:
            cur.execute("""
                SELECT id, password_hash, is_active
                FROM users WHERE username = %s
            """, (username,))
            user = cur.fetchone()

            # Constant-time response regardless of whether user exists
            # (prevents user enumeration via timing differences)
            if user is None:
                # Perform a dummy hash verify to normalise timing
                verify_password('dummy_password', hash_password('dummy_reference'))
                return {'statusCode': 401, 'body': 'Invalid credentials'}

            if not user['is_active']:
                return {'statusCode': 403, 'body': 'Account disabled'}

            if not verify_password(password, user['password_hash']):
                cur.execute("""
                    INSERT INTO audit_log (user_id, action, success)
                    VALUES (%s, 'LOGIN', FALSE)
                """, (str(user['id']),))
                conn.commit()
                return {'statusCode': 401, 'body': 'Invalid credentials'}

            # Successful login
            # Check if hash needs upgrading (e.g., parameters were strengthened)
            if needs_rehash(user['password_hash']):
                new_hash = hash_password(password)
                cur.execute("UPDATE users SET password_hash = %s WHERE id = %s",
                            (new_hash, user['id']))
                logger.info(f"Password hash upgraded for user: {username}")

            cur.execute("""
                INSERT INTO audit_log (user_id, action, success)
                VALUES (%s, 'LOGIN', TRUE)
            """, (str(user['id']),))
            conn.commit()

            logger.info(f"Successful login: {username}")
            return {
                'statusCode': 200,
                'body': json.dumps({'message': 'Login successful', 'user_id': str(user['id'])})
            }

    except Exception as e:
        conn.rollback()
        logger.error(f"Login error: {e}", exc_info=True)
        return {'statusCode': 500, 'body': 'Internal server error'}


# ============================================================
# Read Sensitive Data Handler
# ============================================================

def handle_get_profile(body: Dict[str, Any]) -> Dict[str, Any]:
    """
    Retrieve and decrypt a user's sensitive profile data.
    Requires authenticated caller (user_id passed from authoriser).
    """
    user_id = body.get('user_id')
    if not user_id:
        return {'statusCode': 400, 'body': 'user_id is required'}

    conn = get_db_connection()
    try:
        with conn.cursor(cursor_factory=RealDictCursor) as cur:
            cur.execute("""
                SELECT u.username, u.email_encrypted, u.email_iv,
                       p.national_id_encrypted, p.national_id_iv, p.full_name
                FROM users u
                LEFT JOIN user_profiles p ON p.user_id = u.id
                WHERE u.id = %s AND u.is_active = TRUE
            """, (user_id,))
            row = cur.fetchone()

            if not row:
                return {'statusCode': 404, 'body': 'User not found'}

            # Decrypt only what is needed
            email = decrypt_field(row['email_encrypted'], row['email_iv'])

            national_id = None
            if row.get('national_id_encrypted'):
                national_id = decrypt_field(row['national_id_encrypted'], row['national_id_iv'])

            # Audit every sensitive data access
            cur.execute("""
                INSERT INTO audit_log (user_id, action, resource, success)
                VALUES (%s, 'READ_PROFILE', 'user_profiles', TRUE)
            """, (user_id,))
            conn.commit()

        return {
            'statusCode': 200,
            'body': json.dumps({
                'username': row['username'],
                'email': email,
                'full_name': row.get('full_name'),
                'national_id': national_id  # None if not set
            })
        }

    except Exception as e:
        conn.rollback()
        logger.error(f"Profile read error: {e}", exc_info=True)
        return {'statusCode': 500, 'body': 'Internal server error'}


# ============================================================
# Lambda Entry Point
# ============================================================

def lambda_handler(event: Dict[str, Any], context) -> Dict[str, Any]:
    """
    Main Lambda handler. Routes by action field in request body.
    In production, pair with an API Gateway + Cognito/Keycloak authoriser.
    """
    try:
        body = json.loads(event.get('body', '{}')) if isinstance(event.get('body'), str) else event
        action = body.get('action', '')

        if action == 'signup':
            return handle_signup(body)
        elif action == 'login':
            return handle_login(body)
        elif action == 'get_profile':
            return handle_get_profile(body)
        else:
            return {'statusCode': 400, 'body': f'Unknown action: {action}'}

    except json.JSONDecodeError:
        return {'statusCode': 400, 'body': 'Invalid JSON'}
    except Exception as e:
        logger.error(f"Unhandled error: {e}", exc_info=True)
        return {'statusCode': 500, 'body': 'Internal server error'}
```

---

### 5.6 Insert Test Data (SQL)

After deploying the Lambda, use these test SQL inserts to pre-populate data for validation. In practice, always insert via the Lambda (which hashes/encrypts before writing). These direct inserts use pre-computed values for testing only.

```sql
-- ============================================================
-- Test data — pre-computed values for validation
-- Run these AFTER running the Python test script (section 6.1)
-- to get actual hashes/ciphertexts to insert
-- ============================================================

-- These are placeholders — replace with actual output from test_crypto.py

INSERT INTO users (
    id,
    username,
    email_hash,
    email_encrypted,
    email_iv,
    password_hash
) VALUES (
    gen_random_uuid(),
    'alice_test',
    '<SHA256_OF_alice@example.com>',
    '<AES_GCM_CIPHERTEXT_B64>',
    '<IV_HEX>',
    '$argon2id$v=19$m=65536,t=3,p=2$<salt>$<hash>'
);

-- Verify the insert
SELECT id, username, email_hash,
       LEFT(email_encrypted, 20) || '...' AS email_enc_preview,
       LEFT(password_hash, 30) || '...'   AS hash_preview
FROM users WHERE username = 'alice_test';
```

---

## 6. Testing & Validation

### 6.1 Test Cryptography Functions Locally

```python
# test_crypto_local.py
# Run locally before deploying to Lambda to validate all crypto operations
# Use a mock DEK (not from KMS) for unit testing

import os
import sys
import base64
import secrets
from cryptography.hazmat.primitives.ciphers.aead import AESGCM

# ============================================================
# Mock DEK for local testing — never use in production
# ============================================================
MOCK_DEK = secrets.token_bytes(32)  # Random 256-bit key

def mock_encrypt(plaintext: str):
    iv = secrets.token_bytes(12)
    aesgcm = AESGCM(MOCK_DEK)
    ciphertext = aesgcm.encrypt(iv, plaintext.encode(), None)
    return base64.b64encode(ciphertext).decode(), iv.hex()

def mock_decrypt(ciphertext_b64: str, iv_hex: str) -> str:
    iv = bytes.fromhex(iv_hex)
    ciphertext = base64.b64decode(ciphertext_b64)
    aesgcm = AESGCM(MOCK_DEK)
    return aesgcm.decrypt(iv, ciphertext, None).decode()

# ============================================================
# Test 1: Password Hashing
# ============================================================
print("=" * 60)
print("TEST 1: Password Hashing (Argon2id)")
print("=" * 60)

from argon2 import PasswordHasher
from argon2.exceptions import VerifyMismatchError

ph = PasswordHasher(time_cost=3, memory_cost=65536, parallelism=2)

password = "MyS3cur3P@ssword!"
hash1 = ph.hash(password)
hash2 = ph.hash(password)

print(f"Password:    {password}")
print(f"Hash 1:      {hash1}")
print(f"Hash 2:      {hash2}")
print(f"Hashes differ (salt is random): {hash1 != hash2}")

# Verify correct password
assert ph.verify(hash1, password) == True
print(f"Verify correct password: PASS ✅")

# Verify wrong password
try:
    ph.verify(hash1, "wrongpassword")
    print("Verify wrong password:   FAIL ❌ (should have raised)")
except VerifyMismatchError:
    print(f"Verify wrong password:   PASS ✅ (correctly rejected)")

# ============================================================
# Test 2: AES-256-GCM Encryption Round-Trip
# ============================================================
print("\n" + "=" * 60)
print("TEST 2: AES-256-GCM Encryption Round-Trip")
print("=" * 60)

test_values = [
    "alice@example.com",
    "ID-1234567890",
    "sk_live_abcdef1234567890abcdef1234567890",
    "Unicode test: こんにちは 🔐"
]

for val in test_values:
    enc, iv = mock_encrypt(val)
    dec = mock_decrypt(enc, iv)
    status = "PASS ✅" if dec == val else "FAIL ❌"
    print(f"{status} | Original: {val[:30]:<30} | Ciphertext: {enc[:20]}...")

# ============================================================
# Test 3: IV Uniqueness (same plaintext → different ciphertext)
# ============================================================
print("\n" + "=" * 60)
print("TEST 3: IV Uniqueness (randomness verification)")
print("=" * 60)

enc1, iv1 = mock_encrypt("same_value")
enc2, iv2 = mock_encrypt("same_value")
print(f"Ciphertext 1: {enc1[:40]}")
print(f"Ciphertext 2: {enc2[:40]}")
print(f"IVs differ:          {'PASS ✅' if iv1 != iv2 else 'FAIL ❌'}")
print(f"Ciphertexts differ:  {'PASS ✅' if enc1 != enc2 else 'FAIL ❌'}")

# ============================================================
# Test 4: Tamper Detection (GCM auth tag)
# ============================================================
print("\n" + "=" * 60)
print("TEST 4: Tamper Detection (AES-GCM authentication tag)")
print("=" * 60)

from cryptography.exceptions import InvalidTag

original = "sensitive-data"
enc, iv = mock_encrypt(original)
tampered = base64.b64decode(enc)
tampered = tampered[:-1] + bytes([tampered[-1] ^ 0xFF])  # Flip bits in last byte
tampered_b64 = base64.b64encode(tampered).decode()

try:
    mock_decrypt(tampered_b64, iv)
    print("Tamper detection: FAIL ❌ (should have raised InvalidTag)")
except InvalidTag:
    print("Tamper detection: PASS ✅ (InvalidTag raised on tampered ciphertext)")

# ============================================================
# Test 5: Email Search Hash
# ============================================================
print("\n" + "=" * 60)
print("TEST 5: Email Search Hash (deterministic)")
print("=" * 60)

import hashlib

def email_hash(email):
    return hashlib.sha256(email.strip().lower().encode()).hexdigest()

h1 = email_hash("Alice@Example.COM")
h2 = email_hash("alice@example.com")
h3 = email_hash("  alice@example.com  ")
print(f"Hash consistency (case/whitespace): {'PASS ✅' if h1 == h2 == h3 else 'FAIL ❌'}")
print(f"Hash value: {h1}")

print("\n" + "=" * 60)
print("All local crypto tests complete.")
print("=" * 60)
```

---

### 6.2 Test IAM Restrictions

```bash
# ============================================================
# Test 1: Authorised access — Lambda role can decrypt
# ============================================================
# Invoke the Lambda as normal (it uses its execution role)
aws lambda invoke \
  --function-name insightgen-auth \
  --payload '{"action": "signup", "username": "testuser", "password": "Test@12345", "email": "test@example.com"}' \
  --cli-binary-format raw-in-base64-out \
  response.json

cat response.json
# Expected: {"statusCode": 201, "body": "{\"user_id\": \"...\"}"}

# ============================================================
# Test 2: Unauthorised access — explicit deny on KMS
# ============================================================
# Temporarily test what happens when Lambda role loses kms:Decrypt permission
# Add this to the KMS key policy to deny a test role:

aws kms put-key-policy \
  --key-id alias/insightgen-data-key \
  --policy-name default \
  --policy '{
    "Statement": [
      {
        "Sid": "ExplicitDenyTestRole",
        "Effect": "Deny",
        "Principal": {"AWS": "arn:aws:iam::ACCOUNT_ID:role/test-unauthorized-role"},
        "Action": "kms:*",
        "Resource": "*"
      }
    ]
  }'

# Invoke Lambda assuming the unauthorised role
# Expected: AccessDeniedException from KMS → Lambda returns 500

# ============================================================
# Test 3: Verify CloudTrail captures KMS usage
# ============================================================
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=Decrypt \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --query 'Events[].{Time:EventTime, User:Username, KeyId:Resources[0].ResourceName}'
```

---

### 6.3 End-to-End Validation Checklist

```bash
# ============================================================
# 1. Verify passwords are NEVER stored as plaintext
# ============================================================
# Connect directly to PostgreSQL and inspect the password_hash column
psql -h localhost -U app_user -d insightgen_db -c \
  "SELECT username, LEFT(password_hash, 15) AS hash_prefix FROM users LIMIT 5;"

# Expected output — hash prefix should always start with $argon2id$
#  username  |  hash_prefix
# -----------+-----------------
#  alice     | $argon2id$v=19$
#  bob       | $argon2id$v=19$

# ============================================================
# 2. Verify sensitive fields are stored as opaque ciphertext
# ============================================================
psql -h localhost -U app_user -d insightgen_db -c \
  "SELECT username, LEFT(email_encrypted, 40) AS email_preview FROM users LIMIT 5;"

# Expected: random-looking base64 strings, never recognisable as emails

# ============================================================
# 3. Verify different IVs per row (no IV reuse)
# ============================================================
psql -h localhost -U app_user -d insightgen_db -c \
  "SELECT COUNT(*) AS total_rows, COUNT(DISTINCT email_iv) AS unique_ivs FROM users;"

# Expected: total_rows == unique_ivs (every row has a unique IV)

# ============================================================
# 4. Verify TLS is enforced on the DB connection
# ============================================================
psql -h localhost -U app_user -d insightgen_db \
  -c "SELECT pg_ssl.ssl, pg_ssl.version FROM pg_stat_ssl JOIN pg_stat_activity
      ON pg_stat_ssl.pid = pg_stat_activity.pid
      WHERE pg_stat_activity.usename = 'app_user';"

# Expected: ssl=t, version=TLSv1.3 (or TLSv1.2 minimum)

# ============================================================
# 5. Verify decrypt round-trip produces original value
# ============================================================
# Call the get_profile Lambda action and confirm email decrypts correctly
aws lambda invoke \
  --function-name insightgen-auth \
  --payload '{"action": "get_profile", "user_id": "<USER_ID_FROM_SIGNUP>"}' \
  --cli-binary-format raw-in-base64-out \
  profile_response.json

cat profile_response.json
# Expected: {"statusCode": 200, "body": "{\"email\": \"alice@example.com\", ...}"}
```

---

### 6.4 Expected Outputs Summary

| Test | Expected Result | Failure Indicator |
|---|---|---|
| Password hash format | Starts with `$argon2id$v=19$` | Plaintext, bcrypt prefix, or empty |
| Two hashes of same password | Different strings | Same string (missing salt) |
| Password verify (correct) | `True` | `False` (broken verify logic) |
| Password verify (wrong) | `False` / `VerifyMismatchError` | `True` (catastrophic) |
| Email column in DB | Base64 ciphertext (~60-80 chars) | Recognisable email address |
| IV uniqueness | All IVs distinct | Duplicate IVs (IV reuse — catastrophic) |
| Tampered ciphertext decrypt | `InvalidTag` exception | Successful decryption (no integrity) |
| Unauthorised KMS access | `AccessDeniedException` | Successful decryption (policy misconfigured) |
| TLS on DB connection | `ssl=t`, `version=TLSv1.3` | `ssl=f` (plaintext connection) |

---

## Appendix: Security Hardening Checklist

- [ ] KMS CMK automatic rotation enabled (annual)
- [ ] CloudTrail enabled in all regions with log file integrity validation
- [ ] Lambda function in private VPC subnet (no direct internet access)
- [ ] PostgreSQL port (5432) not exposed to internet (Security Group: allow from Lambda SG only)
- [ ] EBS volume encrypted with CMK (not default AWS key)
- [ ] Parameter Store values encrypted with CMK (not default `aws/ssm` key)
- [ ] DB password stored in Secrets Manager with rotation Lambda configured
- [ ] `sslmode=require` or `verify-full` on all PostgreSQL connections
- [ ] `pg_hba.conf` set to `hostssl` only (reject non-TLS connections at the DB level)
- [ ] Lambda execution role follows least-privilege (`kms:Decrypt` only, not `kms:*`)
- [ ] No hardcoded credentials anywhere in code or environment variables
- [ ] `pgaudit` extension enabled and logging to CloudWatch
- [ ] Docker container running PostgreSQL has no `--privileged` flag
- [ ] `POSTGRES_PASSWORD` in Docker env sourced from Secrets Manager at runtime, not embedded in compose file
- [ ] Argon2id parameters reviewed and adjusted annually as hardware improves
