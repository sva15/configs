# Secrets Management — Cloud-Native Implementation
**Project:** InsightGen  
**Approach:** AWS SSM Parameter Store + GCP Secret Manager  
**Author:** fitindevops  
**Status:** Implementation guide

---

## Overview

This document covers implementing cloud-native secret retrieval for PostgreSQL credentials across AWS and GCP. The same Python application code is deployed to both clouds. A single environment variable `CLOUD_PROVIDER` switches which backend is used. The actual secret retrieval logic is isolated in one file — `secrets_client.py` — so all business logic in the Lambda / Cloud Function remains identical.

```
CLOUD_PROVIDER=AWS  →  reads from AWS SSM Parameter Store (existing setup)
CLOUD_PROVIDER=GCP  →  reads from GCP Secret Manager (new setup)
```

---

## Architecture

```
AWS Lambda                          GCP Cloud Function
    │                                       │
    │ CLOUD_PROVIDER=AWS                    │ CLOUD_PROVIDER=GCP
    │                                       │
    ▼                                       ▼
secrets_client.py  ←── same file ──▶  secrets_client.py
    │                                       │
    ▼                                       ▼
AWS SSM Parameter Store             GCP Secret Manager
/insightgen/postgres/*              insightgen-postgres-*
    │                                       │
    ▼                                       ▼
KMS (custom key, existing)          Cloud KMS (new CMEK key)
    │                                       │
    ▼                                       ▼
PostgreSQL container                PostgreSQL container
```

---

## Section 1 — AWS Setup (Already Done)

Your AWS setup is complete from previous work. For reference, the existing configuration is:

**SSM Parameters (already created):**
```
/insightgen/postgres/host
/insightgen/postgres/port
/insightgen/postgres/dbname
/insightgen/postgres/username
/insightgen/postgres/password
```

**KMS Key Policy (already hardened):** The existing policy allows:
- Root account for emergency recovery
- Approved SSO user sessions for manual access
- Lambda execution role (`HCL-User-Role-InsightGen-Lambda`) for runtime decryption

**Lambda IAM Role:** Already has `ssm:GetParameter`, `ssm:GetParametersByPath`, and `kms:Decrypt` scoped to the insightgen KMS key.

No changes needed on the AWS side.

---

## Section 2 — GCP Setup

### Step 1 — Identify Your GCP Access

Based on your GCP IAM roles:

| Principal | Role | What it means |
|---|---|---|
| `app-svc-ifrs-fs` service account | `Secret Manager Secret Accessor` | Cloud Functions can read secrets at runtime |
| `app-svc-ifrs-fs` service account | `Cloud KMS CryptoKey Decrypter` | Can decrypt CMEK-encrypted secrets |
| `GCP-ifrs-user` group (you) | `Secret Manager Admin` | You can create and manage secrets |
| `GCP-ifrs-user` group (you) | `Cloud KMS Admin` | You can create and manage KMS keys |

You do not need to create a new service account or request additional permissions. Everything required is already granted.

---

### Step 2 — Set Environment Variables

Run these in your terminal before running any gcloud commands:

```bash
export PROJECT_ID="your-actual-gcp-project-id"
export REGION="us-central1"   # Replace with your actual GCP region
export SA_EMAIL="app-svc-ifrs-fs@${PROJECT_ID}.iam.gserviceaccount.com"
```

Verify you are authenticated as your group user:
```bash
gcloud auth list
gcloud config set project $PROJECT_ID
```

---

### Step 3 — Create Cloud KMS Key for Secret Encryption

This is the GCP equivalent of your AWS custom KMS key. Secrets will be encrypted with this key (CMEK) instead of Google's default key.

