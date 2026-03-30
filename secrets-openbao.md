# Secrets Management — OpenBao Unified Implementation
**Project:** InsightGen  
**Approach:** OpenBao as a single secrets store across AWS and GCP  
**Author:** fitindevops  
**Status:** Implementation guide (POC / evaluation)

---

## Overview

This document covers deploying OpenBao as a self-managed, cloud-agnostic secrets store. The same Python code runs on both AWS Lambda and GCP Cloud Functions. The only thing that changes between clouds is the `VAULT_ADDR` environment variable — the endpoint pointing to the OpenBao instance in that cloud.

OpenBao is an open-source fork of HashiCorp Vault. The Python client library (`hvac`) and all API paths are identical to Vault. If you have used Vault before, OpenBao is a drop-in.

```
AWS Lambda                          GCP Cloud Function
    │                                       │
    │ VAULT_ADDR=http://aws-host:8200       │ VAULT_ADDR=http://gcp-host:8200
    │ VAULT_TOKEN=token-from-ssm            │ VAULT_TOKEN=token-from-secret-mgr
    │                                       │
    ▼                                       ▼
         hvac Python client (identical code both sides)
                          │
              ┌───────────┴───────────┐
              ▼                       ▼
    OpenBao on EC2 (AWS)      OpenBao on GCP VM
    insightgen/postgres/*     insightgen/postgres/*
              │                       │
              ▼                       ▼
    PostgreSQL (AWS)          PostgreSQL (GCP)
```

---

## When to Use OpenBao vs Cloud-Native

Use OpenBao when:
- You have three or more clouds and want one unified secrets API
- Your organisation already runs Vault and OpenBao fits that pattern
- You want one access policy model across all environments

Use cloud-native (SSM + Secret Manager) when:
- You have two clouds (AWS + GCP) — the code difference is minimal
- You want zero extra infrastructure to operate
- You are still in POC stage with a small team

For a two-cloud POC, cloud-native is simpler. This document exists so you can evaluate OpenBao and decide if it fits your roadmap.

---

## Important Consideration — The Bootstrap Problem

OpenBao stores your secrets. But to connect to OpenBao, your Lambda or Cloud Function needs a token. That token itself must be stored somewhere. This is called the bootstrap problem.

The solution used in this guide is to store the OpenBao token in the native secret store of each cloud — AWS SSM or GCP Secret Manager — as a single bootstrap secret. The application fetches that token at cold start and then uses it to fetch all other secrets from OpenBao.

```
Cold start flow:
  1. Lambda reads VAULT_TOKEN from AWS SSM          (one SSM call)
  2. Lambda calls OpenBao with that token           (one OpenBao call)
  3. OpenBao returns PostgreSQL credentials         (cached for instance lifetime)

  GCP Cloud Function does the same via GCP Secret Manager for the token.
```

---

## Architecture Decisions

### Dev Mode vs Production Mode

OpenBao can run in two modes:

**Dev mode** — single node, in-memory storage, auto-unsealed. Data is lost on restart. Good for testing and POC only.

**Production mode** — persistent storage, requires unsealing on startup, supports HA. Required for any real usage.

This guide covers dev mode for testing and production mode for actual deployment.

### Where to Deploy OpenBao

You deploy one OpenBao instance per cloud. This keeps your secrets close to the compute that uses them — Lambda reads from an AWS-hosted OpenBao, Cloud Functions read from a GCP-hosted OpenBao. Both OpenBao instances store the same secrets (mirrored manually or via replication).

---

## Section 1 — Deploy OpenBao on AWS EC2

OpenBao runs as a Docker container alongside your existing services on the EC2 instance.

### Step 1 — Add OpenBao to Docker Compose

In your existing `docker-compose.yml`, add the OpenBao service:

```yaml
services:

  # Add this to your existing services
  openbao:
    image: openbao/openbao:2.0.3
    container_name: insightgen-openbao
    restart: unless-stopped
    cap_add:
      - IPC_LOCK    # Required to prevent secrets from being swapped to disk
    environment:
      BAO_LOG_LEVEL: "info"
    ports:
      - "127.0.0.1:8200:8200"    # Bind to localhost only — same as Kong Admin API
    volumes:
      - openbao-config:/openbao/config
      - openbao-data:/openbao/data
      - openbao-logs:/openbao/logs
    command: server -config=/openbao/config/config.hcl
    networks:
      - insightgen-network
    healthcheck:
      test: ["CMD", "bao", "status"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s

volumes:
  openbao-config:
    name: insightgen-openbao-config
  openbao-data:
    name: insightgen-openbao-data
  openbao-logs:
    name: insightgen-openbao-logs
```

### Step 2 — Create OpenBao Configuration File

