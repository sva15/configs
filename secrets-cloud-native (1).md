# Secrets Management — Cloud-Native Implementation
**Project:** InsightGen  
**Approach:** AWS SSM Parameter Store + GCP Secret Manager  
**Author:** fitindevops  
**Last updated:** March 2026

---

## Overview

This document covers implementing cloud-native secret retrieval for PostgreSQL credentials across AWS (Lambda) and GCP (Cloud Run). The same Python application code is deployed to both clouds. A single environment variable `CLOUD_PROVIDER` switches which backend is used at runtime.

All secret retrieval logic is isolated in one file — `secrets_client.py`. The `main.py` application file is identical on both clouds with zero cloud-specific code.

```
CLOUD_PROVIDER=AWS  →  reads from AWS SSM Parameter Store
CLOUD_PROVIDER=GCP  →  reads from GCP Secret Manager
```

**Credential type:** Static PostgreSQL credentials (host, port, dbname, username, password). These do not rotate frequently, making them a good fit for the singleton pattern described in Section 3.

---

## Architecture

```
AWS Lambda                          GCP Cloud Run (HTTP)
    │                                       │
    │ CLOUD_PROVIDER=AWS                    │ CLOUD_PROVIDER=GCP
    │                                       │
    ▼                                       ▼
secrets_client.py  ←── same file ──▶  secrets_client.py
[Singleton loads once at import]      [Singleton loads once at import]
    │                                       │
    ▼                                       ▼
AWS SSM Parameter Store             GCP Secret Manager
/insightgen/postgres/*              insightgen-postgres-*
    │                                       │
    ▼                                       ▼
KMS (custom key, existing)          Cloud KMS (CMEK key)
    │                                       │
    ▼                                       ▼
PostgreSQL container                PostgreSQL container
```

---

## Section 1 — AWS Setup (Already Complete)

Your AWS setup is complete from previous work. For reference:

**SSM Parameters already created:**
```
/insightgen/postgres/host
/insightgen/postgres/port
/insightgen/postgres/dbname
/insightgen/postgres/username
/insightgen/postgres/password
```

**KMS key policy already hardened** with:
- Root account emergency recovery
- Approved SSO user session access
- Lambda execution role (`HCL-User-Role-InsightGen-Lambda`) decrypt access scoped to SSM via service context

**Lambda IAM role** already has `ssm:GetParametersByPath` and `kms:Decrypt` permissions.

No changes needed on the AWS side.

---

## Section 2 — GCP Setup

### Your GCP Access

Based on your IAM roles, everything needed is already granted:

| Principal | Role | What it allows |
|---|---|---|
| `app-svc-ifrs-fs` (service account) | `Secret Manager Secret Accessor` | Cloud Run can read secret values at runtime |
| `app-svc-ifrs-fs` (service account) | `Cloud KMS CryptoKey Decrypter` | Can decrypt CMEK-encrypted secrets |
| `GCP-ifrs-user` group (your user) | `Secret Manager Admin` | You can create and manage secrets |
| `GCP-ifrs-user` group (your user) | `Cloud KMS Admin` | You can create and manage KMS keys |

You do not need to request any new permissions.

---

### Step 1 — Set Shell Variables

Run these before any other commands so you do not have to repeat the values:

```bash
export PROJECT_ID="your-actual-gcp-project-id"
export REGION="us-central1"    # Replace with your actual GCP region
export SA_EMAIL="app-svc-ifrs-fs@${PROJECT_ID}.iam.gserviceaccount.com"

# Confirm you are authenticated as your group user
gcloud auth list
gcloud config set project $PROJECT_ID
```

---

### Step 2 — Create Cloud KMS Key for Secret Encryption

This is the GCP equivalent of your AWS custom KMS key. Secrets are encrypted with this CMEK key giving you control over who can decrypt rather than relying on Google's default key.