```bash
# Create a key ring — logical container for keys
gcloud kms keyrings create insightgen-keyring \
  --location $REGION \
  --project $PROJECT_ID

# Create the encryption key
gcloud kms keys create insightgen-secrets-key \
  --location $REGION \
  --keyring insightgen-keyring \
  --purpose encryption \
  --project $PROJECT_ID

# Note your full key resource name for use in next steps
export KMS_KEY="projects/${PROJECT_ID}/locations/${REGION}/keyRings/insightgen-keyring/cryptoKeys/insightgen-secrets-key"
echo "KMS Key: $KMS_KEY"
```

Grant the Secret Manager service agent permission to use this key. Secret Manager needs this to encrypt and decrypt secret values on your behalf:

```bash
# Get the Secret Manager service agent email
PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format='value(projectNumber)')
SM_SA="service-${PROJECT_NUMBER}@gcp-sa-secretmanager.iam.gserviceaccount.com"

echo "Secret Manager service agent: $SM_SA"

# Grant it permission to use the KMS key
gcloud kms keys add-iam-policy-binding insightgen-secrets-key \
  --location $REGION \
  --keyring insightgen-keyring \
  --member "serviceAccount:${SM_SA}" \
  --role "roles/cloudkms.cryptoKeyEncrypterDecrypter" \
  --project $PROJECT_ID

echo "Secret Manager service agent granted KMS access"
```

Enable KMS key rotation (mirrors what you did on AWS):
```bash
gcloud kms keys update insightgen-secrets-key \
  --location $REGION \
  --keyring insightgen-keyring \
  --rotation-period 365d \
  --next-rotation-time $(date -d "+365 days" --iso-8601) \
  --project $PROJECT_ID

echo "Annual KMS key rotation enabled"
```

---

### Step 4 — Create PostgreSQL Secrets

Create each secret using your CMEK key. Naming convention mirrors your AWS path structure — `/insightgen/postgres/{key}` becomes `insightgen-postgres-{key}` in GCP.

```bash
# Create secret definitions
for SECRET_NAME in host port dbname username password; do
  gcloud secrets create insightgen-postgres-${SECRET_NAME} \
    --project $PROJECT_ID \
    --replication-policy user-managed \
    --locations $REGION \
    --kms-key-name $KMS_KEY
  echo "Created secret: insightgen-postgres-${SECRET_NAME}"
done
```

Add the actual values. Replace placeholder values with your real GCP PostgreSQL credentials:

```bash
# Host — IP or hostname of your GCP PostgreSQL container/instance
echo -n "YOUR_GCP_POSTGRES_HOST" | \
  gcloud secrets versions add insightgen-postgres-host \
  --data-file=- --project $PROJECT_ID

# Port
echo -n "5432" | \
  gcloud secrets versions add insightgen-postgres-port \
  --data-file=- --project $PROJECT_ID

# Database name
echo -n "insightgen" | \
  gcloud secrets versions add insightgen-postgres-dbname \
  --data-file=- --project $PROJECT_ID

# Username
echo -n "insightgen_app" | \
  gcloud secrets versions add insightgen-postgres-username \
  --data-file=- --project $PROJECT_ID

# Password — use -n to avoid trailing newline
echo -n "YOUR_ACTUAL_PASSWORD" | \
  gcloud secrets versions add insightgen-postgres-password \
  --data-file=- --project $PROJECT_ID
```

Verify all secrets were created with active versions:
```bash
gcloud secrets list \
  --project $PROJECT_ID \
  --filter="name:insightgen-postgres" \
  --format="table(name, createTime)"
```

---

### Step 5 — Verify Service Account Access

Confirm the service account that runs your Cloud Functions can read these secrets:

```bash
# Check existing IAM binding at project level
gcloud projects get-iam-policy $PROJECT_ID \
  --flatten="bindings[].members" \
  --filter="bindings.members:${SA_EMAIL} AND bindings.role:secretmanager" \
  --format="table(bindings.role)"
```

You should see `roles/secretmanager.secretAccessor`. If it shows at the project level, the service account can read all secrets in the project.

**Optional — Scope access to only insightgen secrets (tighter security):**

If you want the service account to access only the insightgen PostgreSQL secrets (not all secrets in the project), remove the project-wide binding and add per-secret bindings:

```bash
# Step A: Remove project-wide accessor (only if it was granted at project level)
gcloud projects remove-iam-policy-binding $PROJECT_ID \
  --member "serviceAccount:${SA_EMAIL}" \
  --role "roles/secretmanager.secretAccessor"

# Step B: Grant on individual secrets only
for SECRET_NAME in host port dbname username password; do
  gcloud secrets add-iam-policy-binding insightgen-postgres-${SECRET_NAME} \
    --project $PROJECT_ID \
    --member "serviceAccount:${SA_EMAIL}" \
    --role "roles/secretmanager.secretAccessor"
  echo "Secret-level access granted: insightgen-postgres-${SECRET_NAME}"
done
```

---

### Step 6 — Enable Secret Manager Audit Logs

This is the GCP equivalent of CloudTrail — every secret access is logged:

```bash
# Enable data access audit logs for Secret Manager
# Run this gcloud command to configure audit logging
cat > /tmp/audit-policy.yaml << 'EOF'
auditConfigs:
- auditLogConfigs:
  - logType: DATA_READ
  - logType: DATA_WRITE
  service: secretmanager.googleapis.com
EOF

gcloud projects set-iam-policy $PROJECT_ID /tmp/audit-policy.yaml
echo "Secret Manager audit logging enabled — all reads and writes are now logged"
```

---

## Section 3 — Python Implementation

### File: `secrets_client.py`

Drop this single file into both your Lambda package and your GCP Cloud Function package. It is the only file with cloud-specific code. Everything else in your application stays identical.