Connect to EC2 via SSM Session Manager and create the config:

```bash
# Create config directory
docker volume create insightgen-openbao-config
CONFIG_PATH=$(docker volume inspect insightgen-openbao-config --format '{{.Mountpoint}}')

cat > $CONFIG_PATH/config.hcl << 'EOF'
# OpenBao server configuration for InsightGen

ui = false    # No web UI needed — reduces attack surface

listener "tcp" {
  address       = "0.0.0.0:8200"
  tls_disable   = true    # TLS handled by the VPC boundary for now
                          # Enable tls_cert_file and tls_key_file for production
}

storage "file" {
  path = "/openbao/data"
}

# Prevent OpenBao from running as root
default_lease_ttl = "768h"   # 32 days
max_lease_ttl     = "8760h"  # 1 year
EOF

echo "OpenBao config created at: $CONFIG_PATH/config.hcl"
```

### Step 3 — Start OpenBao and Initialise

```bash
# Start OpenBao container
docker compose up -d openbao
sleep 10

# Verify it started
docker logs insightgen-openbao --tail 20

# Set environment for interacting with OpenBao from EC2
export BAO_ADDR="http://127.0.0.1:8200"

# Initialise — this generates the root token and unseal keys
# IMPORTANT: Save the output somewhere safe immediately
docker exec insightgen-openbao bao operator init \
  -key-shares=3 \
  -key-threshold=2 \
  -format=json > /tmp/openbao-init.json

cat /tmp/openbao-init.json
```

The output contains three unseal keys and one root token. Save all of these securely. You need any two of the three unseal keys to unseal OpenBao after a restart.

```bash
# Extract values from init output
UNSEAL_KEY_1=$(cat /tmp/openbao-init.json | python3 -c "import sys,json; print(json.load(sys.stdin)['unseal_keys_b64'][0])")
UNSEAL_KEY_2=$(cat /tmp/openbao-init.json | python3 -c "import sys,json; print(json.load(sys.stdin)['unseal_keys_b64'][1])")
ROOT_TOKEN=$(cat /tmp/openbao-init.json | python3 -c "import sys,json; print(json.load(sys.stdin)['root_token'])")

# Unseal using two of the three keys
docker exec insightgen-openbao bao operator unseal $UNSEAL_KEY_1
docker exec insightgen-openbao bao operator unseal $UNSEAL_KEY_2

# Verify it is unsealed and ready
docker exec -e BAO_ADDR=http://127.0.0.1:8200 \
  insightgen-openbao bao status
```

You should see `Sealed: false` in the status output.

### Step 4 — Handle Unseal on Container Restart

OpenBao seals itself every time it restarts. You must unseal it before any application can use it. Create an unseal script:

```bash
cat > /home/ec2-user/scripts/unseal-openbao.sh << 'SCRIPT'
#!/bin/bash
# Run this after any EC2 reboot or OpenBao container restart
# Store unseal keys in AWS SSM as SecureString for safe retrieval

export BAO_ADDR="http://127.0.0.1:8200"

# Fetch unseal keys from SSM
KEY1=$(aws ssm get-parameter \
  --name "/insightgen/openbao/unseal-key-1" \
  --with-decryption --query 'Parameter.Value' --output text)

KEY2=$(aws ssm get-parameter \
  --name "/insightgen/openbao/unseal-key-2" \
  --with-decryption --query 'Parameter.Value' --output text)

# Check if already unsealed
STATUS=$(docker exec insightgen-openbao bao status -format=json 2>/dev/null \
  | python3 -c "import sys,json; print(json.load(sys.stdin).get('sealed', True))")

if [ "$STATUS" = "False" ]; then
  echo "OpenBao is already unsealed"
  exit 0
fi

docker exec -e BAO_ADDR=$BAO_ADDR insightgen-openbao bao operator unseal $KEY1
docker exec -e BAO_ADDR=$BAO_ADDR insightgen-openbao bao operator unseal $KEY2

echo "OpenBao unsealed at $(date)"
SCRIPT

chmod +x /home/ec2-user/scripts/unseal-openbao.sh
```

Store unseal keys in SSM:
```bash
# Store first two unseal keys in SSM (never store all three together)
aws ssm put-parameter \
  --name "/insightgen/openbao/unseal-key-1" \
  --value "$UNSEAL_KEY_1" \
  --type SecureString \
  --key-id YOUR_KMS_KEY_ID \
  --overwrite

aws ssm put-parameter \
  --name "/insightgen/openbao/unseal-key-2" \
  --value "$UNSEAL_KEY_2" \
  --type SecureString \
  --key-id YOUR_KMS_KEY_ID \
  --overwrite

echo "Unseal keys stored in SSM"
echo "Store UNSEAL_KEY_3 and ROOT_TOKEN somewhere else (password manager)"
```