```bash
# Create a key ring — logical container for keys in GCP KMS
gcloud kms keyrings create insightgen-keyring \
  --location $REGION \
  --project $PROJECT_ID

# Create the encryption key
gcloud kms keys create insightgen-secrets-key \
  --location $REGION \
  --keyring insightgen-keyring \
  --purpose encryption \
  --project $PROJECT_ID

# Store the full key resource name — used in subsequent commands
export KMS_KEY="projects/${PROJECT_ID}/locations/${REGION}/keyRings/insightgen-keyring/cryptoKeys/insightgen-secrets-key"
echo "KMS Key: $KMS_KEY"
```

Grant the Secret Manager service agent permission to use this key. Secret Manager needs this to perform encryption and decryption on your behalf when storing and retrieving secret versions:

```bash
# Get the Secret Manager service agent email for your project
PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format='value(projectNumber)')
SM_SA="service-${PROJECT_NUMBER}@gcp-sa-secretmanager.iam.gserviceaccount.com"

echo "Granting KMS access to Secret Manager service agent: $SM_SA"

gcloud kms keys add-iam-policy-binding insightgen-secrets-key \
  --location $REGION \
  --keyring insightgen-keyring \
  --member "serviceAccount:${SM_SA}" \
  --role "roles/cloudkms.cryptoKeyEncrypterDecrypter" \
  --project $PROJECT_ID

echo "Done"
```

---

### Step 3 — Create PostgreSQL Secrets

Create each secret using your CMEK key. The naming convention mirrors your AWS SSM path — `/insightgen/postgres/{key}` becomes `insightgen-postgres-{key}` in GCP.

```bash
# Create the five secret definitions
for KEY in host port dbname username password; do
  gcloud secrets create insightgen-postgres-${KEY} \
    --project $PROJECT_ID \
    --replication-policy user-managed \
    --locations $REGION \
    --kms-key-name $KMS_KEY

  echo "Created: insightgen-postgres-${KEY}"
done
```

Add the actual credential values. Replace each placeholder with your real GCP PostgreSQL connection details:

```bash
# Host — IP address or hostname of your GCP PostgreSQL container
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

# Password — the -n flag is important: prevents a trailing newline
# being stored as part of the password value
echo -n "YOUR_ACTUAL_POSTGRES_PASSWORD" | \
  gcloud secrets versions add insightgen-postgres-password \
  --data-file=- --project $PROJECT_ID
```

Verify all five secrets were created with active versions:

```bash
gcloud secrets list \
  --project $PROJECT_ID \
  --filter="name:insightgen-postgres" \
  --format="table(name, createTime)"

# Spot-check one value to confirm it stored correctly
gcloud secrets versions access latest \
  --secret insightgen-postgres-host \
  --project $PROJECT_ID
```

---

### Step 4 — Verify Service Account Access

Confirm the service account that runs your Cloud Run functions can read these secrets:

```bash
# Check what Secret Manager roles the service account has at project level
gcloud projects get-iam-policy $PROJECT_ID \
  --flatten="bindings[].members" \
  --filter="bindings.members:${SA_EMAIL} AND bindings.role:secretmanager" \
  --format="table(bindings.role)"
```

You should see `roles/secretmanager.secretAccessor` in the output. If the role is granted at the project level, the service account can read all secrets in the project.

**Optional — scope to insightgen secrets only:**

If you want tighter scoping so the service account can only read the insightgen PostgreSQL secrets and nothing else in the project, run these commands. This mirrors the path-scoped approach in your AWS KMS policy:

```bash
# Remove project-wide accessor
gcloud projects remove-iam-policy-binding $PROJECT_ID \
  --member "serviceAccount:${SA_EMAIL}" \
  --role "roles/secretmanager.secretAccessor"

# Grant per-secret accessor instead
for KEY in host port dbname username password; do
  gcloud secrets add-iam-policy-binding insightgen-postgres-${KEY} \
    --project $PROJECT_ID \
    --member "serviceAccount:${SA_EMAIL}" \
    --role "roles/secretmanager.secretAccessor"
  echo "Per-secret access granted: insightgen-postgres-${KEY}"
done
```

---

## Section 3 — The Singleton Pattern

Your architect recommended the singleton pattern for loading these credentials. This is the correct approach for static credentials, and here is exactly what it means in practice.

