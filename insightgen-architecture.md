# InsightGen — Architecture and Request Flow
**Project:** InsightGen  
**Environment:** AWS (primary), GCP (secondary)  
**Document type:** Architecture reference  
**Author:** fitindevops  
**Last updated:** March 2026

---

## 1. Architecture Overview

InsightGen is an internal AI agent platform. Users interact through an Angular frontend to view and analyse alerts using AI agents powered by AWS Bedrock. All infrastructure runs inside a private AWS VPC. No component is directly accessible from the public internet — all public-facing entry points terminate at the load balancer which sits at the boundary.

The platform is currently deployed at POC stage across two environments: `dev` and `int`. Three MVP versions (mvp1, mvp2, mvp3) of the Angular application run concurrently on the same EC2 instance.

---

## 2. Component Inventory

### AWS Infrastructure

| Component | Type | Location |
|---|---|---|
| Internal Application Load Balancer | AWS ALB | VPC — public-facing with private backend |
| Route53 DNS Zone | AWS Route53 | Public hosted zone — resolves to ALB |
| EC2 Instance (t3.large) | AWS EC2 | Private subnet |
| Lambda Functions | AWS Lambda | Private subnets (VPC-attached) |
| SSM Parameter Store | AWS SSM | Regional service — accessed via VPC endpoint |
| KMS Custom Key | AWS KMS | Regional service — accessed via VPC endpoint |
| AWS Bedrock | AWS Bedrock | Regional service — accessed via VPC endpoint |
| S3 (Lambda packages, artifacts) | AWS S3 | Accessed via S3 VPC gateway endpoint |

### EC2 Containers (all on single t3.large instance)

| Container | Purpose | Internal Port |
|---|---|---|
| Nginx | Reverse proxy — routes traffic to correct container | 80 |
| Angular mvp1 | Frontend — version 1 | internal |
| Angular mvp2 | Frontend — version 2 | internal |
| Angular mvp3 | Frontend — version 3 | internal |
| Kong API Gateway | API gateway — authenticates and routes API calls | 8000 (proxy) / 8001 (admin — localhost only) |
| Kong PostgreSQL | Kong's own database | 5432 internal |
| PostgreSQL (main) | Application database | 5432 internal |
| pgAdmin | Database management UI | 5050 |
| Keycloak | User authentication and authorization | 8080 |
| Keycloak PostgreSQL | Keycloak's own database | 5432 internal |
| MCP Server | AI tool server — deployed, not yet integrated | — |

### Network Boundaries

| Traffic type | Protocol | Notes |
|---|---|---|
| User browser → ALB | HTTPS (TLS 1.2+) | Only HTTPS boundary in the system |
| ALB → EC2 Nginx | HTTP | Internal VPC — private subnet |
| Nginx → Angular containers | HTTP | Localhost on EC2 |
| Angular → ALB (Kong route) | HTTP | Internal VPC |
| ALB → EC2 Nginx → Kong | HTTP | Internal VPC |
| Kong → Lambda | HTTPS | AWS Lambda plugin — Kong calls Lambda invoke API |
| Lambda → PostgreSQL | TLS (sslmode=require) | Encrypted, certificate not yet verified |
| Lambda → SSM / KMS / Bedrock | HTTPS | Via VPC endpoints — traffic never leaves AWS network |
| EC2 → SSM Session Manager | HTTPS | Admin access — no SSH or bastion |

---

## 3. Network Topology

