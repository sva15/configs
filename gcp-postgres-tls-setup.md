# Task — Enable PostgreSQL TLS (Encryption in Transit) on GCP
**Project:** InsightGen  
**Environment:** GCP  
**Author:** fitindevops  
**Date:** March 2026

---

## Context

PostgreSQL runs as a Docker container on a private GCP VM. Cloud Run connects to it to fetch application data. The Cloud Run code had `sslmode=require` removed as a workaround because PostgreSQL on GCP did not have SSL configured. This task enables SSL on the PostgreSQL container and then upgrades the connection from `sslmode=require` to `sslmode=verify-ca`.

**GCP at-rest encryption:** The GCP VM disk is encrypted by default managed by Google. This task covers in-transit encryption only.

---

## SSL Mode Comparison — Why verify-ca Is Better

| Mode | What it does | Protects against |
|---|---|---|
| `sslmode=require` | Enforces TLS encryption on the connection | Passive eavesdropping — nobody can read the traffic |
| `sslmode=verify-ca` | Enforces TLS + verifies the server's certificate against a trusted CA | Eavesdropping + man-in-the-middle — confirms you are talking to the real PostgreSQL server |
| `sslmode=verify-full` | Enforces TLS + verifies CA + verifies hostname matches cert | Everything above + DNS spoofing |

For an internal private VPC where the connection goes directly to a known IP, `verify-ca` is the right level. The server certificate is verified to be signed by your CA — it is not possible for another server to impersonate your PostgreSQL container unless it has your private CA key. `verify-full` requires the CN in the certificate to match the hostname used in the connection string, which adds complexity for container deployments.

**Today's implementation plan:**
1. Generate certificates and enable SSL on PostgreSQL container (`sslmode=require` working)
2. Store `ca.crt` in GCP Secret Manager
3. Update Cloud Run code to fetch `ca.crt` and use `sslmode=verify-ca`
4. Test and confirm

---

## Understanding Secret Rotation with a Self-Managed Container

Before the implementation, it is worth understanding what managed rotation can and cannot do for a containerised PostgreSQL.

**Managed databases (AWS RDS, GCP Cloud SQL):**
Secrets Manager calls a cloud API to rotate the password. The cloud service updates the database internally. Fully automated.

**Your PostgreSQL container:**
Secret rotation is possible but you own the rotation logic. A Lambda or Cloud Run job must connect to PostgreSQL directly and run `ALTER USER insightgen_app PASSWORD 'new_password'`, then update the secret store. The database does not care that it runs in a container — PostgreSQL is PostgreSQL. The rotation Lambda you already built (`insightgen-password-rotation`) does exactly this.

**TLS certificate rotation on a container:**
GCP Certificate Authority Service and AWS ACM cannot reach inside a Docker container to update certificate files on disk. Certificate rotation on a self-managed container requires: generate new cert, copy into container, reload PostgreSQL. This is a manual or scripted process. The practical approach is to use long-validity certificates (10 years) so rotation is infrequent. When rotation is needed, the process is: generate new CA and server cert, copy into container, update the `ca.crt` secret version in Secret Manager, and redeploy Cloud Run so it fetches the new CA at cold start.

**Where to store `ca.crt` for verify-ca:**
Secret Manager is the right place. It is already integrated into your code, the service account already has access, and it keeps all certificate material alongside credentials in one governed place. No files to bundle into container images, no S3 buckets to manage.

---

## Key Lesson from AWS — Read Before Starting

OpenSSL 3.x generates private keys in PKCS#8 format by default:
```
-----BEGIN PRIVATE KEY-----
```

PostgreSQL requires RSA format:
```
-----BEGIN RSA PRIVATE KEY-----
```

Using `openssl genrsa` directly (as shown in Step 3) produces RSA format and avoids the problem. If the wrong format is used, PostgreSQL crash-loops with:
```
FATAL: could not load private key file "server.key": unsupported
```

---

## Prerequisites