### What the pattern does

A module-level variable holds the credentials after the first load. The load function checks whether the variable is already populated — if yes, it returns immediately without any network call. If no, it fetches from SSM or Secret Manager, stores the result, and returns it.

```
First call (cold start):
  _postgres_config is None
  → fetch from SSM or Secret Manager  (one network call)
  → store in _postgres_config
  → return credentials

All subsequent calls (same instance, warm):
  _postgres_config is already populated
  → return immediately, zero network calls
```

For static credentials that never change this is the right pattern because:
- The credential fetch happens once per Lambda container or Cloud Run instance lifetime
- No TTL or expiry logic is needed since the values do not rotate
- If the instance is recycled (redeployment, scaling event), the next cold start fetches fresh values automatically

### Why not `@lru_cache`

`@lru_cache` achieves a similar result but the singleton with a module-level variable is more explicit — you can see exactly what is cached, inspect it in tests, and reset it manually if needed during testing. It is also easier for developers unfamiliar with Python decorators to understand.

---

## Section 4 — Python Implementation

### Project structure

```
your-function/
├── main.py              # Lambda handler + Cloud Run entry point — identical both clouds
├── secrets_client.py    # All cloud-specific code lives here
├── requirements-aws.txt # Dependencies for Lambda layer
└── requirements-gcp.txt # Dependencies for Cloud Run package
```

---

### File: `secrets_client.py`