```python
"""
secrets_client.py

Cloud-agnostic secret retrieval for InsightGen PostgreSQL credentials.
Set CLOUD_PROVIDER=AWS or CLOUD_PROVIDER=GCP in environment variables.

AWS:  reads from SSM Parameter Store under /insightgen/postgres/
GCP:  reads from Secret Manager under insightgen-postgres-{key}

Secrets are cached after first fetch — one network call per cold start.
"""

import os
import logging
from functools import lru_cache

logger = logging.getLogger(__name__)

CLOUD_PROVIDER = os.environ.get('CLOUD_PROVIDER', 'AWS').upper()
REQUIRED_KEYS = {'host', 'port', 'dbname', 'username', 'password'}


# ─── AWS SSM Implementation ───────────────────────────────────────────────────

def _fetch_from_aws() -> dict:
    """
    Fetch all PostgreSQL credentials from AWS SSM Parameter Store.
    Uses GetParametersByPath for a single API call covering all keys.
    KMS decryption is handled automatically via the Lambda execution role.
    """
    import boto3
    from botocore.exceptions import ClientError

    region = os.environ.get('AWS_REGION', 'us-east-1')
    ssm = boto3.client('ssm', region_name=region)
    prefix = '/insightgen/postgres/'

    try:
        response = ssm.get_parameters_by_path(
            Path=prefix,
            WithDecryption=True,
            Recursive=False
        )
    except ClientError as e:
        code = e.response['Error']['Code']
        logger.error(f"SSM GetParametersByPath failed: {code} — {e}")
        raise RuntimeError(f"Failed to fetch secrets from AWS SSM: {code}") from e

    if not response.get('Parameters'):
        raise RuntimeError(
            f"No parameters found under {prefix}. "
            f"Verify SSM parameters exist and Lambda role has ssm:GetParametersByPath."
        )

    secrets = {
        p['Name'].replace(prefix, ''): p['Value']
        for p in response['Parameters']
    }

    logger.info(f"Fetched {len(secrets)} parameters from AWS SSM ({region})")
    return secrets


# ─── GCP Secret Manager Implementation ───────────────────────────────────────

def _fetch_from_gcp() -> dict:
    """
    Fetch all PostgreSQL credentials from GCP Secret Manager.
    Uses the service account attached to the Cloud Function — no token needed.
    Fetches each secret individually (GCP has no batch-by-prefix API).
    """
    from google.cloud import secretmanager
    from google.api_core.exceptions import GoogleAPIError, NotFound, PermissionDenied

    project_id = os.environ.get('GCP_PROJECT_ID')
    if not project_id:
        raise ValueError(
            "GCP_PROJECT_ID environment variable is required when CLOUD_PROVIDER=GCP"
        )

    client = secretmanager.SecretManagerServiceClient()
    secrets = {}

    for key in REQUIRED_KEYS:
        secret_name = (
            f"projects/{project_id}/secrets/"
            f"insightgen-postgres-{key}/versions/latest"
        )
        try:
            response = client.access_secret_version(request={"name": secret_name})
            secrets[key] = response.payload.data.decode("UTF-8")
        except NotFound:
            raise RuntimeError(
                f"Secret not found: insightgen-postgres-{key}. "
                f"Verify the secret exists in project {project_id}."
            )
        except PermissionDenied:
            raise RuntimeError(
                f"Permission denied reading insightgen-postgres-{key}. "
                f"Verify service account has roles/secretmanager.secretAccessor."
            )
        except GoogleAPIError as e:
            logger.error(f"GCP Secret Manager error for {key}: {e}")
            raise RuntimeError(f"Failed to fetch secret insightgen-postgres-{key}") from e

    logger.info(f"Fetched {len(secrets)} secrets from GCP Secret Manager (project: {project_id})")
    return secrets


# ─── Unified Public API ───────────────────────────────────────────────────────

@lru_cache(maxsize=1)
def get_postgres_config() -> dict:
    """
    Fetch all PostgreSQL connection config from the appropriate cloud secrets store.

    Cached with lru_cache — executes once per Lambda or Cloud Function cold start.
    All subsequent calls within the same instance return the cached result
    with zero network overhead.

    Returns:
        dict with keys: host, port, dbname, username, password

    Raises:
        ValueError: if CLOUD_PROVIDER is not AWS or GCP
        RuntimeError: if secrets cannot be fetched
        KeyError: if any required key is missing from the secret store
    """
    logger.info(f"Fetching PostgreSQL config from {CLOUD_PROVIDER} secrets store")

    if CLOUD_PROVIDER == 'AWS':
        secrets = _fetch_from_aws()
    elif CLOUD_PROVIDER == 'GCP':
        secrets = _fetch_from_gcp()
    else:
        raise ValueError(
            f"Unsupported CLOUD_PROVIDER: '{CLOUD_PROVIDER}'. "
            f"Must be 'AWS' or 'GCP'."
        )

    # Validate all required keys are present
    missing = REQUIRED_KEYS - set(secrets.keys())
    if missing:
        raise KeyError(
            f"Missing required PostgreSQL config keys: {missing}. "
            f"Check your {CLOUD_PROVIDER} secret store."
        )

    logger.info(f"PostgreSQL config loaded successfully from {CLOUD_PROVIDER}")
    return secrets


def get_postgres_dsn() -> str:
    """
    Returns a libpq-compatible DSN string for psycopg2.
    SSL is enforced — matches TLS configuration on the PostgreSQL container.

    Example return value:
        host=10.x.x.x port=5432 dbname=insightgen user=insightgen_app
        password=xxx sslmode=require
    """
    cfg = get_postgres_config()
    return (
        f"host={cfg['host']} "
        f"port={cfg['port']} "
        f"dbname={cfg['dbname']} "
        f"user={cfg['username']} "
        f"password={cfg['password']} "
        f"sslmode=require"
    )


def get_postgres_url() -> str:
    """
    Returns a SQLAlchemy-compatible PostgreSQL URL.
    Use this if your application uses SQLAlchemy instead of psycopg2 directly.
    """
    cfg = get_postgres_config()
    return (
        f"postgresql+psycopg2://{cfg['username']}:{cfg['password']}"
        f"@{cfg['host']}:{cfg['port']}/{cfg['dbname']}"
        f"?sslmode=require"
    )
```