- SSH access to the GCP VM via `gcloud compute ssh`
- PostgreSQL container name (confirm with `docker ps`)
- Cloud Run function deployed and connecting to PostgreSQL
- `PROJECT_ID`, `REGION`, `ZONE` shell variables set

---

## Step 1 — Connect to the GCP VM

```bash
gcloud compute ssh YOUR_GCP_VM_NAME \
  --project $PROJECT_ID \
  --zone $ZONE
```

---

## Step 2 — Check OpenSSL Version

```bash
openssl version
```

If it shows OpenSSL 3.x, follow the `genrsa` commands exactly in Step 3 to get RSA format directly.

---

## Step 3 — Generate CA and Server Certificates

```bash
mkdir -p /opt/postgres-ssl
cd /opt/postgres-ssl

# Generate CA private key — genrsa produces RSA format directly
openssl genrsa -out ca.key 4096

# Generate CA certificate — self-signed, 10 year validity
openssl req -new -x509 \
  -days 3650 \
  -key ca.key \
  -out ca.crt \
  -subj "/CN=InsightGen-GCP-CA/O=InsightGen/C=US"

# Generate server private key
openssl genrsa -out server.key 4096

# Verify RSA format — MUST show: -----BEGIN RSA PRIVATE KEY-----
head -1 server.key
```

If you see `-----BEGIN PRIVATE KEY-----` instead, convert before continuing:
```bash
openssl rsa -in server.key -out server.key.rsa && mv server.key.rsa server.key
```

Continue:
```bash
# Generate certificate signing request
openssl req -new \
  -key server.key \
  -out server.csr \
  -subj "/CN=postgres/O=InsightGen/C=US"

# Sign the server certificate with the CA
openssl x509 -req \
  -days 3650 \
  -in server.csr \
  -CA ca.crt \
  -CAkey ca.key \
  -CAcreateserial \
  -out server.crt

# Verify certificate chain
openssl verify -CAfile ca.crt server.crt
```

Expected:
```
server.crt: OK
```

Set permissions:
```bash
chmod 600 server.key
chmod 644 server.crt ca.crt

ls -la /opt/postgres-ssl/
```

---

## Step 4 — Find PostgreSQL Container Name and Set Variables

```bash
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Status}}" | grep -i postgres

# Set these for use in remaining steps
CONTAINER_NAME="insightgen-postgres"  # Update if different
DB_USER="insightgen_app"              # Update to your DB app username
DB_NAME="insightgen"                  # Update to your DB name
```

---

## Step 5 — Copy Certificates Into the Running Container

```bash
docker cp /opt/postgres-ssl/ca.crt     $CONTAINER_NAME:/var/lib/postgresql/ca.crt
docker cp /opt/postgres-ssl/server.crt $CONTAINER_NAME:/var/lib/postgresql/server.crt
docker cp /opt/postgres-ssl/server.key $CONTAINER_NAME:/var/lib/postgresql/server.key

# Fix ownership
docker exec $CONTAINER_NAME chown postgres:postgres \
  /var/lib/postgresql/ca.crt \
  /var/lib/postgresql/server.crt \
  /var/lib/postgresql/server.key

# Fix permissions
docker exec $CONTAINER_NAME chmod 600 /var/lib/postgresql/server.key
docker exec $CONTAINER_NAME chmod 644 /var/lib/postgresql/server.crt
docker exec $CONTAINER_NAME chmod 644 /var/lib/postgresql/ca.crt

# Confirm — key must be 600, owner must be postgres
docker exec $CONTAINER_NAME ls -la /var/lib/postgresql/
```

---

## Step 6 — Enable SSL in PostgreSQL Configuration