```python
"""
secrets_client.py
-----------------
Cloud-agnostic PostgreSQL credential retrieval using the singleton pattern.

Set CLOUD_PROVIDER=AWS or CLOUD_PROVIDER=GCP in the function's environment variables.

AWS:  fetches from SSM Parameter Store at /insightgen/postgres/
GCP:  fetches from Secret Manager at insightgen-postgres-{key}

Credentials are static and are loaded once at module import time.
All subsequent calls within the same instance return the cached value
with zero network overhead.
"""

import os
import logging

logger = logging.getLogger(__name__)

# ─── Singleton state ──────────────────────────────────────────────────────────
# Module-level variable — None until first load, populated once, never cleared.
# This is the singleton: one instance of the credentials per container lifetime.
_postgres_config = None

# Required keys that must be present in the fetched credentials
_REQUIRED_KEYS = {'host', 'port', 'dbname', 'username', 'password'}

# Read CLOUD_PROVIDER once at module import — does not change during runtime
_CLOUD_PROVIDER = os.environ.get('CLOUD_PROVIDER', 'AWS').upper()


# ─── AWS SSM fetch ────────────────────────────────────────────────────────────

def _load_from_aws() -> dict:
    """
    Fetch all PostgreSQL parameters from AWS SSM in a single batch call.
    Decryption is handled by the Lambda execution role via the KMS key policy.
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
        raise RuntimeError(
            f"SSM GetParametersByPath failed [{code}]. "
            f"Check Lambda role has ssm:GetParametersByPath on {prefix}* "
            f"and kms:Decrypt on the InsightGen KMS key."
        ) from e

    if not response.get('Parameters'):
        raise RuntimeError(
            f"No parameters returned from SSM path: {prefix}. "
            f"Verify parameters exist and the Lambda role KMS policy is correct."
        )

    # Convert /insightgen/postgres/host → host
    result = {
        p['Name'].replace(prefix, ''): p['Value']
        for p in response['Parameters']
    }

    logger.info(
        "PostgreSQL config loaded from AWS SSM "
        f"[region={region}, params={list(result.keys())}]"
    )
    return result


# ─── GCP Secret Manager fetch ─────────────────────────────────────────────────

def _load_from_gcp() -> dict:
    """
    Fetch all PostgreSQL secrets from GCP Secret Manager.
    Authentication uses the service account attached to the Cloud Run function.
    No token or key file is needed — the service account identity is automatic.
    """
    from google.cloud import secretmanager
    from google.api_core.exceptions import NotFound, PermissionDenied, GoogleAPIError

    project_id = os.environ.get('GCP_PROJECT_ID')
    if not project_id:
        raise ValueError(
            "GCP_PROJECT_ID environment variable must be set when CLOUD_PROVIDER=GCP."
        )

    client = secretmanager.SecretManagerServiceClient()
    result = {}

    for key in _REQUIRED_KEYS:
        secret_name = (
            f"projects/{project_id}/secrets/"
            f"insightgen-postgres-{key}/versions/latest"
        )
        try:
            response = client.access_secret_version(request={"name": secret_name})
            result[key] = response.payload.data.decode("UTF-8")

        except NotFound:
            raise RuntimeError(
                f"Secret not found: insightgen-postgres-{key} "
                f"in project {project_id}. "
                f"Run: gcloud secrets create insightgen-postgres-{key} ..."
            )
        except PermissionDenied:
            raise RuntimeError(
                f"Permission denied reading insightgen-postgres-{key}. "
                f"Verify service account has roles/secretmanager.secretAccessor "
                f"on this secret."
            )
        except GoogleAPIError as e:
            raise RuntimeError(
                f"GCP Secret Manager error reading insightgen-postgres-{key}: {e}"
            ) from e

    logger.info(
        "PostgreSQL config loaded from GCP Secret Manager "
        f"[project={project_id}, secrets={list(result.keys())}]"
    )
    return result


# ─── Singleton loader ─────────────────────────────────────────────────────────

def _load_postgres_config() -> dict:
    """
    Internal — fetch credentials from the appropriate cloud backend.
    Called exactly once per container lifetime by get_postgres_config().
    """
    if _CLOUD_PROVIDER == 'AWS':
        return _load_from_aws()
    elif _CLOUD_PROVIDER == 'GCP':
        return _load_from_gcp()
    else:
        raise ValueError(
            f"Unsupported CLOUD_PROVIDER: '{_CLOUD_PROVIDER}'. "
            f"Must be 'AWS' or 'GCP'."
        )


# ─── Public API ───────────────────────────────────────────────────────────────

def get_postgres_config() -> dict:
    """
    Return PostgreSQL connection credentials.

    Singleton pattern: credentials are loaded once on the first call and
    cached in the module-level _postgres_config variable. All subsequent
    calls return the cached value immediately with no network calls.

    This is correct for static credentials that do not rotate.
    If credentials are ever rotated, redeploy the function to get a fresh
    cold start which will load the new values.

    Returns:
        dict with keys: host, port, dbname, username, password

    Raises:
        ValueError:  CLOUD_PROVIDER is not AWS or GCP
        RuntimeError: credentials could not be fetched from the secret store
        KeyError:    required keys are missing from the secret store
    """
    global _postgres_config

    # Singleton check — if already loaded, return immediately (no network call)
    if _postgres_config is not None:
        return _postgres_config

    # First call only — load from cloud secret store
    logger.info(f"Cold start: loading PostgreSQL config from {_CLOUD_PROVIDER}")
    raw = _load_postgres_config()

    # Validate all required keys are present before caching
    missing = _REQUIRED_KEYS - set(raw.keys())
    if missing:
        raise KeyError(
            f"PostgreSQL config is missing required keys: {missing}. "
            f"Verify all five parameters exist in your {_CLOUD_PROVIDER} secret store: "
            f"host, port, dbname, username, password."
        )

    # Store in singleton — all future calls return this directly
    _postgres_config = raw
    return _postgres_config


def get_postgres_dsn() -> str:
    """
    Return a libpq connection string (DSN) for use with psycopg2.
    SSL is enforced via sslmode=require, matching the TLS config on the
    PostgreSQL container configured in previous tasks.

    Example:
        host=10.x.x.x port=5432 dbname=insightgen
        user=insightgen_app password=secret sslmode=require
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
    Return a SQLAlchemy-compatible PostgreSQL connection URL.
    Use this if your application uses SQLAlchemy instead of psycopg2 directly.
    """
    cfg = get_postgres_config()
    return (
        f"postgresql+psycopg2://"
        f"{cfg['username']}:{cfg['password']}"
        f"@{cfg['host']}:{cfg['port']}/{cfg['dbname']}"
        f"?sslmode=require"
    )
```