---

### File: `main.py` (Lambda handler / Cloud Function entry point)

This is your application code. It is identical on both clouds — no cloud-specific logic here:

```python
"""
main.py — Lambda handler and GCP Cloud Function entry point.
Cloud-specific logic lives entirely in secrets_client.py.
This file is identical on both AWS and GCP.
"""

import json
import logging
import psycopg2
from secrets_client import get_postgres_dsn, get_postgres_config

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Module-level connection — persists across warm invocations
_db_conn = None


def _get_db_connection():
    """
    Return a live PostgreSQL connection.
    Reuses the existing connection across warm Lambda/Cloud Function invocations.
    Reconnects if the connection was closed.
    """
    global _db_conn
    if _db_conn is None or _db_conn.closed != 0:
        logger.info("Creating new PostgreSQL connection")
        dsn = get_postgres_dsn()
        _db_conn = psycopg2.connect(dsn)
        _db_conn.autocommit = True
    return _db_conn


# ─── AWS Lambda entry point ───────────────────────────────────────────────────

def lambda_handler(event, context):
    """AWS Lambda handler — called by API Gateway / ALB / direct invocation."""
    try:
        conn = _get_db_connection()
        with conn.cursor() as cur:
            cur.execute("SELECT version(), current_database(), current_user")
            row = cur.fetchone()

        return {
            'statusCode': 200,
            'headers': {'Content-Type': 'application/json'},
            'body': json.dumps({
                'status': 'connected',
                'cloud': 'AWS',
                'db_version': row[0],
                'database': row[1],
                'user': row[2]
            })
        }
    except Exception as e:
        logger.error(f"Lambda handler error: {e}")
        return {
            'statusCode': 500,
            'body': json.dumps({'error': str(e)})
        }


# ─── GCP Cloud Function entry point ──────────────────────────────────────────

def cloud_function_handler(request):
    """GCP Cloud Function HTTP handler — called by Cloud Functions runtime."""
    import flask
    try:
        conn = _get_db_connection()
        with conn.cursor() as cur:
            cur.execute("SELECT version(), current_database(), current_user")
            row = cur.fetchone()

        return flask.Response(
            json.dumps({
                'status': 'connected',
                'cloud': 'GCP',
                'db_version': row[0],
                'database': row[1],
                'user': row[2]
            }),
            status=200,
            mimetype='application/json'
        )
    except Exception as e:
        logger.error(f"Cloud Function handler error: {e}")
        return flask.Response(
            json.dumps({'error': str(e)}),
            status=500,
            mimetype='application/json'
        )
```

---

### File: `requirements.txt`

Two separate requirements files — one per deployment target. Do not install GCP libraries in the AWS Lambda package and vice versa — it bloats the deployment package unnecessarily.

**`requirements-aws.txt`** (for Lambda layer):
```
psycopg2-binary==2.9.9
boto3==1.34.0
```

**`requirements-gcp.txt`** (for Cloud Function package):
```
psycopg2-binary==2.9.9
google-cloud-secret-manager==2.20.0
flask==3.0.0
functions-framework==3.5.0
```

---

## Section 4 — Deployment

### Deploying to AWS Lambda

Environment variables to set on the Lambda function:

```bash
FUNCTION_NAME="dev-api-configurator"   # Replace with your function name

aws lambda update-function-configuration \
  --function-name $FUNCTION_NAME \
  --environment 'Variables={
    CLOUD_PROVIDER=AWS,
    APP_ENV=dev,
    APP_VERSION=mvp3
  }'
```

The Lambda execution role already has the required SSM and KMS permissions from your existing setup. No IAM changes needed.

---

### Deploying to GCP Cloud Function