```bash
# Enable SSL in postgresql.conf
docker exec $CONTAINER_NAME bash -c "cat >> /var/lib/postgresql/data/postgresql.conf << 'EOF'

# SSL Configuration — encryption in transit
ssl = on
ssl_ca_file = '/var/lib/postgresql/ca.crt'
ssl_cert_file = '/var/lib/postgresql/server.crt'
ssl_key_file = '/var/lib/postgresql/server.key'
ssl_min_protocol_version = 'TLSv1.2'
EOF"

# View current pg_hba.conf before editing
docker exec $CONTAINER_NAME cat /var/lib/postgresql/data/pg_hba.conf

# Add hostssl rule — replace 10.0.0.0/8 with your GCP VPC subnet CIDR
docker exec $CONTAINER_NAME bash -c "cat >> /var/lib/postgresql/data/pg_hba.conf << 'EOF'

# Require SSL for all remote connections
hostssl  all  all  10.0.0.0/8  scram-sha-256
EOF"
```

---

## Step 7 — Restart the Container

```bash
docker stop $CONTAINER_NAME
docker start $CONTAINER_NAME
sleep 15

# Confirm running — no crash loop
docker ps | grep $CONTAINER_NAME

# Check logs
docker logs $CONTAINER_NAME --tail 30
```

Successful startup shows:
```
LOG:  database system is ready to accept connections
```

**If you see** `FATAL: could not load private key file "server.key": unsupported`:
```bash
docker cp $CONTAINER_NAME:/var/lib/postgresql/server.key /tmp/server.key
openssl rsa -in /tmp/server.key -out /tmp/server_rsa.key
docker cp /tmp/server_rsa.key $CONTAINER_NAME:/var/lib/postgresql/server.key
docker exec $CONTAINER_NAME chown postgres:postgres /var/lib/postgresql/server.key
docker exec $CONTAINER_NAME chmod 600 /var/lib/postgresql/server.key
docker restart $CONTAINER_NAME
sleep 15
docker logs $CONTAINER_NAME --tail 20
```

---

## Step 8 — Verify SSL Is Active in PostgreSQL

```bash
# Confirm ssl=on
docker exec $CONTAINER_NAME \
  psql -U $DB_USER -d $DB_NAME \
  -c "SHOW ssl;"

# Confirm TLS version and cipher
docker exec $CONTAINER_NAME \
  psql -U $DB_USER -d $DB_NAME \
  -c "SELECT ssl, version, cipher FROM pg_stat_ssl WHERE pid = pg_backend_pid();"
```

Expected:
```
 ssl | version |           cipher
-----+---------+-----------------------------
 t   | TLSv1.3 | TLS_AES_256_GCM_SHA384
```

---

## Step 9 — Make Certificates Persistent via Docker Compose

Update `docker-compose.yml` to mount certificates from the VM host path:

```yaml
services:
  postgres:
    image: postgres:15.4-alpine3.18
    container_name: insightgen-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgres-data:/var/lib/postgresql/data
      # SSL certificates — mounted from GCP VM, persistent across container rebuilds
      - /opt/postgres-ssl/server.crt:/var/lib/postgresql/server.crt:ro
      - /opt/postgres-ssl/server.key:/var/lib/postgresql/server.key:ro
      - /opt/postgres-ssl/ca.crt:/var/lib/postgresql/ca.crt:ro
    networks:
      - insightgen-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5
```

```bash
docker compose up -d postgres
docker ps | grep $CONTAINER_NAME
docker logs $CONTAINER_NAME --tail 20
```

---

## Step 10 — Test with sslmode=require First

Before upgrading to `verify-ca`, confirm basic TLS connectivity works with `sslmode=require`. This validates that PostgreSQL SSL is working before adding certificate verification.

Update `secrets_client.py` temporarily:

```python
def get_postgres_dsn() -> str:
    cfg = get_postgres_config()
    return (
        f"host={cfg['host']} "
        f"port={cfg['port']} "
        f"dbname={cfg['dbname']} "
        f"user={cfg['username']} "
        f"password={cfg['password']} "
        f"sslmode=require"
    )
```

Push and deploy:
```bash
git add secrets_client.py
git commit -m "enable sslmode=require — PostgreSQL SSL configured"
git push origin main
```

Test from Cloud Shell:
```bash
curl -s "http://10.131.36.204/int/secrets" | python3 -m json.tool
```