---

### File: `main.py`

Application code — identical on both clouds. No cloud-specific imports or logic here:

```python
"""
main.py
-------
Lambda handler (AWS) and Cloud Run HTTP handler (GCP).

This file is identical on both clouds.
All cloud-specific code lives in secrets_client.py.
"""

import json
import logging
import psycopg2
from secrets_client import get_postgres_dsn

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Module-level database connection — reused across warm invocations.
# Combined with the singleton in secrets_client.py, this means:
#   Cold start: fetch credentials → open connection
#   Warm call:  reuse existing connection (no credential fetch, no reconnect)
_db_conn = None


def _get_connection():
    """
    Return a live PostgreSQL connection.
    Opens a new connection on cold start or after a connection is closed.
    Reuses the existing connection on warm invocations.
    """
    global _db_conn
    if _db_conn is None or _db_conn.closed != 0:
        logger.info("Opening PostgreSQL connection")
        _db_conn = psycopg2.connect(get_postgres_dsn())
        _db_conn.autocommit = True
        logger.info("PostgreSQL connection established")
    return _db_conn


# ─── AWS Lambda ───────────────────────────────────────────────────────────────

def lambda_handler(event, context):
    """
    AWS Lambda entry point.
    Called by API Gateway, ALB, or direct invocation from the console.
    """
    try:
        conn = _get_connection()
        with conn.cursor() as cur:
            cur.execute(
                "SELECT version(), current_database(), current_user, inet_server_addr()"
            )
            pg_version, db_name, db_user, db_host = cur.fetchone()

        return {
            'statusCode': 200,
            'headers': {'Content-Type': 'application/json'},
            'body': json.dumps({
                'status': 'connected',
                'cloud': 'AWS',
                'db_version': pg_version,
                'database': db_name,
                'user': db_user,
                'connected_to': str(db_host)
            })
        }

    except psycopg2.OperationalError as e:
        logger.error(f"PostgreSQL connection error: {e}")
        return {
            'statusCode': 503,
            'headers': {'Content-Type': 'application/json'},
            'body': json.dumps({
                'status': 'error',
                'type': 'database_connection',
                'detail': str(e)
            })
        }
    except Exception as e:
        logger.error(f"Unexpected error: {e}")
        return {
            'statusCode': 500,
            'headers': {'Content-Type': 'application/json'},
            'body': json.dumps({
                'status': 'error',
                'type': 'internal',
                'detail': str(e)
            })
        }


# ─── GCP Cloud Run (HTTP function) ────────────────────────────────────────────

def cloud_run_handler(request):
    """
    GCP Cloud Run HTTP function entry point.
    Called via HTTP — tested with curl.
    """
    import flask

    if request.method not in ('GET', 'POST'):
        return flask.Response(
            json.dumps({'error': 'Method not allowed'}),
            status=405,
            mimetype='application/json'
        )

    try:
        conn = _get_connection()
        with conn.cursor() as cur:
            cur.execute(
                "SELECT version(), current_database(), current_user, inet_server_addr()"
            )
            pg_version, db_name, db_user, db_host = cur.fetchone()

        return flask.Response(
            json.dumps({
                'status': 'connected',
                'cloud': 'GCP',
                'db_version': pg_version,
                'database': db_name,
                'user': db_user,
                'connected_to': str(db_host)
            }),
            status=200,
            mimetype='application/json'
        )

    except psycopg2.OperationalError as e:
        logger.error(f"PostgreSQL connection error: {e}")
        return flask.Response(
            json.dumps({
                'status': 'error',
                'type': 'database_connection',
                'detail': str(e)
            }),
            status=503,
            mimetype='application/json'
        )
    except Exception as e:
        logger.error(f"Unexpected error: {e}")
        return flask.Response(
            json.dumps({
                'status': 'error',
                'type': 'internal',
                'detail': str(e)
            }),
            status=500,
            mimetype='application/json'
        )
```

---

### File: `requirements-aws.txt`