```bash
# Navigate to your function source directory
cd your-function-directory

# Deploy with the GCP-specific requirements
gcloud functions deploy insightgen-api-configurator \
  --project $PROJECT_ID \
  --region $REGION \
  --runtime python312 \
  --trigger-http \
  --allow-unauthenticated \
  --entry-point cloud_function_handler \
  --service-account $SA_EMAIL \
  --set-env-vars \
    CLOUD_PROVIDER=GCP,\
    GCP_PROJECT_ID=${PROJECT_ID},\
    APP_ENV=dev,\
    APP_VERSION=mvp3 \
  --source .
```

If your Cloud Function requires authentication (recommended for internal APIs), remove `--allow-unauthenticated` and call it with a Bearer token from your Kong/Keycloak setup.

---

## Section 5 — Testing

### Test AWS Secret Retrieval

Run this locally or in a Lambda test event to confirm AWS SSM access works:

```python
# test_aws.py — run with: AWS_PROFILE=your-profile python test_aws.py
import os
os.environ['CLOUD_PROVIDER'] = 'AWS'

from secrets_client import get_postgres_config, get_postgres_dsn

try:
    config = get_postgres_config()
    print("AWS SSM fetch: PASS")
    print(f"  Host:     {config['host']}")
    print(f"  Port:     {config['port']}")
    print(f"  Database: {config['dbname']}")
    print(f"  User:     {config['username']}")
    print(f"  Password: {'*' * len(config['password'])}")
    print(f"  DSN:      {get_postgres_dsn().replace(config['password'], '***')}")
except Exception as e:
    print(f"AWS SSM fetch: FAIL — {e}")
```

### Test GCP Secret Retrieval

```python
# test_gcp.py — run with: GOOGLE_APPLICATION_CREDENTIALS=key.json python test_gcp.py
import os
os.environ['CLOUD_PROVIDER'] = 'GCP'
os.environ['GCP_PROJECT_ID'] = 'your-project-id'

from secrets_client import get_postgres_config, get_postgres_dsn

try:
    config = get_postgres_config()
    print("GCP Secret Manager fetch: PASS")
    print(f"  Host:     {config['host']}")
    print(f"  Port:     {config['port']}")
    print(f"  Database: {config['dbname']}")
    print(f"  User:     {config['username']}")
    print(f"  Password: {'*' * len(config['password'])}")
    print(f"  DSN:      {get_postgres_dsn().replace(config['password'], '***')}")
except Exception as e:
    print(f"GCP Secret Manager fetch: FAIL — {e}")
```

### Test PostgreSQL Connection

After confirming secrets fetch correctly, test actual database connectivity:

```python
# test_connection.py
import os
import psycopg2

# Set CLOUD_PROVIDER before importing
os.environ['CLOUD_PROVIDER'] = 'GCP'  # or 'AWS'
os.environ['GCP_PROJECT_ID'] = 'your-project-id'

from secrets_client import get_postgres_dsn

try:
    dsn = get_postgres_dsn()
    conn = psycopg2.connect(dsn)
    with conn.cursor() as cur:
        cur.execute("SELECT version(), pg_is_in_recovery()")
        version, is_replica = cur.fetchone()
    conn.close()
    print(f"Database connection: PASS")
    print(f"  PostgreSQL version: {version}")
    print(f"  Is replica: {is_replica}")
except psycopg2.OperationalError as e:
    print(f"Database connection: FAIL — {e}")
```

---

## Section 6 — GCP IAM Policy Reference

The equivalent mapping between your AWS KMS policy statements and GCP IAM:

| AWS KMS Policy Statement | GCP Equivalent |
|---|---|
| `EnableRootAccessAndPreventPermissionDelegation` | GCP org admin always retains access — no equivalent statement needed |
| `AllowAdminUserSessionFullKMSAccess` | `GCP-ifrs-user` group has `Cloud KMS Admin` — already in place |
| `AllowDecryptViaSSMForApprovedUserSessionsOnAllowedPath` | Secret-level IAM binding for approved users with `secretVersionAccessor` |
| `DenyDecryptForAllOtherSessionsOfSharedSSORole` | GCP uses allow-only model — no access = no access. No explicit deny needed |
| `AllowLambdaRoleDecryptViaSSMForInsightGenPostgres` | Service account `app-svc-ifrs-fs` has `secretAccessor` on insightgen-postgres-* secrets |

