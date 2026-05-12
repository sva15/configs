# IFRS InsightGen — HTTPS Migration Task List

---

## Task 1 — AWS ALB Port 443 Listener Validation and Default Route Verification

**Status:** ✅ Completed

- Identified that the internal ALB (`internal-Insightgen-ALB-716509456`) had two listeners — port 80 (active with all rules) and port 443 (configured with SSL certificate but no routing rules)
- Confirmed that the SSL certificate for `ifrs-insightgen.com` (ACM, valid until Feb 2027) was already attached to the 443 listener
- Identified that all traffic types were on port 80 — both IP-based (frontend + Kong) and Lambda-based (direct API invocations)
- Determined that for the 443 listener, only IP-based target groups are needed — Lambda-based target group rules are not required since Kong API Gateway will handle all API routing internally
- Verified that the default listener rule on port 443 pointing to `Insightgen-IPbased-tg` (IP: `10.132.191.157:80`, Status: Healthy) is **sufficient to handle all paths** — individual ALB rules per Angular application are not needed because Nginx on the EC2 instance handles all path-based routing internally
- Confirmed HTTPS is working by accessing `https://ifrs-insightgen.com/dev/synthetic-ui/alert-dashboard` successfully from AWS AVD → EC2 Windows instance (same VPC)
- Concluded that **no further changes are needed from the ALB/Network team**

---

## Task 2 — DNS Configuration for Domain `ifrs-insightgen.com`

**Status:** ✅ Completed

- Created a Route 53 **public hosted zone** for `ifrs-insightgen.com`
- Created an **A record (Alias)** pointing `ifrs-insightgen.com` to the ALB: `dualstack.internal-insightgen-alb-716509456.us-east-1.elb.amazonaws.com`
- Understood that since the ALB is internal, the domain resolves to a **private VPC IP** — users outside the VPC/network resolve the DNS but cannot reach the private IP, which is the intended behavior
- Confirmed DNS works correctly from inside the VPC (accessible via AWS AVD → EC2 Windows instance)
- Identified that access from **developer laptops** requires the Cisco ZTNA team to enable routing for the `ifrs-insightgen.com` domain so that VPN/ZTNA tunnels the traffic to the private VPC network

---

## Task 3 — Cisco ZTNA Team — Enable Domain Access from Developer Laptops

**Status:** 🔲 Pending — Need to raise request

- Currently `https://ifrs-insightgen.com` is only accessible from within the VPC (AWS AVD → EC2 Windows instance)
- Direct access from developer laptops via browser does not work because the domain resolves to a private IP that is unreachable without VPN/ZTNA tunnel
- Previously accessing applications using the raw internal ALB URL over HTTP from AWS AVD was working — domain-based HTTPS access requires ZTNA to recognize the new domain
- **Action:** Raise a request with the Cisco ZTNA team to enable the domain `ifrs-insightgen.com` so that traffic from developer laptops is tunnelled through ZTNA to the internal VPC network
- Information to share with ZTNA team:
  - Domain: `ifrs-insightgen.com`
  - Resolves to: Private VPC IP (internal ALB)
  - Port required: 443 (HTTPS)
  - VPC: `vpc-07630ba9cc78ba2db` (us-east-1)

---

## Task 4 — Nginx Configuration Fixes on EC2 Instance

**Status:** 🔲 Pending

- Reviewed the existing Nginx `default.conf` running inside the `nginx:alpine` Docker container on the EC2 instance (`10.132.191.157`)
- Identified the following issues that must be fixed before HTTPS is fully functional:

**Fix 1 — Keycloak `X-Forwarded-Port` Header (Critical)**
- Identified that Nginx currently sets `X-Forwarded-Port 80` and `X-Forwarded-Proto $scheme` for the `/auth/` location
- This will cause Keycloak to generate post-login redirect URLs with `http://` and port `80`, breaking the login flow in all Angular applications
- Must change `X-Forwarded-Port` to `443` and `X-Forwarded-Proto` to hardcoded `https` in the Nginx `/auth/` location block

**Fix 2 — Add Missing `/framework/` Location Block**
- Identified that the Nginx config has a location block for `/dev/framework/` (port 4306) but no block for `/framework/`
- The `insightgen-domain-framework` container serves both int and dev on port 4306
- Must add a `/framework/` location block pointing to `172.17.0.1:4306`

**Fix 3 — Unlock Kong Admin API Security**
- Identified that the IP allow/deny rules for the `/kong-admin-api/` location are commented out in Nginx
- Kong Admin API is currently accessible to anyone who can reach the EC2 instance
- Must uncomment and configure IP restrictions to restrict access to VPC CIDR only

- After all Nginx config changes: run `sudo nginx -s reload` inside or outside the Nginx container to apply

---

## Task 5 — Fix `/configurator/` Application Bug (Port Mismatch)

**Status:** 🔲 Pending

