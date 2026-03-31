# Task Summary — Cross-Cloud Secrets Management Implementation
**Project:** InsightGen  
**Engineer:** fitindevops  
**Completed:** March 2026

---

## Task Overview

Implemented a cloud-native secrets management solution to securely retrieve PostgreSQL credentials in both AWS (Lambda) and GCP (Cloud Run) using a single shared codebase. The solution uses the singleton pattern so credentials are loaded once per container lifetime with zero repeated network calls.

---

## Task 1 — Resolved Existing AWS Lambda SSM Decryption Issue

**What was done:**

The Lambda function was failing to decrypt SSM Parameter Store values at runtime. The root cause was that the Lambda execution role (`HCL-User-Role-InsightGen-Lambda`) was not included in the KMS key policy. The KMS key policy was updated to add a dedicated statement allowing the Lambda role to call `kms:Decrypt` scoped specifically to SSM via service context and restricted to the `/insightgen/postgres/*` parameter path.

**KMS policy statement added:**
```json
{
  "Sid": "AllowLambdaRoleDecryptViaSSMForInsightGenPostgres",
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::xxxx:role/HCL-User-Role-InsightGen-Lambda"
  },
  "Action": "kms:Decrypt",
  "Resource": "*",
  "Condition": {
    "StringEquals": {
      "kms:ViaService": "ssm.us-east-1.amazonaws.com"
    },
    "StringLike": {
      "kms:EncryptionContext:PARAMETER_ARN": 
        "arn:aws:ssm:us-east-1:xxxx:parameter/insightgen/postgres/*"
    }
  }
}
```

**Outcome:** Lambda successfully decrypts and retrieves all five PostgreSQL SSM parameters at cold start.

---

## Task 2 — Designed and Decided Secrets Management Approach

**What was done:**

Evaluated two architectural options for handling secrets across AWS and GCP:

**Option A — Cloud-native:** AWS SSM Parameter Store for Lambda, GCP Secret Manager for Cloud Run. Two different SDKs but no extra infrastructure to manage.

**Option B — OpenBao (unified):** Single open-source secrets store deployed as a container in both clouds. One SDK (`hvac`), only the endpoint URL changes between clouds. Adds operational burden — unseal on restart, token rotation, bootstrap problem.

**Decision:** Cloud-native approach selected for the POC. With only two clouds, the code difference is minimal (approximately 10 lines in one file). OpenBao adds a critical dependency that must be operated and secured, which is not justified at POC stage.

**Architect input:** Singleton pattern recommended for static credentials — load once at cold start, cache for the container lifetime, no TTL or refresh logic needed since credentials do not rotate frequently.

**Outcome:** Architecture decision documented. Cloud-native approach with singleton pattern selected.

---

## Task 3 — Created GCP Cloud KMS Keyring and Key (GCP Console)

**What was done:**

Created a customer-managed encryption key (CMEK) in GCP KMS via the GCP Console to encrypt Secret Manager values, equivalent to the custom KMS key already in use on AWS.

- Created a global keyring named `insightgen-keyring`
- Created an encryption key named `insightgen-secrets-key` inside the keyring
- Granted the Secret Manager service agent (`service-{PROJECT_NUMBER}@gcp-sa-secretmanager.iam.gserviceaccount.com`) the `roles/cloudkms.cryptoKeyEncrypterDecrypter` role on the key so Secret Manager can encrypt and decrypt secret versions on behalf of the project

**Note:** A regional keyring was initially created in `us-central1` during early setup. The final working keyring is global.

**Outcome:** CMEK key ready for use when creating secrets in Secret Manager.

---

## Task 4 — Created PostgreSQL Secrets in GCP Secret Manager

**What was done:**

Created five global secrets in GCP Secret Manager storing the PostgreSQL connection credentials for the GCP environment:

```
insightgen-postgres-host
insightgen-postgres-port
insightgen-postgres-dbname
insightgen-postgres-username
insightgen-postgres-password
```

Each secret was created using the custom CMEK key from Task 3 so values are encrypted with the project-controlled key rather than Google's default key. Secret values were added as versions using `echo -n` to prevent trailing newlines being stored as part of the credential.

**Issue encountered — Regional vs Global Secret Manager confusion:**

Secrets were initially created in the Regional Secret Manager tab in the GCP Console (`us-central1` location). This is a separate GCP product used for data residency compliance — it uses a different API endpoint (`us-central1-secretmanager.googleapis.com`) and a different resource path format (`projects/{project}/locations/{region}/secrets/{name}`).

This caused two problems:
- `gcloud secrets list` does not list regional secrets — only global ones appear
- The service account had no permissions on regional secrets even though it had `secretAccessor` at project level for global secrets
- The code was updated to use the regional endpoint which caused a `501 received http2 header with status 404` error because the regional Secret Manager API was not properly enabled

**Resolution:** All five secrets were recreated in the Global Secret Manager (default tab in GCP Console, no location qualifier). The global secrets are accessed via the standard `secretmanager.googleapis.com` endpoint with the path format `projects/{project}/secrets/{name}/versions/latest`. The original regional secrets were deleted after confirming the global ones worked.