GCP IAM is an allow-only model by default. Unlike AWS where you needed an explicit Deny statement to block other sessions sharing the same role, in GCP simply not granting a binding means no access. This makes the policy model simpler but also means you cannot override a higher-level grant with a lower-level deny — be careful with project-wide bindings.

---

## Section 7 — Troubleshooting

### AWS — Permission Denied on SSM GetParameter

```
ClientError: An error occurred (AccessDeniedException) when calling
the GetParametersByPath operation
```

Check:
1. Lambda execution role has `ssm:GetParametersByPath` for the path `/insightgen/postgres/*`
2. Lambda execution role is in the KMS key policy under `AllowLambdaRoleDecryptViaSSMForInsightGenPostgres`
3. The KMS key ID in the policy matches the key used to encrypt the SSM parameters

```bash
# Verify Lambda role ARN matches what is in KMS key policy
aws lambda get-function-configuration \
  --function-name dev-api-configurator \
  --query 'Role'
```

---

### GCP — Permission Denied on Secret Access

```
google.api_core.exceptions.PermissionDenied: 403 Permission denied on
resource project PROJECT_ID
```

Check:
1. The Cloud Function is running as the correct service account
2. The service account has `roles/secretmanager.secretAccessor` on the secret

```bash
# Check which service account a Cloud Function runs as
gcloud functions describe insightgen-api-configurator \
  --project $PROJECT_ID \
  --region $REGION \
  --format="value(serviceAccountEmail)"

# Check IAM on the specific secret
gcloud secrets get-iam-policy insightgen-postgres-password \
  --project $PROJECT_ID
```

---

### GCP — Secret Not Found

```
google.api_core.exceptions.NotFound: 404 Secret [projects/X/secrets/Y] not found
```

Check:
1. Secret names follow exact convention: `insightgen-postgres-{key}` where key is one of `host`, `port`, `dbname`, `username`, `password`
2. Secret has at least one active version

```bash
# List secrets and their versions
gcloud secrets list --project $PROJECT_ID --filter="name:insightgen-postgres"

# Check versions of a specific secret
gcloud secrets versions list insightgen-postgres-password --project $PROJECT_ID
```

---

### LRU Cache — Stale Credentials After Rotation

The `@lru_cache` on `get_postgres_config()` means the function fetches secrets once per Lambda/Cloud Function instance lifetime. After rotating credentials, new instances pick up the new value immediately. Existing warm instances continue using the cached old value until they are recycled (typically within 15 minutes for Lambda, sooner if you force a redeployment).

To force immediate pickup of rotated credentials without a code deployment:

```bash
# AWS — update function to force new instances
aws lambda update-function-configuration \
  --function-name dev-api-configurator \
  --description "Force credential refresh $(date)"

# GCP — redeploy (no code change, just triggers new instances)
gcloud functions deploy insightgen-api-configurator \
  --project $PROJECT_ID --region $REGION --source .
```

---

## Summary

| Item | AWS | GCP |
|---|---|---|
| Secret store | SSM Parameter Store | Secret Manager |
| Encryption key | Custom KMS key (existing) | Cloud KMS CMEK (new) |
| Path/naming | `/insightgen/postgres/{key}` | `insightgen-postgres-{key}` |
| Runtime auth | Lambda execution role (IAM) | Service account `app-svc-ifrs-fs` |
| Human access | KMS key policy (approved SSO session) | Secret-level IAM binding (group user) |
| Audit trail | CloudTrail (enabled) | Secret Manager audit logs (enable in console) |
| Code change per cloud | None — `CLOUD_PROVIDER` env var only | None — `CLOUD_PROVIDER` env var only |
| Python SDK | `boto3` | `google-cloud-secret-manager` |