```
                        INTERNET
                            │
                            │ HTTPS only
                            ▼
                    ┌───────────────┐
                    │   Route53     │
                    │ Public DNS    │
                    │ (public zone) │
                    └───────┬───────┘
                            │ resolves to ALB
                            ▼
              ┌─────────────────────────────┐
              │  Internal Application       │
              │  Load Balancer              │
              │                             │
              │  Listener: HTTPS 443        │
              │  TLS termination here       │
              │  ─────────────────          │
              │  Routes:                    │
              │  /           → Angular      │
              │  /mvp1/      → Angular mvp1 │
              │  /mvp2/      → Angular mvp2 │
              │  /mvp3/      → Angular mvp3 │
              │  /kong-api/  → Kong proxy   │
              └──────────────┬──────────────┘
                             │ HTTP (port 80)
                             │ IP-based target group
                             ▼
┌────────────────────────────────────────────────────────────────────┐
│  AWS VPC — Private Subnet                                          │
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │  EC2 Instance (t3.large) — Private IP                       │  │
│  │                                                             │  │
│  │  ┌──────────────────────────────────────────────────────┐  │  │
│  │  │  Nginx (port 80)                                     │  │  │
│  │  │  Reverse proxy — routes by path prefix               │  │  │
│  │  │                                                      │  │  │
│  │  │  /           → Angular mvp3 container                │  │  │
│  │  │  /mvp1/      → Angular mvp1 container                │  │  │
│  │  │  /mvp2/      → Angular mvp2 container                │  │  │
│  │  │  /kong-api/  → Kong proxy (port 8000)                │  │  │
│  │  └──────────────────────────────────────────────────────┘  │  │
│  │                                                             │  │
│  │  ┌────────────┐  ┌─────────────────────────────────────┐  │  │
│  │  │ Angular    │  │  Kong API Gateway (port 8000)        │  │  │
│  │  │ mvp1/2/3   │  │  Admin API: 127.0.0.1:8001 only     │  │  │
│  │  │ containers │  │  Plugins: key-auth, JWT (configured) │  │  │
│  │  └────────────┘  └──────────────┬──────────────────────┘  │  │
│  │                                  │                          │  │
│  │  ┌───────────┐  ┌────────────┐  │  ┌────────────────────┐  │  │
│  │  │ Keycloak  │  │  Kong      │  │  │  MCP Server        │  │  │
│  │  │ :8080     │  │ PostgreSQL │  │  │  (deployed,        │  │  │
│  │  └─────┬─────┘  └────────────┘  │  │  not integrated)   │  │  │
│  │        │                        │  └────────────────────┘  │  │
│  │  ┌─────┴──────┐                 │                          │  │
│  │  │ Keycloak   │                 │ AWS Lambda plugin        │  │
│  │  │ PostgreSQL │                 │ invokes Lambda ARN       │  │
│  │  └────────────┘                 │                          │  │
│  │                                 │                          │  │
│  │  ┌─────────────────────────┐    │                          │  │
│  │  │  PostgreSQL (main app)  │◄───┼──────────────────────┐  │  │
│  │  │  TLS enabled            │    │                      │  │  │
│  │  │  Self-signed CA cert    │    │                      │  │  │
│  │  │  sslmode=require        │    │                      │  │  │
│  │  └─────────────────────────┘    │                      │  │  │
│  │                                 │                      │  │  │
│  │  ┌──────────┐                   │                      │  │  │
│  │  │ pgAdmin  │                   │                      │  │  │
│  │  │ :5050    │                   │                      │  │  │
│  │  └──────────┘                   │                      │  │  │
│  └─────────────────────────────────┘                      │  │  │
│                                                            │  │  │
│  ┌─────────────────────────────────────────────────────┐  │  │  │
│  │  AWS Lambda Functions (private subnets, same VPC)   │  │  │  │
│  │                                                     │  │  │  │
│  │  dev-api-configurator   int-api-configurator        │  │  │  │
│  │  dev-api-bedrock-agent  int-api-bedrock-agent       │  │  │  │
│  │  dev-api-[others]       int-api-[others]            │  │  │  │
│  │                                                     │  │  │  │
│  │  On cold start:                                     │  │  │  │
│  │  SSM Parameter Store ◄──────────────────────────┐  │  │  │  │
│  │  KMS decrypt ◄──────────────────────────────┐   │  │  │  │  │
│  │                                             │   │  │  │  │  │
│  │  On every request:                          │   │  │  │  │  │
│  │  PostgreSQL connection (cached, TLS) ───────┼───┼──┘  │  │  │
│  │  Bedrock API call (AI processing) ──────────┼───┘     │  │  │
│  └─────────────────────────────────────────────┼─────────┘  │  │
│                                                 │            │  │
│  ┌──────────────────────────────────────────────┼────────┐  │  │
│  │  VPC Endpoints (traffic stays in AWS network)│        │  │  │
│  │                                              │        │  │  │
│  │  ssm.us-east-1.amazonaws.com ◄───────────────┘        │  │  │
│  │  kms.us-east-1.amazonaws.com ◄────────────────────────┘  │  │
│  │  bedrock-runtime.us-east-1.amazonaws.com                  │  │
│  │  s3 (gateway endpoint)                                    │  │
│  └───────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────┘

  EC2 Admin Access:
  DevOps engineer → AWS SSM Session Manager (HTTPS) → EC2 instance
  No SSH. No bastion. No public port 22 open.
```