- Identified a bug by cross-referencing the Nginx config with the running Docker containers list
- Nginx routes `/configurator/` (int environment) to port `4305` on the host, but there is **no Docker container mapped to port 4305**
- The only running configurator container is `ifrs-configurator-frontend` mapped to port `4304`, which serves `/dev/configurator/`
- `/configurator/` currently returns **502 Bad Gateway**
- Two resolution options identified:
  - Option A: Start a new container instance of the configurator image mapped to port 4305 for the int environment
  - Option B: If int and dev share the same app, update Nginx to point `/configurator/` to port 4304

---

## Task 6 — Kong API Gateway Route Configuration for All Lambda Functions

**Status:** 🔲 Pending

- Identified that on port 80, Lambda functions are invoked directly via dedicated ALB listener rules (one rule per Lambda, ~44 Lambda target groups)
- On port 443 this pattern is replaced entirely by Kong — a single ALB rule handles all API traffic via `/kong-api/*` which routes to Nginx → Kong
- Nginx strips the `/kong-api/` prefix (via trailing slash in `proxy_pass`) before forwarding to Kong on port 9000, so Kong receives the path starting from `/int/` or `/dev/`
- Kong uses the **AWS Lambda plugin** to invoke Lambda functions — no HTTP endpoint needed, invocation is direct via AWS SDK
- Identified all 44 routes that need to be configured in Kong, mapping each `/int/` and `/dev/` path to its corresponding Lambda function name
- For each Lambda, need to create: one Kong **Service** (placeholder), one Kong **Route** (with path), one **AWS Lambda plugin** instance (with function name, region, invocation type)

**Key configurations per route:**
- `invocation_type`: RequestResponse (synchronous)
- `forward_request_body`, `forward_request_headers`, `forward_request_method`, `forward_request_uri`: all `true`
- `aws_region`: us-east-1
- IAM auth: use EC2 instance role (no hardcoded keys)

**Full Route List (Kong Path → Lambda Function):**

| Kong Route Path | Lambda Function |
|---|---|
| `/int/getai/` | `int-getalertsummary` |
| `/dev/getai/` | `dev-getalertsummary` |
| `/int/get/` | `int-getdatapg` |
| `/dev/get/` | `dev-getdatapg` |
| `/config/` | `int-insightgen-api-config` |
| `/dev/config/` | `dev-insightgen-api-config` |
| `/int/store/` | `int-storedatapg` |
| `/dev/store/` | `dev-storedatapg` |
| `/int/ifrs-syn-generator` | `int-syntheticdatagenerator` |
| `/dev/ifrs-syn-generator` | `dev-syntheticdatagenerator` |
| `/int/refresh` | `int-updatedsyntheticdata` |
| `/dev/refresh` | `dev-updatedsyntheticdata` |
| `/int/reference/` | `int-referenceData` |
| `/dev/reference/` | `dev-referenceData` |
| `/int/poll/` | `int-pollData` |
| `/dev/poll/` | `dev-pollData` |
| `/dev/analyze` | `dev-analyzesyntheticdata` |
| `/int/analyze` | `int-analyzesyntheticdata` |
| `/dev/scenario-creation/` | `dev-scenarioCreation` |
| `/int/scenario-creation/` | `int-scenarioCreation` |
| `/dev/retrieve-data/` | `dev-retrieveData` |
| `/int/retrieve-data/` | `int-retrieveData` |
| `/dev/store-data/` | `dev-storeData` |
| `/int/store-data/` | `int-storeData` |
| `/dev/scenario-functions/` | `dev-scenarioFunctions` |
| `/int/scenario-functions/` | `int-scenarioFunctions` |
| `/dev/map/` | `dev-map` |
| `/int/map/` | `int-map` |
| `/int/insightgen-agent/` | `int-agent` |
| `/dev/insightgen-agent/` | `dev-agent` |
| `/int/audit-log/` | `int-audit` |
| `/dev/audit-log/` | `dev-audit` |
| `/dev/synthetic-data/` | `dev-syntheticData` |
| `/int/synthetic-data/` | `int-syntheticData` |
| `/int/scenario/` | `int-scenario` |
| `/dev/scenario/` | `dev-scenario` |
| `/dev/analysis/` | `dev-analysis` |
| `/int/analysis/` | `int-analysis` |
| `/dev/synthetic-data-publisher/` | `dev-syn-data-publish` |
| `/int/synthetic-data-publisher/` | `int-syn-data-publish` |
| `/dev/insightgen-agenticAnalysis/` | `dev-agent-analysis` |
| `/int/insightgen-agenticAnalysis/` | `int-agent-analysis` |

---

## Task 7 — Kong API Key Authentication Setup

**Status:** 🔲 Pending

- Identified that Kong was chosen over AWS API Gateway due to project restrictions on using AWS API Manager
- Kong's **key-auth plugin** will be used to secure all API routes — every API call must include a valid API key in the request header
- Plan to apply the key-auth plugin at the **global level** in Kong so all routes are automatically secured without configuring it per service
- Need to create Kong **Consumers** (one per application or team) and generate API keys for each
- API calls from Angular applications must include the header: `apikey: <generated-key>`
- Identified that the `abuseList` API call from `insightgen-syn-generator` (seen in the Mixed Content error) will need a valid API key once key-auth is enabled — Angular apps must include the key in their HTTP service calls