Add unseal to Docker Compose healthcheck startup order:
```bash
# Add to crontab — runs on reboot
echo "@reboot sleep 30 && /home/ec2-user/scripts/unseal-openbao.sh" | crontab -
```

---

### Step 5 — Store PostgreSQL Secrets in OpenBao (AWS)

```bash
export BAO_ADDR="http://127.0.0.1:8200"
export BAO_TOKEN="$ROOT_TOKEN"   # Use root token only for setup

# Enable KV version 2 secrets engine at the insightgen path
docker exec \
  -e BAO_ADDR=$BAO_ADDR \
  -e BAO_TOKEN=$BAO_TOKEN \
  insightgen-openbao \
  bao secrets enable -path=insightgen kv-v2

echo "KV secrets engine enabled at path: insightgen"

# Store all PostgreSQL credentials as a single secret
# This is more efficient than one secret per key
docker exec \
  -e BAO_ADDR=$BAO_ADDR \
  -e BAO_TOKEN=$BAO_TOKEN \
  insightgen-openbao \
  bao kv put insightgen/postgres \
    host="YOUR_AWS_POSTGRES_HOST" \
    port="5432" \
    dbname="insightgen" \
    username="insightgen_app" \
    password="YOUR_ACTUAL_PASSWORD"

# Verify it was stored
docker exec \
  -e BAO_ADDR=$BAO_ADDR \
  -e BAO_TOKEN=$BAO_TOKEN \
  insightgen-openbao \
  bao kv get insightgen/postgres
```

---

### Step 6 — Create a Scoped Policy and Application Token

Never use the root token in your application. Create a policy that allows only reading the postgres secret, then create a token bound to that policy.

```bash
# Create policy file
cat > /tmp/insightgen-policy.hcl << 'EOF'
# Allow reading the postgres secret and its metadata
path "insightgen/data/postgres" {
  capabilities = ["read"]
}

path "insightgen/metadata/postgres" {
  capabilities = ["read"]
}

# Allow token to renew itself
path "auth/token/renew-self" {
  capabilities = ["update"]
}

# Allow token to look up its own properties
path "auth/token/lookup-self" {
  capabilities = ["read"]
}
EOF

# Write the policy to OpenBao
docker exec \
  -e BAO_ADDR=$BAO_ADDR \
  -e BAO_TOKEN=$BAO_TOKEN \
  -i insightgen-openbao \
  bao policy write insightgen-readonly - < /tmp/insightgen-policy.hcl

echo "Policy created: insightgen-readonly"

# Create a long-lived token for Lambda to use
APP_TOKEN=$(docker exec \
  -e BAO_ADDR=$BAO_ADDR \
  -e BAO_TOKEN=$BAO_TOKEN \
  insightgen-openbao \
  bao token create \
    -policy=insightgen-readonly \
    -ttl=8760h \
    -renewable=true \
    -display-name="insightgen-lambda-aws" \
    -format=json \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['auth']['client_token'])")

echo "Application token created: ${APP_TOKEN:0:8}..."
```

Store the application token in SSM (the bootstrap secret):
```bash
aws ssm put-parameter \
  --name "/insightgen/openbao/app-token" \
  --value "$APP_TOKEN" \
  --type SecureString \
  --key-id YOUR_KMS_KEY_ID \
  --overwrite

echo "App token stored in SSM at /insightgen/openbao/app-token"
```

---

## Section 2 — Deploy OpenBao on GCP

Deploy a second OpenBao instance on a GCP VM. This instance holds the same secrets for the GCP side.

### Step 1 — Create a GCP VM for OpenBao

```bash
# Create a small VM — OpenBao is lightweight
gcloud compute instances create insightgen-openbao \
  --project $PROJECT_ID \
  --zone ${REGION}-a \
  --machine-type e2-small \
  --image-family debian-12 \
  --image-project debian-cloud \
  --service-account $SA_EMAIL \
  --scopes cloud-platform \
  --tags openbao-server \
  --metadata startup-script='#!/bin/bash
apt-get update -y
apt-get install -y docker.io
systemctl start docker
systemctl enable docker'
```

### Step 2 — Install and Configure OpenBao on GCP VM

SSH into the GCP VM:
```bash
gcloud compute ssh insightgen-openbao \
  --project $PROJECT_ID \
  --zone ${REGION}-a
```

On the GCP VM, run identical setup to AWS:

```bash
# Create config directory
mkdir -p /opt/openbao/config /opt/openbao/data /opt/openbao/logs

cat > /opt/openbao/config/config.hcl << 'EOF'
ui = false

listener "tcp" {
  address   = "0.0.0.0:8200"
  tls_disable = true
}

storage "file" {
  path = "/openbao/data"
}

default_lease_ttl = "768h"
max_lease_ttl     = "8760h"
EOF

# Run OpenBao container
docker run -d \
  --name insightgen-openbao \
  --restart unless-stopped \
  --cap-add IPC_LOCK \
  -p 127.0.0.1:8200:8200 \
  -v /opt/openbao/config:/openbao/config:ro \
  -v /opt/openbao/data:/openbao/data \
  -v /opt/openbao/logs:/openbao/logs \
  openbao/openbao:2.0.3 \
  server -config=/openbao/config/config.hcl

sleep 5

export BAO_ADDR="http://127.0.0.1:8200"

# Initialise
bao operator init \
  -key-shares=3 \
  -key-threshold=2 \
  -format=json > /tmp/openbao-init.json

cat /tmp/openbao-init.json

# Save the output — extract and unseal
UNSEAL_KEY_1=$(cat /tmp/openbao-init.json | python3 -c "import sys,json; print(json.load(sys.stdin)['unseal_keys_b64'][0])")
UNSEAL_KEY_2=$(cat /tmp/openbao-init.json | python3 -c "import sys,json; print(json.load(sys.stdin)['unseal_keys_b64'][1])")
ROOT_TOKEN=$(cat /tmp/openbao-init.json | python3 -c "import sys,json; print(json.load(sys.stdin)['root_token'])")

bao operator unseal $UNSEAL_KEY_1
bao operator unseal $UNSEAL_KEY_2

bao status
```

### Step 3 — Store Secrets in GCP OpenBao