---

## 4. Main Request Flow — User API Call

This is the complete path of a single API request from user browser to database and back.

### Step-by-step

```
Step 1 — DNS Resolution
  User's browser resolves the application hostname
  DNS query → Route53 public hosted zone
  Route53 returns the ALB DNS name / IP
  Connection established to ALB on port 443 (HTTPS)

Step 2 — TLS Termination at ALB
  ALB holds the TLS certificate
  Browser ↔ ALB: HTTPS (TLS 1.2+) — encrypted
  ALB decrypts the request
  All traffic beyond this point is HTTP inside the VPC private subnet

Step 3 — ALB Routes to Nginx
  ALB evaluates the request path
  Matches against configured listener rules:
    /kong-api/* → IP-based target group → EC2 private IP, port 80
    /mvp3/*     → IP-based target group → EC2 private IP, port 80
    /mvp1/*     → IP-based target group → EC2 private IP, port 80
  Forwards HTTP request to EC2 instance on port 80

Step 4 — Nginx Reverse Proxy on EC2
  Nginx receives the request on port 80
  Evaluates the path prefix:
    /kong-api/* → proxy_pass to Kong on port 8000
                  strips /kong-api prefix before forwarding
    /mvp3/*     → proxy_pass to Angular mvp3 container
    /mvp1/*     → proxy_pass to Angular mvp1 container
    /mvp2/*     → proxy_pass to Angular mvp2 container

Step 5 — Kong API Gateway (port 8000)
  Kong receives the API request (e.g. /dev/api-configurator/configs)
  Evaluates configured plugins in order:

  [Plugin 1: key-auth]
  Kong checks the request for a valid API key
  API key passed in header: apikey: <key>
  Kong looks up the key in its PostgreSQL database
  If key is invalid or missing → 401 Unauthorized — request rejected here
  If key is valid → request passes to next plugin

  [Plugin 2: JWT (configured, not primary)]
  JWT plugin is configured and can validate Bearer tokens
  Validation uses Keycloak's RS256 public key stored in Kong consumer
  No call to Keycloak per request — signature verified locally
  Currently key-auth is the primary active authentication method

  [Route matching]
  Kong matches the request path to a configured service
  Each service has the AWS Lambda plugin attached

Step 6 — Kong AWS Lambda Plugin
  Kong invokes the target Lambda function directly
  Uses AWS Lambda plugin — calls Lambda Invoke API
  Lambda function name resolved from the Kong service configuration
  Kong uses the IAM role / credentials configured on the EC2 instance
  Request payload (headers, body, path, query params) forwarded to Lambda

Step 7 — Lambda Cold Start (first invocation only)
  If Lambda container is new (cold start):
    Lambda calls SSM GetParametersByPath on /insightgen/postgres/*
    SSM call goes through SSM VPC endpoint — never leaves AWS network
    SSM calls KMS to decrypt SecureString values
    KMS call goes through KMS VPC endpoint — never leaves AWS network
    KMS validates Lambda execution role is in key policy
    Decrypted credentials stored in singleton (_postgres_config)
    Lambda opens PostgreSQL connection with TLS (sslmode=require)
    Connection stored in module-level variable (_db_conn)
  
  If Lambda container is warm (subsequent invocations):
    _postgres_config already populated — no SSM or KMS call
    _db_conn already open — no reconnection
    Proceeds directly to business logic

Step 8 — Lambda Business Logic
  Lambda executes the application logic
  Queries PostgreSQL over TLS-encrypted connection
  If AI analysis is required:
    Lambda calls AWS Bedrock (bedrock-runtime endpoint)
    Bedrock call goes through Bedrock VPC endpoint
    Bedrock invokes Claude model (Sonnet or Haiku)
    Response returned to Lambda

Step 9 — Response Path (reverse)
  Lambda returns JSON response to Kong
  Kong returns response to Nginx
  Nginx returns response to ALB
  ALB encrypts response with TLS and returns to browser
```

### Simplified flow diagram