**Outcome:** Five global secrets created, CMEK-encrypted, with correct values.

---

## Task 5 — Configured Service Account Permissions on GCP Secrets

**What was done:**

Granted the Cloud Run service account (`app-svc-ifrs-fs@{PROJECT_ID}.iam.gserviceaccount.com`) access to read the five PostgreSQL secrets. When creating each secret with the custom KMS key selected in the GCP Console, the console prompted to grant the service account `Encrypt` and `Decrypt` permissions — this was accepted and applied.

Confirmed the service account has `roles/secretmanager.secretAccessor` at the project level by checking the IAM policy bindings.

**Note on access model:**
- The service account has project-level `secretAccessor` — it can read all global secrets in the project
- The engineer's user group (`GCP-ifrs-user`) has `Secret Manager Admin` — can create and manage all secrets
- This mirrors the AWS model: Lambda role can decrypt via SSM, approved users can manage via the KMS key policy

**Outcome:** Service account confirmed able to read all five insightgen PostgreSQL secrets.

---

## Task 6 — Wrote the Python Secrets Client and Application Code

**What was done:**

Created two Python files that form the complete implementation:

**`secrets_client.py` — singleton pattern, all cloud-specific code:**

- Module-level variable `_postgres_config = None` holds credentials after first load
- `get_postgres_config()` checks if the singleton is populated — returns immediately on warm calls
- `_load_from_aws()` fetches all five parameters in a single `GetParametersByPath` batch call from SSM
- `_load_from_gcp()` fetches each of the five secrets individually from GCP Secret Manager using the global client
- `get_postgres_dsn()` builds a libpq connection string with `sslmode=require`
- `CLOUD_PROVIDER` environment variable read once at module import time — controls which backend runs

**`main.py` — application code, identical on both clouds:**

- `lambda_handler(event, context)` — AWS Lambda entry point
- `cloud_run_handler(request)` — GCP Cloud Run HTTP entry point
- Module-level `_db_conn` variable — reuses the PostgreSQL connection across warm invocations
- Both handlers call the same `_get_connection()` function which calls `get_postgres_dsn()` from the singleton

**Issue encountered — `inet_server_addr()` causing Lambda timeout:**

The initial version of `main.py` included `inet_server_addr()` in the SQL query to show which host PostgreSQL was connected on. This function requires the `pg_monitor` role or superuser privileges. The `insightgen_app` database user does not have either, causing the query to hang until the 60-second Lambda timeout was hit.

**Resolution:** Removed `inet_server_addr()` from the SQL query. Both handlers now use `SELECT version(), current_database(), current_user` which requires no special privileges.

**Outcome:** Both files written, tested, and working on both clouds.

---

## Task 7 — Created GitHub Repo and Cloud Build Pipeline for GCP Deployment

**What was done:**

Created a new repository in the GitHub Enterprise organisation to hold the Cloud Run function code. Repository contains:

```
insightgen-cloud-run/
├── main.py
├── secrets_client.py
└── requirements-gcp.txt
```

Created a Cloud Build trigger in GCP Console connected to the GitHub organisation repository. Created a `cloudbuild.yaml` pipeline that builds and deploys the Cloud Run function on every push to the main branch. This mirrors the existing CodeBuild pipeline pattern used for Lambda on AWS.

Configured the Cloud Run function deployment with the following environment variables set via the pipeline:

```
CLOUD_PROVIDER=GCP
GCP_PROJECT_ID={PROJECT_ID}
APP_ENV=int
APP_VERSION=mvp3
```

**Outcome:** Code push to GitHub triggers automatic Cloud Build pipeline which deploys updated Cloud Run function.

---

## Task 8 — Deployed and Configured GCP Cloud Run Function

**What was done:**

Deployed the Cloud Run function (`insightgen-api-configurator`) with the service account `app-svc-ifrs-fs` attached. Configured the function to use `cloud_run_handler` as the entry point.

**Issue encountered — Secret Manager timeout (Private Google Access):**

After deployment, calling the function through the load balancer returned:

```
type: internal
detail: gcp secret manager error reading insightgen-postgres-dbname
timeout 60s exceeded — 503 failed to connect to all addresses
last error UNKNOWN ipv4: tcp handshake shutdown
```

Root cause: The Cloud Run function was running inside a VPC. When Cloud Run has VPC egress enabled, all outbound traffic routes through the VPC. The VPC subnet did not have Private Google Access enabled, so traffic to `secretmanager.googleapis.com` (a Google API) could not exit the VPC to reach Google's network. The TCP handshake never completed.

**Resolution:** Updated the Cloud Run VPC egress setting to `private-ranges-only`. This keeps traffic to internal RFC1918 addresses (like the PostgreSQL host) inside the VPC, while allowing Google API traffic (`secretmanager.googleapis.com`, `kms.googleapis.com`) to take the direct path outside the VPC without being blocked. The permanent fix is to enable Private Google Access on the VPC subnet — the `private-ranges-only` setting is the operational workaround until the networking team makes that change.