Used when building the Lambda layer. No GCP libraries included:

```
psycopg2-binary==2.9.9
boto3==1.34.0
```

---

### File: `requirements-gcp.txt`

Used when packaging the Cloud Run function. No AWS libraries included:

```
psycopg2-binary==2.9.9
google-cloud-secret-manager==2.20.0
flask==3.0.0
functions-framework==3.5.0
```

---

## Section 5 — Deployment

### Deploy to AWS Lambda

Set the two required environment variables on the function. No secrets go here — only the provider name and app metadata:

```bash
FUNCTION_NAME="dev-api-configurator"   # Replace with your actual function name

aws lambda update-function-configuration \
  --function-name $FUNCTION_NAME \
  --environment 'Variables={
    CLOUD_PROVIDER=AWS,
    APP_ENV=dev,
    APP_VERSION=mvp3
  }'

echo "Environment updated on: $FUNCTION_NAME"
```

No IAM changes needed — the Lambda execution role already has the required SSM and KMS permissions.

---

### Deploy to GCP Cloud Run

```bash
# From your function source directory
cd your-function-directory

gcloud run deploy insightgen-api-configurator \
  --project $PROJECT_ID \
  --region $REGION \
  --runtime python312 \
  --trigger-http \
  --service-account $SA_EMAIL \
  --set-env-vars \
    CLOUD_PROVIDER=GCP,\
    GCP_PROJECT_ID=${PROJECT_ID},\
    APP_ENV=dev,\
    APP_VERSION=mvp3 \
  --source .
```

After deployment, note the function URL printed in the output — you need it for curl testing. It looks like:
```
https://insightgen-api-configurator-xxxx-uc.a.run.app
```

---

## Section 6 — Testing

### Testing on AWS Lambda — Console

**Step 1 — Open the Lambda function**

Go to AWS Lambda console → open your function (e.g. `dev-api-configurator`) → click the **Test** tab.

**Step 2 — Create a test event**

Click **Create new event**. Use this JSON as the test payload and name it `ConnectionTest`:

```json
{
  "httpMethod": "GET",
  "path": "/health",
  "headers": {
    "Content-Type": "application/json"
  },
  "body": null,
  "queryStringParameters": null
}
```

Click **Save**.

**Step 3 — Run the test**

Click the **Test** button. The Execution results panel opens with the response.

**Expected success response:**

```json
{
  "statusCode": 200,
  "headers": {
    "Content-Type": "application/json"
  },
  "body": "{\"status\": \"connected\", \"cloud\": \"AWS\", \"db_version\": \"PostgreSQL 15.4...\", \"database\": \"insightgen\", \"user\": \"insightgen_app\", \"connected_to\": \"10.x.x.x\"}"
}
```

**Step 4 — Diagnose failures**

| `statusCode` | `type` or `detail` | What to check |
|---|---|---|
| 200 | `status: connected` | All working correctly |
| 500 | mentions `SSM` or `AccessDeniedException` | Lambda role missing `ssm:GetParametersByPath` or wrong parameter path |
| 500 | mentions `KMS` or `AccessDenied` | Lambda role not in KMS key policy — check `AllowLambdaRoleDecryptViaSSMForInsightGenPostgres` statement |
| 500 | mentions `No parameters returned` | Parameters do not exist at `/insightgen/postgres/` — verify with `aws ssm describe-parameters` |
| 503 | `type: database_connection` | Secrets fetched correctly but PostgreSQL unreachable — check VPC routing between Lambda and EC2 |

**Step 5 — Check logs for singleton behaviour**

Click **View CloudWatch logs** in the Execution results panel.

On a cold start (first test run), you should see:
```
INFO: Cold start: loading PostgreSQL config from AWS
INFO: PostgreSQL config loaded from AWS SSM [region=us-east-1, params=['host', 'port', 'dbname', 'username', 'password']]
INFO: Opening PostgreSQL connection
INFO: PostgreSQL connection established
```

Run the test a second time immediately. You should NOT see the SSM or connection messages again — only the handler output. This confirms the singleton is working and secrets are not being re-fetched on every call.