```
Browser
  │ HTTPS
  ▼
Route53 DNS
  │ resolves to ALB
  ▼
Internal ALB  ──── TLS terminates here ────
  │ HTTP (port 80)
  ▼
Nginx (EC2 port 80)
  │ /kong-api/* → Kong port 8000
  ▼
Kong API Gateway
  │ key-auth plugin validates API key
  │ AWS Lambda plugin invokes function
  ▼
Lambda Function (private subnet, same VPC)
  │ cold start: SSM → KMS → decrypt credentials → open DB connection
  │ warm: use cached credentials and connection
  ├──► PostgreSQL (TLS, sslmode=require)
  └──► Bedrock (via VPC endpoint, AI processing)
  │
  ▼
Response travels back the same path → Browser
```

---

## 5. Authentication Flow

### Current state — key-auth (active)

```
Developer/tester obtains an API key from Kong admin
API key stored in Kong's PostgreSQL database against a consumer

Request flow:
  Client sends request with header: apikey: <value>
  Kong key-auth plugin intercepts request
  Kong looks up key in Kong PostgreSQL
  Key valid → request continues to Lambda
  Key invalid or missing → 401 returned immediately
  Lambda is never called for invalid requests
```

### Current state — Keycloak (user auth, not yet in Kong)

Keycloak handles user login and session management for the Angular application. It runs as a container on the EC2 instance alongside the other services.

```
User opens Angular app in browser
  │
  ▼
Angular redirects to Keycloak login page
  │ http://ec2-internal/keycloak (via Nginx)
  ▼
Keycloak authenticates user (username/password)
  │ validates against Keycloak's own PostgreSQL
  ▼
Keycloak issues JWT (RS256 signed)
  │ access token + refresh token
  ▼
Angular stores token in browser memory
  │
  ▼
Angular includes token in API calls:
  Authorization: Bearer <jwt>

Current state: Kong does not validate this JWT on every request.
Key-auth is the active API authentication mechanism.
```

### Planned state — JWT fully integrated into Kong

When the JWT plugin is fully activated in Kong, the flow will extend to:

```
Kong receives request with Authorization: Bearer <jwt>
  │
  ▼
Kong JWT plugin validates the token locally
  No call to Keycloak — uses Keycloak's RS256 public key
  stored in the Kong consumer configuration
  Checks: signature valid, token not expired (exp claim),
          issuer matches Keycloak realm URL (iss claim)
  │
  Token invalid → 401 returned
  Token valid → check consumer and role claims
  │
  ▼
Role-based access control:
  JWT claims carry user roles from Keycloak
  Kong can allow or block based on role
  Example: only users with role=analyst can call /api-bedrock-agent
  │
  ▼
Request proceeds to Lambda
```

---

## 6. Database Architecture and Encryption

### PostgreSQL container setup

PostgreSQL runs as a Docker container on the EC2 instance. It is not a managed database — it is self-managed inside the container with data persisted on the EBS volume attached to the EC2 instance.

```
EC2 Instance
  │
  └── Docker container: insightgen-postgres
        │
        ├── Configuration: postgresql.conf
        │     ssl = on
        │     ssl_min_protocol_version = TLSv1.2
        │     ssl_cert_file = server.crt
        │     ssl_key_file = server.key
        │     ssl_ca_file = ca.crt
        │
        ├── Authentication: pg_hba.conf
        │     hostssl all all 0.0.0.0/0 scram-sha-256
        │     (requires SSL for all remote connections)
        │
        └── Certificates (mounted from EC2 host)
              ca.crt    — self-signed Certificate Authority
              server.crt — server certificate signed by the CA
              server.key — server private key (RSA format)
                           permissions: 600, owned by postgres user
```

### How TLS was set up

The original issue during setup was that OpenSSL 3.x on the EC2 host generates private keys in PKCS#8 format (`BEGIN PRIVATE KEY`) by default. PostgreSQL requires the traditional RSA format (`BEGIN RSA PRIVATE KEY`). This caused PostgreSQL to crash-loop on startup with the error `FATAL: could not load private key file: unsupported`.

Resolution: the key was converted to RSA format using `openssl rsa` and file permissions were corrected to `600` owned by the `postgres` user inside the container.

### Lambda to PostgreSQL connection