Expected: `"status": "connected"`. If this works, PostgreSQL TLS is functioning. Now proceed to `verify-ca`.

---

## Step 11 — Store ca.crt in GCP Secret Manager

`verify-ca` requires Cloud Run to have the CA certificate available at runtime to verify the server's certificate against. The cleanest way to provide this is GCP Secret Manager — same place as the DB credentials, no files to bundle into the container image.

On the GCP VM, read the CA certificate content:
```bash
cat /opt/postgres-ssl/ca.crt
```

From your laptop or Cloud Shell, store it as a secret:
```bash
# Create the secret
gcloud secrets create insightgen-postgres-ca-cert \
  --project $PROJECT_ID \
  --replication-policy user-managed \
  --locations $REGION \
  --kms-key-name $KMS_KEY

# Store the CA certificate as a secret version
# Use the actual content from the cat command above
cat /opt/postgres-ssl/ca.crt | \
  gcloud secrets versions add insightgen-postgres-ca-cert \
  --data-file=- --project $PROJECT_ID

# Verify it was stored
gcloud secrets versions access latest \
  --secret insightgen-postgres-ca-cert \
  --project $PROJECT_ID
```

The output should show the full certificate content starting with `-----BEGIN CERTIFICATE-----`.

Grant the Cloud Run service account access if not already covered by project-level binding:
```bash
gcloud secrets add-iam-policy-binding insightgen-postgres-ca-cert \
  --project $PROJECT_ID \
  --member "serviceAccount:${SA_EMAIL}" \
  --role "roles/secretmanager.secretAccessor"
```

---

## Step 12 — Update secrets_client.py for verify-ca

`verify-ca` requires writing the CA certificate to a temporary file on disk because psycopg2 accepts the CA path as `sslrootcert` in the connection string. Cloud Run functions have a writable `/tmp` directory available at runtime.

Update `secrets_client.py`:

```python
"""
secrets_client.py
-----------------
Cloud-agnostic PostgreSQL credential and CA certificate retrieval.
Uses singleton pattern — loaded once at cold start, cached for instance lifetime.
"""

import os
import logging
import tempfile

logger = logging.getLogger(__name__)

_CLOUD_PROVIDER = os.environ.get('CLOUD_PROVIDER', 'AWS').upper()
_REQUIRED_KEYS = {'host', 'port', 'dbname', 'username', 'password'}

# Singletons — populated once at cold start
_postgres_config = None
_ca_cert_path = None          # Path to ca.crt written to /tmp at cold start


# ─── AWS SSM fetch ────────────────────────────────────────────────────────────

def _load_from_aws() -> dict:
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
        raise RuntimeError(
            f"SSM GetParametersByPath failed [{e.response['Error']['Code']}]. "
            f"Check Lambda role has ssm:GetParametersByPath on {prefix}*"
        ) from e

    if not response.get('Parameters'):
        raise RuntimeError(f"No parameters found at SSM path: {prefix}")

    result = {p['Name'].replace(prefix, ''): p['Value'] for p in response['Parameters']}
    logger.info(f"PostgreSQL config loaded from AWS SSM [params={list(result.keys())}]")
    return result


# ─── GCP Secret Manager fetch ─────────────────────────────────────────────────

def _load_from_gcp() -> dict:
    from google.cloud import secretmanager
    from google.api_core.exceptions import NotFound, PermissionDenied, GoogleAPIError

    project_id = os.environ.get('GCP_PROJECT_ID')
    if not project_id:
        raise ValueError("GCP_PROJECT_ID environment variable must be set.")

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
                f"Secret not found: insightgen-postgres-{key} in project {project_id}"
            )
        except PermissionDenied:
            raise RuntimeError(
                f"Permission denied reading insightgen-postgres-{key}. "
                f"Check service account has secretAccessor on this secret."
            )
        except GoogleAPIError as e:
            raise RuntimeError(
                f"GCP Secret Manager error reading insightgen-postgres-{key}: {e}"
            ) from e

    logger.info(f"PostgreSQL config loaded from GCP Secret Manager [project={project_id}]")
    return result


def _load_ca_cert_from_gcp() -> str:
    """
    Fetch the CA certificate from GCP Secret Manager and write it to a temp file.
    Returns the file path. Called once at cold start — path cached in _ca_cert_path.

    The CA cert is needed for sslmode=verify-ca — psycopg2 requires a file path
    for sslrootcert, not the cert content directly. /tmp is writable in Cloud Run.
    """
    from google.cloud import secretmanager
    from google.api_core.exceptions import GoogleAPIError

    project_id = os.environ.get('GCP_PROJECT_ID')
    if not project_id:
        raise ValueError("GCP_PROJECT_ID environment variable must be set.")

    client = secretmanager.SecretManagerServiceClient()
    secret_name = (
        f"projects/{project_id}/secrets/"
        f"insightgen-postgres-ca-cert/versions/latest"
    )

    try:
        response = client.access_secret_version(request={"name": secret_name})
        ca_cert_content = response.payload.data.decode("UTF-8")
    except GoogleAPIError as e:
        raise RuntimeError(
            f"Failed to fetch CA certificate from Secret Manager: {e}"
        ) from e

    # Write to a temp file — psycopg2 sslrootcert requires a file path
    # NamedTemporaryFile with delete=False persists for the container lifetime
    tmp = tempfile.NamedTemporaryFile(
        mode='w',
        suffix='.crt',
        prefix='pg_ca_',
        dir='/tmp',
        delete=False
    )
    tmp.write(ca_cert_content)
    tmp.flush()
    tmp.close()

    logger.info(f"CA certificate written to {tmp.name}")
    return tmp.name


# ─── Singleton loaders ────────────────────────────────────────────────────────

def get_postgres_config() -> dict:
    """
    Return PostgreSQL connection credentials.
    Singleton: loaded once at cold start, cached for container lifetime.
    """
    global _postgres_config

    if _postgres_config is not None:
        return _postgres_config

    logger.info(f"Cold start: loading PostgreSQL config from {_CLOUD_PROVIDER}")

    if _CLOUD_PROVIDER == 'AWS':
        raw = _load_from_aws()
    elif _CLOUD_PROVIDER == 'GCP':
        raw = _load_from_gcp()
    else:
        raise ValueError(f"Unsupported CLOUD_PROVIDER: '{_CLOUD_PROVIDER}'. Must be AWS or GCP.")

    missing = _REQUIRED_KEYS - set(raw.keys())
    if missing:
        raise KeyError(f"Missing required PostgreSQL config keys: {missing}")

    _postgres_config = raw
    return _postgres_config


def _get_ca_cert_path() -> str:
    """
    Return path to the CA certificate file.
    For GCP: fetched from Secret Manager and written to /tmp once at cold start.
    For AWS: reads from the path set in POSTGRES_CA_CERT_PATH env var,
             or falls back to no CA verification if not set.
    Singleton: fetched and written once, path cached.
    """
    global _ca_cert_path

    if _ca_cert_path is not None:
        return _ca_cert_path

    if _CLOUD_PROVIDER == 'GCP':
        _ca_cert_path = _load_ca_cert_from_gcp()
    elif _CLOUD_PROVIDER == 'AWS':
        # On AWS, ca.crt can be bundled in the Lambda layer and path set via env var
        # If not configured, verify-ca is skipped and sslmode=require is used
        _ca_cert_path = os.environ.get('POSTGRES_CA_CERT_PATH', '')

    return _ca_cert_path


# ─── Public API ───────────────────────────────────────────────────────────────

def get_postgres_dsn() -> str:
    """
    Return a libpq connection string.

    sslmode behaviour:
      - If CA cert is available: sslmode=verify-ca
        Verifies the server certificate is signed by our CA.
        Protects against eavesdropping and man-in-the-middle attacks.
      - If CA cert is not available: sslmode=require
        Encrypts the connection but does not verify the server certificate.
        Falls back gracefully rather than failing if CA cert is missing.
    """
    cfg = get_postgres_config()
    ca_path = _get_ca_cert_path()

    if ca_path:
        # verify-ca: encrypted + server cert verified against our CA
        dsn = (
            f"host={cfg['host']} "
            f"port={cfg['port']} "
            f"dbname={cfg['dbname']} "
            f"user={cfg['username']} "
            f"password={cfg['password']} "
            f"sslmode=verify-ca "
            f"sslrootcert={ca_path}"
        )
        logger.info("PostgreSQL DSN: sslmode=verify-ca")
    else:
        # require: encrypted but server cert not verified
        dsn = (
            f"host={cfg['host']} "
            f"port={cfg['port']} "
            f"dbname={cfg['dbname']} "
            f"user={cfg['username']} "
            f"password={cfg['password']} "
            f"sslmode=require"
        )
        logger.info("PostgreSQL DSN: sslmode=require (no CA cert configured)")

    return dsn


def get_postgres_url() -> str:
    """
    Return a SQLAlchemy-compatible PostgreSQL URL.
    """
    cfg = get_postgres_config()
    ca_path = _get_ca_cert_path()

    url = (
        f"postgresql+psycopg2://{cfg['username']}:{cfg['password']}"
        f"@{cfg['host']}:{cfg['port']}/{cfg['dbname']}"
        f"?sslmode=verify-ca&sslrootcert={ca_path}" if ca_path
        else f"postgresql+psycopg2://{cfg['username']}:{cfg['password']}"
             f"@{cfg['host']}:{cfg['port']}/{cfg['dbname']}?sslmode=require"
    )
    return url
```