```bash
gcloud run services update insightgen-api-configurator \
  --vpc-egress private-ranges-only
```

**Outcome:** Cloud Run function able to reach Secret Manager and retrieve credentials.

---

## Task 9 — Configured Load Balancer and Tested End-to-End

**What was done:**

The Cloud Run function was registered as a Serverless NEG (Network Endpoint Group) backend on the existing internal load balancer. A route was added so traffic to `http://10.131.36.204/int/secrets` is forwarded to the Cloud Run function.

**Issue encountered — Direct function URL returning page not found:**

Testing the Cloud Run function URL directly returned a page not found error. This was because testing should be done through the load balancer URL, not the function URL directly. The load balancer is the entry point — the function URL bypasses the routing configuration.

**Networking option issue — VPC connector failing:**

The Cloud Run function has two networking options:

1. **Send traffic directly to VPC** — traffic goes directly from the Cloud Run instance into the VPC network. Uses existing VPC routing and firewall rules. This worked correctly.

2. **Use Serverless VPC Access Connector** — traffic routes through a separate set of small connector VMs before entering the VPC. The connector VMs had no firewall rules allowing them to reach the PostgreSQL host on port 5432, causing 503 timeout errors.

**Resolution:** Direct VPC option selected and kept. The connector approach is an older GCP pattern — direct VPC egress is the current recommended approach and is simpler. No connector configuration is needed.

**Testing method:** curl from Cloud Shell using the internal load balancer IP (since the IP is internal to the VPC, testing must be done from within the VPC or Cloud Shell):

```bash
curl -v "http://10.131.36.204/int/secrets"
```

**Expected response:**
```json
{
  "status": "connected",
  "cloud": "GCP",
  "db_version": "PostgreSQL 15.x...",
  "database": "insightgen",
  "user": "insightgen_app",
  "path_received": "/int/secrets"
}
```

**Outcome:** End-to-end flow working. Load balancer routes `/int/secrets` → Cloud Run function → GCP Secret Manager → PostgreSQL → returns connected status.

---

## Final Architecture

```
AWS Flow
────────
API call
  → AWS ALB
  → Lambda (CLOUD_PROVIDER=AWS)
  → secrets_client._load_from_aws()   [cold start only — singleton]
  → SSM Parameter Store /insightgen/postgres/*
  → KMS decrypt (Lambda role in key policy)
  → PostgreSQL container (sslmode=require)
  → Response

GCP Flow
────────
API call
  → Internal Load Balancer (10.131.36.204)
  → Serverless NEG
  → Cloud Run (CLOUD_PROVIDER=GCP)
  → secrets_client._load_from_gcp()   [cold start only — singleton]
  → Secret Manager insightgen-postgres-* (global)
  → Cloud KMS decrypt (global keyring)
  → PostgreSQL container (sslmode=require)
  → Response
```

---

## Issues Encountered and Resolutions — Quick Reference

| # | Issue | Root Cause | Resolution |
|---|---|---|---|
| 1 | Lambda cannot decrypt SSM values | Lambda role missing from KMS key policy | Added `AllowLambdaRoleDecryptViaSSMForInsightGenPostgres` statement to KMS key policy |
| 2 | `gcloud secrets list` not showing secrets | Secrets created in Regional Secret Manager, not Global | Recreated all five secrets in Global Secret Manager |
| 3 | Service account had no permission on regional secrets | Regional and Global Secret Manager are separate products with separate IAM | Moved to global secrets where SA already had project-level `secretAccessor` |
| 4 | `501 received http2 header with status 404` | Code was calling regional API endpoint for secrets that were still global | Reverted to global `SecretManagerServiceClient()` with no `ClientOptions` |
| 5 | Secret Manager timeout — TCP handshake shutdown | VPC egress routing blocked traffic to Google APIs — Private Google Access not enabled on subnet | Set Cloud Run VPC egress to `private-ranges-only` as workaround |
| 6 | Direct Cloud Run URL returning page not found | Testing through wrong URL — internal LB IP must be used for VPC-internal functions | Tested via `curl http://10.131.36.204/int/secrets` from Cloud Shell |
| 7 | Lambda timeout on updated `main.py` | `inet_server_addr()` SQL function requires `pg_monitor` role — `insightgen_app` user does not have it | Removed `inet_server_addr()` from SQL query — used `version(), current_database(), current_user` only |
| 8 | VPC connector networking failing with 503 | Connector VM cluster had no firewall rules to reach PostgreSQL | Switched to direct VPC networking — connector approach abandoned |

---

## What Was Confirmed Working

| Component | Status |
|---|---|
| Lambda → SSM Parameter Store → PostgreSQL (AWS) | Working |
| Cloud Run → Secret Manager (global) → PostgreSQL (GCP) | Working |
| Load balancer route `/int/secrets` → Cloud Run | Working |
| Singleton pattern — one credential fetch per cold start | Confirmed |
| CLOUD_PROVIDER env var switching between backends | Confirmed |
| Cloud Build pipeline — push to GitHub triggers deployment | Working |
| Direct VPC networking on Cloud Run | Working |
| KMS CMEK encryption on both clouds | Working |