```
Lambda function (private subnet)
  │
  │  Connection string: host=<ec2-ip> port=5432 dbname=insightgen
  │                     user=insightgen_app password=<from-SSM>
  │                     sslmode=require
  │
  │  sslmode=require:
  │    - Connection is TLS encrypted
  │    - Server certificate is NOT verified (no CA check)
  │    - Protects against passive eavesdropping
  │    - Does not protect against MITM within the VPC
  │
  │  Planned upgrade — sslmode=verify-ca:
  │    - Connection is TLS encrypted
  │    - Server certificate verified against ca.crt
  │    - ca.crt bundled into Lambda layer or fetched from S3
  │    - Protects against both eavesdropping and MITM
  │
  ▼
PostgreSQL container (same VPC, EC2 private IP)
  Accepts connection, negotiates TLS 1.2+
  Authenticates user with scram-sha-256
  Connection established
```

### Data at rest encryption

The EC2 EBS volume storing PostgreSQL data files is encrypted using AWS KMS. This was enabled during a previous task as part of the encryption-at-rest initiative. All data written by PostgreSQL to disk is transparently encrypted by the EBS volume.

---

## 7. Secrets Management Flow

### How Lambda retrieves database credentials

Credentials are stored as SSM SecureString parameters encrypted with a custom KMS key. The Lambda function uses the singleton pattern — credentials are fetched once at cold start and cached for the container lifetime.

```
SSM Parameter Store
  Path: /insightgen/postgres/
  Parameters:
    /insightgen/postgres/host      → String (plaintext)
    /insightgen/postgres/port      → String (plaintext)
    /insightgen/postgres/dbname    → String (plaintext)
    /insightgen/postgres/username  → String (plaintext)
    /insightgen/postgres/password  → SecureString (KMS encrypted)
```

### Cold start credential fetch

```
Lambda container starts (cold start)
  │
  ▼
secrets_client.py — get_postgres_config() called
  │ _postgres_config is None — first call
  │
  ▼
boto3 SSM client: GetParametersByPath
  Path: /insightgen/postgres/
  WithDecryption: True
  │ request travels through SSM VPC interface endpoint
  │ never leaves AWS private network
  │
  ▼
SSM service calls KMS to decrypt SecureString values
  │ SSM calls KMS via service context: kms:ViaService = ssm.us-east-1.amazonaws.com
  │ KMS evaluates key policy
  │
  ▼
KMS key policy evaluation
  ┌─ Is caller the Lambda execution role?
  │   Condition: Principal = HCL-User-Role-InsightGen-Lambda
  │   Condition: kms:ViaService = ssm.us-east-1.amazonaws.com
  │   Condition: PARAMETER_ARN like /insightgen/postgres/*
  │   → ALLOW decrypt
  │
  ├─ Is caller an approved SSO user session?
  │   Condition: aws:userid matches approved user's session ID
  │   → ALLOW decrypt
  │
  └─ Is caller any other session of the shared SSO role?
      Condition: NOT the approved user session ID
      → DENY decrypt (explicit deny overrides any allow)
  │
  ▼
Decrypted values returned to Lambda via SSM response
  │
  ▼
secrets_client._postgres_config = {
  'host': '10.x.x.x',
  'port': '5432',
  'dbname': 'insightgen',
  'username': 'insightgen_app',
  'password': 'decrypted-password'
}
  │ Singleton populated — no further SSM or KMS calls for this container
  │
  ▼
psycopg2 connects to PostgreSQL using DSN with sslmode=require
_db_conn = psycopg2.connect(dsn)
  │ Connection cached for warm invocations
  ▼
Lambda handler executes business logic using cached connection
```

### Warm invocation (no credential fetch)

```
Lambda container already running (warm)
  │
  ▼
get_postgres_config() called
  │ _postgres_config is not None — return immediately
  │ Zero network calls to SSM or KMS
  │
  ▼
_get_connection() called
  │ _db_conn is not None and not closed — return existing connection
  │ Zero new database connections
  │
  ▼
Lambda handler executes — only business logic runs
```

---

## 8. AWS Service-to-Service Communication

All AWS API calls from Lambda go through VPC endpoints. Traffic between Lambda and AWS services never leaves Amazon's private network — there is no internet gateway or NAT gateway in the path.