---

## Step 13 — Update Cloud Run Environment Variables

Add `POSTGRES_SSL_MODE` as a reference variable (optional — the code determines mode automatically based on whether the CA cert is available, but documenting it is good practice):

```bash
gcloud run services update insightgen-api-configurator \
  --project $PROJECT_ID \
  --region $REGION \
  --update-env-vars \
    CLOUD_PROVIDER=GCP,\
    GCP_PROJECT_ID=${PROJECT_ID},\
    APP_ENV=int,\
    APP_VERSION=mvp3
```

Push `secrets_client.py` to GitHub and let Cloud Build deploy:

```bash
git add secrets_client.py
git commit -m "upgrade to sslmode=verify-ca using CA cert from Secret Manager"
git push origin main
```

---

## Step 14 — Test verify-ca End-to-End

Test from Cloud Shell via the load balancer:

```bash
curl -s "http://10.131.36.204/int/secrets" | python3 -m json.tool
```

Expected:
```json
{
  "status": "connected",
  "cloud": "GCP",
  "db_version": "PostgreSQL 15.4...",
  "database": "insightgen",
  "user": "insightgen_app"
}
```

Check Cloud Run logs to confirm `verify-ca` is active — not falling back to `require`:

```bash
gcloud logging read \
  "resource.type=cloud_run_revision \
   AND resource.labels.service_name=insightgen-api-configurator" \
  --project $PROJECT_ID \
  --limit 20 \
  --format "table(timestamp, textPayload)" \
  | grep -i "ssl"
```

You should see: `PostgreSQL DSN: sslmode=verify-ca`

---

## Step 15 — Confirm TLS Connection Is Active

Trigger a curl request then immediately check active SSL connections inside PostgreSQL:

```bash
docker exec $CONTAINER_NAME \
  psql -U postgres \
  -c "SELECT pid, ssl, version, cipher, client_addr FROM pg_stat_ssl WHERE ssl = true;"
```

You should see a row with:
- `ssl = t`
- `version = TLSv1.3`
- `client_addr` showing the Cloud Run function's internal IP

---

## Troubleshooting

**verify-ca fails with SSL certificate verify failed**

The CA certificate in Secret Manager does not match the CA that signed the server certificate. Verify they match:

```bash
# On the GCP VM — check what CA signed the server cert
openssl x509 -in /opt/postgres-ssl/server.crt -noout -issuer

# Check what CA is stored in Secret Manager
gcloud secrets versions access latest \
  --secret insightgen-postgres-ca-cert \
  --project $PROJECT_ID \
  | openssl x509 -noout -subject
```