---

## Task 8 — EC2 IAM Role Configuration for Kong Lambda Invocation

**Status:** 🔲 Pending

- Identified that Kong uses the **AWS Lambda plugin** which requires AWS credentials to invoke Lambda functions
- Determined that the best approach is attaching an **IAM instance role** to the EC2 instance instead of hardcoding AWS access keys inside Kong configuration
- IAM policy to attach to the EC2 role uses wildcards to cover all int and dev Lambda functions:
  - Allow `lambda:InvokeFunction` on `arn:aws:lambda:us-east-1:<account-id>:function:int-*`
  - Allow `lambda:InvokeFunction` on `arn:aws:lambda:us-east-1:<account-id>:function:dev-*`
- Need to verify whether the EC2 instance already has an instance role attached — if not, create and attach one

---

## Task 9 — Angular Application API Base URL Migration from HTTP to HTTPS

**Status:** 🔲 In Progress (first issue identified and root cause confirmed)

- Confirmed HTTPS is working by successfully loading `https://ifrs-insightgen.com/dev/synthetic-ui/alert-dashboard`
- Identified **Mixed Content error** in Chrome DevTools — the Angular app (`insightgen-syn-generator`) is making API calls to:
  `http://internal-insightgen-alb-716509456.us-east-1.elb.amazonaws.com/dev/analysis/retrieve/abuseList`
- Browser blocks this because an HTTPS page cannot make HTTP requests (Mixed Content policy)
- Root cause: Angular environment config has the **raw internal ALB URL with `http://`** hardcoded as the API base URL
- Fix identified: Update `environment.ts` in each Angular app to use `https://ifrs-insightgen.com/kong-api` as the API base URL
- All Angular applications need this fix — they all have the same hardcoded HTTP ALB URL pattern:

| Container | Application Path | Environment File |
|---|---|---|
| `insightgen-frontend` | `/ui/` | `environment.ts` |
| `insightgen-syn-generator-keycloak-test` | `/synthetic-ui/` | `environment.ts` |
| `insightgen-syn-generator` | `/dev/synthetic-ui/` | `environment.ts` ← current issue |
| `ifrs-configurator-frontend` | `/dev/configurator/` | `environment.ts` |
| `insightgen-domain-framework` | `/framework/`, `/dev/framework/` | `environment.ts` |

- After updating each environment file: rebuild Docker image → stop old container → run new container on same port
- **Prerequisite:** Kong route for the corresponding API path must be configured and tested before rebuilding (Task 6 must be done first)

---

## Task 10 — Keycloak Configuration Update for HTTPS

**Status:** 🔲 Pending

- Identified that all Angular applications use **Keycloak** for user authentication (login, session, token management)
- Identified a critical Nginx config issue: `X-Forwarded-Port` is hardcoded to `80` — Keycloak uses this header to build its redirect URLs, so post-login redirects will point to `http://` port `80` instead of `https://` port `443`, breaking login for all apps
- Two sets of changes needed:

**Nginx fix (covered in Task 4):**
- Change `X-Forwarded-Port` from `80` to `443` in the `/auth/` location block
- Change `X-Forwarded-Proto` from `$scheme` to hardcoded `https`

**Keycloak Admin Console (for each registered client):**
- Update **Valid Redirect URIs** from `http://ifrs-insightgen.com/*` to `https://ifrs-insightgen.com/*`
- Update **Web Origins** from `http://ifrs-insightgen.com` to `https://ifrs-insightgen.com`
- Update **Admin URL** from `http://` to `https://`

**Angular app Keycloak init config (in each app):**
- Update Keycloak `url` from `http://ifrs-insightgen.com/auth` to `https://ifrs-insightgen.com/auth`
- Applies to all 5 Angular containers listed in Task 9

---

## Task 11 — End-to-End Testing After All Changes

**Status:** 🔲 Pending

- Once Tasks 4 through 10 are complete, perform full end-to-end validation:

**Frontend access:**
- Each Angular app loads correctly over `https://ifrs-insightgen.com/<path>`
- No Mixed Content errors in Chrome DevTools Console tab
- No HTTP requests visible in Chrome DevTools Network tab (all should be HTTPS)

**Keycloak login flow:**
- Login page loads at `https://ifrs-insightgen.com/auth/`
- After login, redirect returns to `https://ifrs-insightgen.com/<app-path>` (not `http://`)
- Token is issued and stored correctly in the Angular app

**API calls via Kong:**
- API calls from Angular apps go to `https://ifrs-insightgen.com/kong-api/<route>`
- Kong routes the call to the correct Lambda function
- Lambda returns a valid response
- Call without API key returns `401 Unauthorized`
- Call with valid API key returns expected data

**Access from developer laptops (after ZTNA is enabled):**
- `https://ifrs-insightgen.com/ui/` accessible from laptop browser without AWS AVD
- Full login and API flow works end to end from laptop

---

*Last updated based on active migration session and confirmed HTTPS access to `https://ifrs-insightgen.com/dev/synthetic-ui/alert-dashboard`*