```
Lambda (private subnet)
  │
  ├──► SSM VPC Interface Endpoint
  │       ssm.us-east-1.amazonaws.com (private DNS)
  │       Used for: GetParametersByPath (credential fetch at cold start)
  │
  ├──► KMS VPC Interface Endpoint
  │       kms.us-east-1.amazonaws.com (private DNS)
  │       Used for: Decrypt (called by SSM internally during GetParameter)
  │
  ├──► Bedrock VPC Interface Endpoint
  │       bedrock-runtime.us-east-1.amazonaws.com (private DNS)
  │       Used for: InvokeModel (AI analysis — Claude Sonnet/Haiku)
  │
  └──► S3 VPC Gateway Endpoint
          Used for: Lambda deployment packages, artifacts
          Gateway type — no data transfer charges, no interface endpoint cost
```

### Bedrock call flow

```
Lambda handler receives alert analysis request
  │
  ▼
boto3 bedrock-runtime client: InvokeModel
  Model: anthropic.claude-3-sonnet-20240229-v1:0
  Payload: { prompt, max_tokens, messages }
  │ request through Bedrock VPC interface endpoint
  │ never leaves AWS network
  │
  ▼
Bedrock invokes Claude model
  Returns: { content, usage: { input_tokens, output_tokens } }
  │
  ▼
Lambda parses response
  Stores result in PostgreSQL
  Returns formatted response to Kong → Nginx → ALB → Browser
```

---

## 9. EC2 Administrative Access

There is no SSH access, no public port 22, and no bastion host. All EC2 administration is done through AWS SSM Session Manager.

```
DevOps engineer (laptop)
  │
  │ HTTPS — through internet
  ▼
AWS SSM Service (VPC interface endpoint on EC2)
  │ SSM agent running on EC2 instance
  │ Encrypted tunnel established
  ▼
EC2 instance shell
  Admin can: manage Docker containers, run scripts,
             check logs, unseal OpenBao (if deployed),
             run SSM Run Command automation documents

Benefits:
  - All session activity logged to CloudTrail
  - No firewall rules needed for port 22
  - Access controlled entirely by IAM policies
  - Session can be terminated by IAM policy changes immediately
```

---

## 10. CI/CD and Deployment Flow

### AWS Lambda deployment

```
Developer pushes code to GitHub Enterprise
  │ push to develop branch
  ▼
CodeBuild pipeline triggered (branch-based trigger)
  │
  ├── Build phase:
  │     zip function code
  │     upload to S3 (Lambda packages bucket — via S3 VPC gateway endpoint)
  │
  ├── Deploy phase:
  │     aws lambda update-function-code
  │     aws lambda wait function-updated
  │     aws lambda publish-version  → immutable snapshot
  │     aws lambda update-alias     → moves dev/int alias to new version
  │
  └── Notification:
        EventBridge → SNS → email on pipeline failure

Rollback: update alias to previous version — 30-second operation
```

### GCP Cloud Run deployment

```
Developer pushes code to GitHub Enterprise (dedicated repo)
  │ push to main branch
  ▼
Cloud Build trigger fires (connected to GitHub org)
  │
  ├── Build phase:
  │     Cloud Build builds container image
  │     Pushes to Artifact Registry
  │
  └── Deploy phase:
        gcloud run deploy insightgen-api-configurator
        Service account: app-svc-ifrs-fs
        Environment: CLOUD_PROVIDER=GCP, GCP_PROJECT_ID, APP_ENV, APP_VERSION
        Networking: direct VPC egress (private-ranges-only)
```

---

## 11. GCP Request Flow (Cloud Run)

InsightGen also runs a secondary deployment on GCP using Cloud Run HTTP functions. The architecture mirrors the AWS deployment but uses GCP-native services for secrets.

```
Internal Load Balancer (GCP)
  IP: 10.131.36.204
  Route: /int/secrets → Serverless NEG backend
  │
  ▼
Cloud Run function: insightgen-api-configurator
  Service account: app-svc-ifrs-fs@project.iam.gserviceaccount.com
  VPC egress: direct VPC (private-ranges-only)
  Entry point: cloud_run_handler
  │
  │ Cold start — secret fetch
  ▼
GCP Secret Manager (global)
  Secrets:
    insightgen-postgres-host
    insightgen-postgres-port
    insightgen-postgres-dbname
    insightgen-postgres-username
    insightgen-postgres-password
  Encryption: Cloud KMS CMEK (global keyring: insightgen-keyring)
  Access: service account has secretAccessor on all five secrets
  API: secretmanager.googleapis.com (global endpoint)
  │
  ▼
Cloud Run connects to PostgreSQL (sslmode=require)
  Same singleton pattern as AWS — credentials cached after first fetch
  │
  ▼
Response returned through load balancer
```