---

### Testing on GCP Cloud Run — curl

**Step 1 — Get the function URL**

```bash
FUNCTION_URL=$(gcloud run services describe insightgen-api-configurator \
  --project $PROJECT_ID \
  --region $REGION \
  --format 'value(status.url)')

echo "Function URL: $FUNCTION_URL"
```

Or copy it directly from GCP Cloud Console → Cloud Run → your function → URL shown at the top of the page.

**Step 2 — Get an identity token**

Cloud Run functions require authentication by default. This command gets a short-lived token for your user account:

```bash
TOKEN=$(gcloud auth print-identity-token)
```

This token expires after about 1 hour. Re-run the command if you get a 401 error.

**Step 3 — Basic connection test**

```bash
curl -s \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "${FUNCTION_URL}" \
  | python3 -m json.tool
```

**Expected success response:**

```json
{
  "status": "connected",
  "cloud": "GCP",
  "db_version": "PostgreSQL 15.4 on x86_64-pc-linux-musl...",
  "database": "insightgen",
  "user": "insightgen_app",
  "connected_to": "10.x.x.x"
}
```

**Step 4 — Useful curl variants**

Check HTTP status code only — useful when you want to confirm the function responds without reading the body:

```bash
curl -s -o /dev/null -w "HTTP status: %{http_code}\n" \
  -H "Authorization: Bearer $TOKEN" \
  "${FUNCTION_URL}"
```

Show both status code and response body together:

```bash
curl -s \
  -w "\n--- HTTP %{http_code} ---\n" \
  -H "Authorization: Bearer $TOKEN" \
  "${FUNCTION_URL}" \
  | python3 -m json.tool
```

Measure cold start versus warm call latency to verify the singleton is caching:

```bash
# First call — cold start (singleton fetches secrets + opens connection)
echo "Cold start:"
time curl -s \
  -H "Authorization: Bearer $TOKEN" \
  "${FUNCTION_URL}" > /dev/null

# Second call immediately after — warm (singleton returns cached, connection reused)
echo "Warm call:"
time curl -s \
  -H "Authorization: Bearer $TOKEN" \
  "${FUNCTION_URL}" > /dev/null
```

The cold start will be noticeably slower due to the Secret Manager network call plus PostgreSQL connection setup. The warm call should complete significantly faster.

**Step 5 — Diagnose failures**

| HTTP status | Response body | What to check |
|---|---|---|
| 200 | `status: connected` | All working correctly |
| 401 | `permission denied` or empty | Token expired — re-run `gcloud auth print-identity-token` |
| 500 | mentions `GCP_PROJECT_ID` | Environment variable not set on the Cloud Run function |
| 500 | `Secret not found` | Secret name wrong or not created — run `gcloud secrets list --filter="name:insightgen-postgres"` |
| 500 | `Permission denied` | Service account missing `secretAccessor` on the secret — run Step 4 of Section 2 |
| 503 | `type: database_connection` | Secrets fetched correctly but PostgreSQL unreachable from Cloud Run — check network/firewall rules |

**Step 6 — View Cloud Run logs**

Tail live logs during testing:

```bash
gcloud run services logs tail insightgen-api-configurator \
  --project $PROJECT_ID \
  --region $REGION
```

Or query recent logs after a test:

```bash
gcloud logging read \
  "resource.type=cloud_run_revision \
   AND resource.labels.service_name=insightgen-api-configurator" \
  --project $PROJECT_ID \
  --limit 30 \
  --format "table(timestamp, textPayload)"
```

On cold start you should see:
```
Cold start: loading PostgreSQL config from GCP
PostgreSQL config loaded from GCP Secret Manager [project=..., secrets=[...]]
Opening PostgreSQL connection
PostgreSQL connection established
```

On subsequent warm calls, those lines should not appear — confirming the singleton and connection reuse are working.

---

## Section 7 — Singleton Behaviour Reference

Understanding when the singleton reloads helps manage static credentials correctly:

| Event | Singleton state | Secret store called? |
|---|---|---|
| First invocation — cold start | `_postgres_config` is None → loads | Yes — once |
| Subsequent calls — warm instance | `_postgres_config` already set → returns | No |
| Lambda redeployment | New container on next invoke → cold start | Yes — once |
| Cloud Run redeployment | New container on next invoke → cold start | Yes — once |
| Lambda instance recycled / timeout | New container → cold start | Yes — once |
| Instance scaled up (new parallel instance) | Fresh container → cold start | Yes — once per instance |

**When credentials change in future:**

Since the singleton loads at cold start, rotating credentials requires forcing new cold starts across all running instances. Redeploy the function with a no-op change:

```bash
# AWS — touch the description to force new containers
aws lambda update-function-configuration \
  --function-name dev-api-configurator \
  --description "Credential refresh $(date +%Y-%m-%d)"

# GCP — update the service to force new instances
gcloud run services update insightgen-api-configurator \
  --project $PROJECT_ID \
  --region $REGION
```

---

## Section 8 — Troubleshooting

### AWS — `AccessDeniedException` on SSM

```
RuntimeError: SSM GetParametersByPath failed [AccessDeniedException].
```

Verify the Lambda execution role ARN matches the `AllowLambdaRoleDecryptViaSSMForInsightGenPostgres` statement in the KMS key policy:

```bash
# Get the Lambda role ARN
aws lambda get-function-configuration \
  --function-name dev-api-configurator \
  --query 'Role' --output text

# View the KMS key policy and compare
aws kms get-key-policy \
  --key-id YOUR_KMS_KEY_ID \
  --policy-name default \
  --query 'Policy' --output text | python3 -m json.tool
```

---

### AWS — `No parameters returned`

```
RuntimeError: No parameters returned from SSM path: /insightgen/postgres/.
```

List what is actually stored and check the path:

```bash
aws ssm describe-parameters \
  --filters "Key=Path,Values=/insightgen/postgres/" \
  --query 'Parameters[*].Name' \
  --output table
```

---

### GCP — 401 on curl

The identity token has expired (tokens last about 1 hour):

```bash
TOKEN=$(gcloud auth print-identity-token)
# Re-run your curl command with the new token
```

---

### GCP — `Permission denied` in function logs

Check which service account the Cloud Run function is actually running as. If it is not `app-svc-ifrs-fs`, the function was deployed without specifying the service account:

```bash
gcloud run services describe insightgen-api-configurator \
  --project $PROJECT_ID \
  --region $REGION \
  --format 'value(spec.template.spec.serviceAccountName)'
```

If the output is wrong, redeploy with `--service-account $SA_EMAIL` explicitly.

---

### Singleton not caching — secrets fetched on every call

This should not happen in normal Lambda or Cloud Run execution. If you see the cold start log message on every invocation, check that `_postgres_config` is declared at module level (not inside any function) in `secrets_client.py`. If the import path changes between calls, Python may be loading the module fresh each time.

To reset the singleton manually during local testing:

```python
import secrets_client
secrets_client._postgres_config = None   # Reset for isolated test
```

---

## Summary

| Item | AWS | GCP |
|---|---|---|
| Secret store | SSM Parameter Store | Secret Manager |
| Encryption key | Custom KMS key (existing) | Cloud KMS CMEK (new) |
| Naming convention | `/insightgen/postgres/{key}` | `insightgen-postgres-{key}` |
| Runtime auth | Lambda execution role (IAM) | Service account `app-svc-ifrs-fs` |
| Credential type | Static — loaded once at cold start | Static — loaded once at cold start |
| Caching pattern | Singleton (`_postgres_config`) | Singleton (`_postgres_config`) |
| Code change per cloud | None — `CLOUD_PROVIDER` env var only | None — `CLOUD_PROVIDER` env var only |
| Python SDK | `boto3` | `google-cloud-secret-manager` |
| Testing method | AWS Lambda console → Test tab | curl with `gcloud auth print-identity-token` |
| Log location | CloudWatch Logs | Cloud Logging / `gcloud run services logs tail` |