On the GCP VM (still SSH'd in):

```bash
export BAO_ADDR="http://127.0.0.1:8200"
export BAO_TOKEN="$ROOT_TOKEN"

# Enable KV engine
bao secrets enable -path=insightgen kv-v2

# Store GCP PostgreSQL credentials
bao kv put insightgen/postgres \
  host="YOUR_GCP_POSTGRES_HOST" \
  port="5432" \
  dbname="insightgen" \
  username="insightgen_app" \
  password="YOUR_GCP_POSTGRES_PASSWORD"

# Create the same scoped policy
cat > /tmp/insightgen-policy.hcl << 'EOF'
path "insightgen/data/postgres" {
  capabilities = ["read"]
}
path "insightgen/metadata/postgres" {
  capabilities = ["read"]
}
path "auth/token/renew-self" {
  capabilities = ["update"]
}
path "auth/token/lookup-self" {
  capabilities = ["read"]
}
EOF

bao policy write insightgen-readonly /tmp/insightgen-policy.hcl

# Create application token for Cloud Functions
APP_TOKEN=$(bao token create \
  -policy=insightgen-readonly \
  -ttl=8760h \
  -renewable=true \
  -display-name="insightgen-cloudfn-gcp" \
  -format=json \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['auth']['client_token'])")

echo "GCP App Token: ${APP_TOKEN:0:8}..."
```

Store the GCP application token in GCP Secret Manager (bootstrap secret):
```bash
# Exit SSH back to your laptop and run this
echo -n "$APP_TOKEN" | \
  gcloud secrets versions add insightgen-openbao-token \
  --data-file=- --project $PROJECT_ID
```

Create the GCP secret first if it does not exist:
```bash
gcloud secrets create insightgen-openbao-token \
  --project $PROJECT_ID \
  --replication-policy user-managed \
  --locations $REGION \
  --kms-key-name $KMS_KEY

echo -n "YOUR_GCP_APP_TOKEN" | \
  gcloud secrets versions add insightgen-openbao-token \
  --data-file=- --project $PROJECT_ID
```

### Step 4 — Store Unseal Keys in GCP Secret Manager

```bash
# Store two of the three GCP unseal keys
echo -n "$GCP_UNSEAL_KEY_1" | \
  gcloud secrets versions add insightgen-openbao-unseal-key-1 \
  --data-file=- --project $PROJECT_ID

echo -n "$GCP_UNSEAL_KEY_2" | \
  gcloud secrets versions add insightgen-openbao-unseal-key-2 \
  --data-file=- --project $PROJECT_ID
```

Create an unseal script on the GCP VM:
```bash
cat > /opt/openbao/unseal.sh << 'SCRIPT'
#!/bin/bash
export BAO_ADDR="http://127.0.0.1:8200"

# Fetch unseal keys from GCP Secret Manager
PROJECT_ID=$(curl -s "http://metadata.google.internal/computeMetadata/v1/project/project-id" \
  -H "Metadata-Flavor: Google")

get_secret() {
  curl -s \
    -H "Authorization: Bearer $(gcloud auth print-access-token)" \
    "https://secretmanager.googleapis.com/v1/projects/${PROJECT_ID}/secrets/${1}/versions/latest:access" \
    | python3 -c "import sys,json,base64; d=json.load(sys.stdin); print(base64.b64decode(d['payload']['data']).decode())"
}

STATUS=$(bao status -format=json 2>/dev/null | python3 -c "import sys,json; print(json.load(sys.stdin).get('sealed', True))" 2>/dev/null)

if [ "$STATUS" = "False" ]; then
  echo "OpenBao already unsealed"
  exit 0
fi

KEY1=$(get_secret "insightgen-openbao-unseal-key-1")
KEY2=$(get_secret "insightgen-openbao-unseal-key-2")

bao operator unseal $KEY1
bao operator unseal $KEY2

echo "OpenBao unsealed at $(date)"
SCRIPT

chmod +x /opt/openbao/unseal.sh

# Add to crontab
echo "@reboot sleep 30 && /opt/openbao/unseal.sh" | crontab -
```

---

## Section 3 — Python Implementation

### File: `secrets_client.py`

This is the entire cloud-specific file. It handles both the bootstrap (fetching the OpenBao token from the native cloud store) and the actual secret retrieval from OpenBao.

```python
"""
secrets_client.py — OpenBao edition

Fetches PostgreSQL credentials from OpenBao.
The OpenBao token itself is bootstrapped from the native cloud secret store:
  - AWS: SSM Parameter Store at /insightgen/openbao/app-token
  - GCP: Secret Manager at insightgen-openbao-token

Only VAULT_ADDR and CLOUD_PROVIDER differ between AWS and GCP deployments.
All business logic in main.py is identical on both clouds.
"""

import os
import logging
from functools import lru_cache

logger = logging.getLogger(__name__)

CLOUD_PROVIDER = os.environ.get('CLOUD_PROVIDER', 'AWS').upper()
VAULT_ADDR = os.environ.get('VAULT_ADDR')          # Set per deployment
VAULT_SECRET_PATH = 'insightgen/postgres'           # Same on both OpenBao instances
REQUIRED_KEYS = {'host', 'port', 'dbname', 'username', 'password'}


# ─── Bootstrap: Get OpenBao Token from Native Cloud Store ─────────────────────

def _get_token_from_aws() -> str:
    """
    Fetch the OpenBao application token from AWS SSM Parameter Store.
    This is the only AWS SDK call made in the AWS deployment.
    """
    import boto3
    from botocore.exceptions import ClientError

    region = os.environ.get('AWS_REGION', 'us-east-1')
    ssm = boto3.client('ssm', region_name=region)
    param_name = '/insightgen/openbao/app-token'

    try:
        response = ssm.get_parameter(Name=param_name, WithDecryption=True)
        token = response['Parameter']['Value']
        logger.info("OpenBao token retrieved from AWS SSM")
        return token
    except ClientError as e:
        code = e.response['Error']['Code']
        raise RuntimeError(
            f"Failed to fetch OpenBao token from SSM ({param_name}): {code}"
        ) from e


def _get_token_from_gcp() -> str:
    """
    Fetch the OpenBao application token from GCP Secret Manager.
    This is the only GCP SDK call made in the GCP deployment.
    """
    from google.cloud import secretmanager
    from google.api_core.exceptions import GoogleAPIError

    project_id = os.environ.get('GCP_PROJECT_ID')
    if not project_id:
        raise ValueError("GCP_PROJECT_ID environment variable is required")

    client = secretmanager.SecretManagerServiceClient()
    secret_name = f"projects/{project_id}/secrets/insightgen-openbao-token/versions/latest"

    try:
        response = client.access_secret_version(request={"name": secret_name})
        token = response.payload.data.decode("UTF-8")
        logger.info("OpenBao token retrieved from GCP Secret Manager")
        return token
    except GoogleAPIError as e:
        raise RuntimeError(
            f"Failed to fetch OpenBao token from GCP Secret Manager: {e}"
        ) from e


@lru_cache(maxsize=1)
def _get_vault_token() -> str:
    """
    Retrieve the OpenBao application token from the native cloud secret store.
    Cached — fetches once per cold start.
    """
    if CLOUD_PROVIDER == 'AWS':
        return _get_token_from_aws()
    elif CLOUD_PROVIDER == 'GCP':
        return _get_token_from_gcp()
    else:
        raise ValueError(
            f"Unsupported CLOUD_PROVIDER: '{CLOUD_PROVIDER}'. Must be AWS or GCP."
        )


# ─── OpenBao Secret Retrieval ─────────────────────────────────────────────────

@lru_cache(maxsize=1)
def get_postgres_config() -> dict:
    """
    Fetch PostgreSQL credentials from OpenBao.
    
    Flow:
      1. Get OpenBao token from native cloud store (SSM or Secret Manager)
      2. Authenticate to OpenBao using that token
      3. Read the postgres secret from insightgen/postgres path
      4. Return as dict — cached for entire Lambda/Cloud Function instance lifetime
    
    Returns:
        dict with keys: host, port, dbname, username, password
    """
    if not VAULT_ADDR:
        raise ValueError(
            "VAULT_ADDR environment variable is required. "
            "Set to http://<openbao-host>:8200"
        )

    try:
        import hvac
    except ImportError:
        raise ImportError(
            "hvac library is required. Add 'hvac' to requirements.txt "
            "and rebuild your deployment package."
        )

    token = _get_vault_token()

    try:
        client = hvac.Client(url=VAULT_ADDR, token=token)
    except Exception as e:
        raise RuntimeError(f"Failed to create OpenBao client for {VAULT_ADDR}: {e}") from e

    if not client.is_authenticated():
        raise RuntimeError(
            f"OpenBao token is not valid or has expired. "
            f"Rotate the token at /insightgen/openbao/app-token and redeploy."
        )

    try:
        response = client.secrets.kv.v2.read_secret_version(
            path='postgres',
            mount_point='insightgen'
        )
        secrets = response['data']['data']
    except hvac.exceptions.Forbidden:
        raise RuntimeError(
            "OpenBao token does not have permission to read insightgen/data/postgres. "
            "Verify the token was created with the insightgen-readonly policy."
        )
    except hvac.exceptions.InvalidPath:
        raise RuntimeError(
            f"Secret path insightgen/postgres not found in OpenBao at {VAULT_ADDR}. "
            "Verify the secret was created with: bao kv put insightgen/postgres ..."
        )
    except Exception as e:
        raise RuntimeError(f"OpenBao secret read failed: {e}") from e

    missing = REQUIRED_KEYS - set(secrets.keys())
    if missing:
        raise KeyError(
            f"OpenBao secret insightgen/postgres is missing keys: {missing}. "
            f"Re-run: bao kv put insightgen/postgres host=... port=... dbname=... "
            f"username=... password=..."
        )

    logger.info(
        f"PostgreSQL config fetched from OpenBao at {VAULT_ADDR} "
        f"({CLOUD_PROVIDER} deployment)"
    )
    return secrets


def get_postgres_dsn() -> str:
    """
    Returns a libpq DSN string for psycopg2 with SSL enforced.
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
    """
    cfg = get_postgres_config()
    return (
        f"postgresql+psycopg2://{cfg['username']}:{cfg['password']}"
        f"@{cfg['host']}:{cfg['port']}/{cfg['dbname']}"
        f"?sslmode=require"
    )
```

---

### File: `main.py`

Identical to the cloud-native version. No changes:

```python
"""
main.py — Lambda handler and GCP Cloud Function entry point.
Identical on both AWS and GCP — no cloud-specific code here.
"""

import json
import logging
import psycopg2
from secrets_client import get_postgres_dsn

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

_db_conn = None


def _get_db_connection():
    global _db_conn
    if _db_conn is None or _db_conn.closed != 0:
        logger.info("Creating new PostgreSQL connection")
        _db_conn = psycopg2.connect(get_postgres_dsn())
        _db_conn.autocommit = True
    return _db_conn


def lambda_handler(event, context):
    """AWS Lambda entry point."""
    try:
        conn = _get_db_connection()
        with conn.cursor() as cur:
            cur.execute("SELECT version(), current_database()")
            row = cur.fetchone()
        return {
            'statusCode': 200,
            'body': json.dumps({
                'status': 'connected',
                'cloud': 'AWS',
                'secrets_backend': 'OpenBao',
                'db_version': row[0],
                'database': row[1]
            })
        }
    except Exception as e:
        logger.error(f"Lambda error: {e}")
        return {'statusCode': 500, 'body': json.dumps({'error': str(e)})}


def cloud_function_handler(request):
    """GCP Cloud Function entry point."""
    import flask
    try:
        conn = _get_db_connection()
        with conn.cursor() as cur:
            cur.execute("SELECT version(), current_database()")
            row = cur.fetchone()
        return flask.Response(
            json.dumps({
                'status': 'connected',
                'cloud': 'GCP',
                'secrets_backend': 'OpenBao',
                'db_version': row[0],
                'database': row[1]
            }),
            status=200,
            mimetype='application/json'
        )
    except Exception as e:
        logger.error(f"Cloud Function error: {e}")
        return flask.Response(
            json.dumps({'error': str(e)}),
            status=500,
            mimetype='application/json'
        )
```

---

### File: `requirements.txt`

OpenBao uses the `hvac` Vault client library — same package for both clouds.

**`requirements-aws.txt`** (Lambda layer):
```
psycopg2-binary==2.9.9
boto3==1.34.0
hvac==2.1.0
```

**`requirements-gcp.txt`** (Cloud Function package):
```
psycopg2-binary==2.9.9
google-cloud-secret-manager==2.20.0
hvac==2.1.0
flask==3.0.0
functions-framework==3.5.0
```

---

## Section 4 — Deployment

### AWS Lambda — Environment Variables

```bash
FUNCTION_NAME="dev-api-configurator"

# Get internal EC2 IP (OpenBao is on the same EC2 instance)
EC2_PRIVATE_IP=$(aws ec2 describe-instances \
  --filters "Name=tag:Project,Values=insightgen" \
  --query 'Reservations[0].Instances[0].PrivateIpAddress' \
  --output text)

aws lambda update-function-configuration \
  --function-name $FUNCTION_NAME \
  --environment "Variables={
    CLOUD_PROVIDER=AWS,
    VAULT_ADDR=http://${EC2_PRIVATE_IP}:8200,
    APP_ENV=dev,
    APP_VERSION=mvp3
  }"
```

Note: Lambda and OpenBao are both inside the VPC. The Lambda connects to OpenBao via the EC2 private IP on port 8200. Ensure the EC2 security group allows inbound traffic on port 8200 from the Lambda security group.

### GCP Cloud Function — Environment Variables

```bash
# Get GCP VM internal IP
GCP_VM_IP=$(gcloud compute instances describe insightgen-openbao \
  --project $PROJECT_ID \
  --zone ${REGION}-a \
  --format='value(networkInterfaces[0].networkIP)')

gcloud functions deploy insightgen-api-configurator \
  --project $PROJECT_ID \
  --region $REGION \
  --runtime python312 \
  --trigger-http \
  --entry-point cloud_function_handler \
  --service-account $SA_EMAIL \
  --set-env-vars \
    CLOUD_PROVIDER=GCP,\
    GCP_PROJECT_ID=${PROJECT_ID},\
    VAULT_ADDR=http://${GCP_VM_IP}:8200,\
    APP_ENV=dev,\
    APP_VERSION=mvp3 \
  --source .
```

---

## Section 5 — OpenBao Policy Reference

### View Current Policy

```bash
export BAO_ADDR="http://127.0.0.1:8200"
export BAO_TOKEN="your-root-token"

# List all policies
bao policy list

# View the insightgen-readonly policy
bao policy read insightgen-readonly
```

### Rotate the Application Token

Token rotation should be done before the TTL expires. Create a new token, update SSM/Secret Manager, then deploy functions to pick it up:

```bash
# Create new token with the same policy
NEW_TOKEN=$(bao token create \
  -policy=insightgen-readonly \
  -ttl=8760h \
  -renewable=true \
  -display-name="insightgen-lambda-aws-rotated" \
  -format=json \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['auth']['client_token'])")

# Update SSM (AWS)
aws ssm put-parameter \
  --name "/insightgen/openbao/app-token" \
  --value "$NEW_TOKEN" \
  --type SecureString \
  --key-id YOUR_KMS_KEY_ID \
  --overwrite

# Update GCP Secret Manager
echo -n "$NEW_TOKEN" | \
  gcloud secrets versions add insightgen-openbao-token \
  --data-file=- --project $PROJECT_ID

echo "Token rotated. Redeploy Lambda and Cloud Function to pick up new token."
```

### Audit Secret Access

OpenBao has an audit log feature. Enable it to see every secret access:

```bash
# Enable file audit log
bao audit enable file file_path=/openbao/logs/audit.log

# View recent access
docker exec insightgen-openbao tail -f /openbao/logs/audit.log | python3 -m json.tool
```

---

## Section 6 — Testing

### Test OpenBao is Running and Accessible

```bash
# From EC2 (AWS)
curl -s http://127.0.0.1:8200/v1/sys/health | python3 -m json.tool

# From another machine inside VPC (replace with EC2 private IP)
curl -s http://EC2_PRIVATE_IP:8200/v1/sys/health | python3 -m json.tool
```

Expected response: `"sealed": false, "initialized": true`

### Test Secret Read With Application Token

```bash
# Test the application token can read the secret
APP_TOKEN="your-application-token"

curl -s \
  -H "X-Vault-Token: $APP_TOKEN" \
  http://127.0.0.1:8200/v1/insightgen/data/postgres \
  | python3 -m json.tool
```

Expected: JSON response containing `data.data` with your PostgreSQL credentials.

### Test Python Client (AWS)

```python
# test_openbao_aws.py
import os
os.environ['CLOUD_PROVIDER'] = 'AWS'
os.environ['VAULT_ADDR'] = 'http://EC2_PRIVATE_IP:8200'

from secrets_client import get_postgres_config, get_postgres_dsn

try:
    config = get_postgres_config()
    print("OpenBao fetch (AWS): PASS")
    print(f"  Host:     {config['host']}")
    print(f"  Port:     {config['port']}")
    print(f"  Database: {config['dbname']}")
    print(f"  User:     {config['username']}")
    print(f"  Password: {'*' * len(config['password'])}")
except Exception as e:
    print(f"OpenBao fetch (AWS): FAIL — {e}")
```

### Test Python Client (GCP)

```python
# test_openbao_gcp.py
import os
os.environ['CLOUD_PROVIDER'] = 'GCP'
os.environ['GCP_PROJECT_ID'] = 'your-project-id'
os.environ['VAULT_ADDR'] = 'http://GCP_VM_PRIVATE_IP:8200'

from secrets_client import get_postgres_config

try:
    config = get_postgres_config()
    print("OpenBao fetch (GCP): PASS")
    print(f"  Host:     {config['host']}")
    print(f"  Port:     {config['port']}")
    print(f"  Database: {config['dbname']}")
    print(f"  User:     {config['username']}")
    print(f"  Password: {'*' * len(config['password'])}")
except Exception as e:
    print(f"OpenBao fetch (GCP): FAIL — {e}")
```

---

## Section 7 — Troubleshooting

### OpenBao is Sealed After Restart

This is expected behaviour. OpenBao seals itself on every restart to protect secrets in memory.

```bash
# Check seal status
docker exec insightgen-openbao bao status

# Unseal manually (or run the unseal script)
/home/ec2-user/scripts/unseal-openbao.sh
```

### hvac.exceptions.InvalidPath

```
hvac.exceptions.InvalidPath: None
```

The path `insightgen/postgres` does not exist. Verify:
```bash
docker exec -e BAO_ADDR=http://127.0.0.1:8200 -e BAO_TOKEN=YOUR_TOKEN \
  insightgen-openbao bao kv get insightgen/postgres
```

If nothing is returned, the secret was not stored. Re-run step 5 of the AWS or GCP setup.

### hvac.exceptions.Forbidden

```
hvac.exceptions.Forbidden: 1 error occurred: permission denied
```

The application token does not have access to the path. Verify the token has the `insightgen-readonly` policy:

```bash
docker exec -e BAO_ADDR=http://127.0.0.1:8200 -e BAO_TOKEN=ROOT_TOKEN \
  insightgen-openbao bao token lookup APP_TOKEN
```

Check the `policies` field in the output — it should include `insightgen-readonly`.

### Lambda Cannot Reach OpenBao

Lambda is inside VPC. OpenBao is on EC2 at port 8200, bound to all interfaces inside the container but may be blocked by security groups.

```bash
# Check EC2 security group allows port 8200 from Lambda security group
aws ec2 describe-security-groups \
  --filters "Name=tag:Project,Values=insightgen" \
  --query 'SecurityGroups[*].IpPermissions[?FromPort==`8200`]'
```

If no rule exists, request the networking team to add:
- Inbound TCP 8200 from the Lambda security group to the EC2 security group.

### Token Expired

OpenBao tokens have a TTL. When a token expires, all Lambda invocations fail until the token is rotated and the functions redeployed.

Monitor token expiry proactively:
```bash
# Check how long the app token has remaining
docker exec -e BAO_ADDR=http://127.0.0.1:8200 -e BAO_TOKEN=ROOT_TOKEN \
  insightgen-openbao bao token lookup APP_TOKEN \
  | grep -E "ttl|expire_time|creation_time"
```

Set a calendar reminder to rotate the token before TTL expiry. At 8760h (1 year), rotate it annually.

---

## Section 8 — Comparison Summary

| Factor | Cloud-Native | OpenBao |
|---|---|---|
| Python code difference | `secrets_client.py` has two backends | `secrets_client.py` is fully unified |
| Extra infrastructure | None | OpenBao container per cloud (2 total) |
| Extra operational burden | None | Unseal on restart, token rotation, audit |
| Bootstrap problem | None (IAM-native auth) | Token stored in SSM/Secret Manager |
| If OpenBao crashes | N/A | Lambda and Cloud Function cannot start |
| Adding a 3rd cloud | Add new backend function in `secrets_client.py` | Add new OpenBao instance, change `VAULT_ADDR` |
| Policy model | SSM/KMS (AWS) + IAM (GCP) — two systems | OpenBao policies — one system |
| Secret rotation | Update SSM or Secret Manager natively | Update one OpenBao instance, sync to others |
| Best for POC | Yes — simpler | Evaluate only |
| Best for 3+ clouds | No | Yes |
| Python library | `boto3` + `google-cloud-secret-manager` | `hvac` only |