**Networking note:** The Cloud Run function uses direct VPC egress (`private-ranges-only`). Internal RFC1918 traffic (PostgreSQL connection) routes through the VPC. Google API traffic (Secret Manager, KMS) takes the direct Google internal path. The VPC connector approach was tested and abandoned — the connector's subnet had no firewall rules to reach PostgreSQL.

---

## 12. Security Layers Summary

The platform has multiple security layers. Each layer is independent — a failure at one layer does not automatically expose the system because other layers are still in place.

| Layer | Mechanism | What it protects |
|---|---|---|
| L1 — Network boundary | Private VPC, private subnets, no public IPs on compute | Prevents direct internet access to any backend component |
| L2 — Transport encryption | ALB: TLS 1.2+ to browser. Lambda→PostgreSQL: TLS sslmode=require | Protects data in transit |
| L3 — API authentication | Kong key-auth plugin | Blocks unauthenticated API calls — Lambda never invoked without a valid key |
| L4 — User authentication | Keycloak (active for Angular login) | Controls which users can access the application UI |
| L5 — Secrets protection | SSM SecureString + custom KMS key with explicit deny | Credentials never exist in plaintext in Lambda config |
| L6 — KMS access control | Per-session deny for shared SSO role | Prevents other users sharing the same IAM role from decrypting credentials |
| L7 — Storage encryption | EBS volume encrypted with KMS | PostgreSQL data files encrypted at rest |
| L8 — Admin access control | SSM Session Manager only — no SSH | All admin activity audited, no uncontrolled shell access |
| L9 — Kong Admin API | Bound to 127.0.0.1:8001 only | Kong configuration cannot be changed from Lambda or any VPC resource |
| L10 — Audit | CloudTrail (all AWS API calls) | Every credential access, KMS usage, Lambda invocation recorded |

---

## 13. What Is Deployed But Not Yet Active

| Component | Status | Notes |
|---|---|---|
| MCP Server | Deployed on EC2 | Not yet integrated into any Lambda or Kong flow |
| Keycloak JWT in Kong | Configured | JWT plugin set up in Kong, key-auth is the active plugin. Full JWT flow is planned — Keycloak issues token, Kong validates RS256 signature locally |
| sslmode=verify-ca | Planned | Currently sslmode=require. Upgrade requires ca.crt bundled into Lambda layer |
| Lambda Destinations | Planned | Async failure routing to SNS not yet configured |
| CloudWatch Synthetics canary | Planned | Health check canary not yet deployed |

---

## 14. Glossary

| Term | Meaning in this document |
|---|---|
| ALB | AWS Application Load Balancer — internal, handles HTTPS termination |
| Nginx | Reverse proxy running on EC2 — routes paths to correct containers |
| Kong | Open-source API gateway — handles authentication and Lambda routing |
| key-auth | Kong plugin — validates API keys on every request |
| JWT | JSON Web Token — signed token issued by Keycloak carrying user identity |
| SSM | AWS Systems Manager Parameter Store — secure credential storage |
| KMS | AWS Key Management Service — encryption key management |
| SecureString | SSM parameter type — value encrypted by KMS |
| sslmode=require | PostgreSQL client setting — enforces TLS, does not verify server cert |
| sslmode=verify-ca | PostgreSQL client setting — enforces TLS and verifies server cert against CA |
| Singleton pattern | Python pattern — module-level variable loaded once, reused on warm invocations |
| Cold start | First Lambda invocation after container creation — slower due to setup |
| Warm invocation | Subsequent Lambda call to existing container — no setup overhead |
| VPC endpoint | Private connection from VPC to AWS service — traffic stays in AWS network |
| CMEK | Customer-managed encryption key — KMS key you control, not Google's default |
| Serverless NEG | GCP Network Endpoint Group — connects load balancer to Cloud Run |
| direct VPC egress | Cloud Run networking — traffic to private IPs routes through VPC directly |