The issuer from the server cert must match the subject of the CA cert.

**CA cert temp file not being created**

Check Cloud Run logs for the `CA certificate written to /tmp/...` message. If it is missing, the Secret Manager fetch failed. Confirm the secret exists and the service account has access:

```bash
gcloud secrets get-iam-policy insightgen-postgres-ca-cert \
  --project $PROJECT_ID
```

**Falls back to sslmode=require instead of verify-ca**

The `_get_ca_cert_path()` function returned an empty string, meaning the GCP secret fetch failed silently. Check the Cloud Run logs for any error mentioning `insightgen-postgres-ca-cert`.

**pg_hba.conf has both host and hostssl rules**

PostgreSQL evaluates rules top to bottom. If a plain `host` rule appears before `hostssl` for the same subnet, unencrypted connections are still allowed. Check the file and ensure `hostssl` is the first matching rule for the Cloud Run IP range:

```bash
docker exec $CONTAINER_NAME cat /var/lib/postgresql/data/pg_hba.conf
```

---

## Certificate Rotation (Future Reference)

Since certificates are self-signed with 10-year validity, rotation is infrequent. When it is needed:

```
1. Generate new CA and server certificate on the GCP VM
   (same commands as Step 3)

2. Copy new certs into the container and restart PostgreSQL
   (same as Steps 5 and 7)

3. Update the CA cert in Secret Manager with the new version
   cat /opt/postgres-ssl/ca.crt | \
     gcloud secrets versions add insightgen-postgres-ca-cert --data-file=-

4. Redeploy Cloud Run to force a cold start
   gcloud run services update insightgen-api-configurator --region $REGION
   Cloud Run fetches the new CA cert from Secret Manager at cold start
   No code changes needed

5. Verify connection with new certificates using Step 15
```

---

## Completion Checklist

```
[ ] Step 1  — Connected to GCP VM
[ ] Step 2  — Confirmed OpenSSL version
[ ] Step 3  — Generated CA and server certificates, verified RSA format
[ ] Step 4  — Identified PostgreSQL container name, set variables
[ ] Step 5  — Copied certs into container, correct ownership and permissions
[ ] Step 6  — ssl=on in postgresql.conf, hostssl rule in pg_hba.conf
[ ] Step 7  — Container restarted, no FATAL errors in logs
[ ] Step 8  — ssl=on and TLS version confirmed via pg_stat_ssl
[ ] Step 9  — docker-compose.yml updated with cert volume mounts
[ ] Step 10 — Tested sslmode=require — status: connected confirmed
[ ] Step 11 — ca.crt stored in GCP Secret Manager as insightgen-postgres-ca-cert
[ ] Step 12 — secrets_client.py updated with verify-ca logic and CA cert fetch
[ ] Step 13 — Cloud Build deployed updated secrets_client.py
[ ] Step 14 — Tested via curl — status: connected, logs show sslmode=verify-ca
[ ] Step 15 — pg_stat_ssl confirms ssl=true with Cloud Run IP
```

---

## Final State After This Task

| Item | Before | After |
|---|---|---|
| PostgreSQL SSL | Off — unencrypted | On — TLSv1.2 minimum |
| Certificates | None | Self-signed CA + RSA server cert |
| Cloud Run connection | sslmode removed (workaround) | sslmode=verify-ca |
| CA cert storage | N/A | GCP Secret Manager — `insightgen-postgres-ca-cert` |
| CA cert delivery | N/A | Fetched at cold start, written to /tmp, singleton cached |
| What verify-ca adds over require | — | Server cert verified against CA — MITM not possible |
| Cert persistence on VM | N/A | /opt/postgres-ssl/ mounted via Docker Compose |
| At-rest encryption | GCP default Google-managed | GCP default unchanged |
| Future cert rotation | N/A | Regenerate certs → update Secret Manager version → redeploy Cloud Run |
