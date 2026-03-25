Updated the two sections with your answers — here's the final complete document:

---

**Promotion Case — DevOps & Cloud Engineer**

*Everything listed below was owned, implemented, and delivered independently.*

---

## 🏆 Certifications — Completed While Delivering All of the Below

| Certification | Provider |
|---|---|
| AWS Certified DevOps Engineer – Professional | Amazon Web Services |
| AWS Certified AI Practitioner | Amazon Web Services |
| AZ-400: Microsoft Azure DevOps Engineer Expert | Microsoft |

---

## PROJECT 1 — Novus BiaB (Azure)

**CI/CD & DevSecOps**

- Designed and built Jenkins CI/CD pipelines for end-to-end application deployments across all environments — covering code checkout, build, test, containerisation, and deployment stages, giving the team a fully automated release process.
- Refactored all existing Jenkins pipelines using Jenkins Shared Libraries — extracted common logic into reusable functions so every pipeline references the same core code. This eliminated duplication, made bug fixes apply everywhere at once, and significantly reduced the time to create new pipelines.
- Implemented parameterised multi-environment builds so a single pipeline definition handles Dev, QA, UAT, and Production deployments — no separate pipeline per environment, no configuration drift between them.
- Embedded a full DevSecOps security scanning stage into every pipeline:
  - SAST (Static Application Security Testing) scans application source code at build time to catch vulnerabilities before they are ever packaged.
  - SCA (Software Composition Analysis) scans all third-party libraries and open-source dependencies for known CVEs, ensuring the team is not shipping vulnerable packages.
  - Trivy scans every Docker container image for OS-level and application-level vulnerabilities before the image is pushed or deployed.
  - Configured automated build gates — if any critical or high-severity issue is found, the pipeline fails immediately and deployment is blocked, enforcing a zero-tolerance policy on known vulnerabilities reaching production.
- Implemented HTTPS/TLS encryption across all Novus BiaB applications for both Dev and Production environments — configured SSL certificates, updated NGINX and application configs, and validated end-to-end encrypted traffic.

**Cloud Infrastructure & IaC**

- Serving as the full Azure Administrator for Novus BiaB — solely responsible for all cloud operations including virtual machine provisioning and management, virtual networking (VNets, subnets, NSGs, peering), storage accounts, Azure Active Directory, RBAC role assignments, and resource group governance.
- Wrote comprehensive Terraform Infrastructure as Code to fully automate the deployment of the entire Novus BiaB project into multiple Azure accounts. Every resource — compute, networking, storage, identity, and application infrastructure — is defined in code, version-controlled in Git, and deployable in a single run. This eliminated manual provisioning entirely and made environment creation fully repeatable.
- Built multi-environment Terraform configurations (Dev, QA, UAT, Production) using Terraform workspaces and variable files — each environment is independently deployable with its own configuration values, eliminating environment drift and manual configuration errors.
- Worked with Azure ARM templates for specific infrastructure provisioning and environment configuration tasks — reading, understanding, modifying, and deploying ARM template definitions.
- **Azure Landing Zone Migration** — independently planned and executed the migration of the entire Azure subscription into an Azure Landing Zone. This was one of the most complex and high-risk tasks undertaken:
  - Remapped all network IP address spaces and CIDRs as the Landing Zone introduced a new hub-and-spoke network topology.
  - Updated all DNS configurations so every service and application remained reachable after the network changes.
  - Reconfigured all application-level settings referencing environment-specific IPs, hostnames, and endpoints.
  - Updated every CI/CD pipeline to reflect the new subscription structure, resource IDs, and service principal permissions.
  - Migrated all Terraform state files to align with the new subscription, resource group, and networking structure.
  - Coordinated closely with networking, security, and application teams throughout the migration.
  - Carefully sequenced all changes and validated each step to ensure zero unplanned downtime throughout the entire migration window.
  - The result was a more secure, governed, and scalable cloud foundation aligned with enterprise Landing Zone best practices.

**Cost Optimisation**

- Independently audited the entire Novus BiaB Azure environment for cost inefficiencies — identified oversized VMs, unused resources, unattached disks, and idle services.
- Executed rightsizing of compute resources, cleaned up unused infrastructure, and optimised storage tiers — **delivering approximately ₹30,000 in cost savings**, entirely self-driven with no direction from management.

**Observability & Monitoring**

- Designed and implemented a full observability stack from scratch covering all Azure VMs and all containerised workloads running on those VMs:
  - **ELK Stack (Elasticsearch, Logstash, Kibana)** — deployed and configured centralised log aggregation from all services. Set up Logstash pipelines to parse, transform, and index logs. Built Kibana dashboards and index patterns enabling structured log search and analysis across the entire platform.
  - **Prometheus** — deployed for time-series metrics collection. Configured Node Exporter on all VMs for system-level metrics (CPU, memory, disk, network) and cAdvisor on all container hosts for container-level resource visibility.
  - **Grafana** — built comprehensive operational dashboards aggregating Prometheus metrics, giving the team real-time visibility into system health, resource utilisation, and application performance at both infrastructure and container level.
  - **SigNoz** — implemented as the APM and distributed tracing solution. Configured applications to emit traces, enabling end-to-end request tracing across microservices to identify service-to-service latency, slow queries, and performance bottlenecks.
- Configured alerting rules in Prometheus Alertmanager and Grafana — defined thresholds covering CPU and memory pressure, container/pod crash loops, HTTP error rate spikes, and disk space exhaustion. All alerts routed to a dedicated Microsoft Teams channel for immediate team notification.

**Platform & Application Operations**

- **NGINX Reverse Proxy** — maintaining all NGINX reverse proxy configurations. Responsible for onboarding new API routes, updating upstream configurations, managing SSL termination, and applying configuration changes with zero-downtime reloads.
- **WSO2 API Manager** — managing the full API lifecycle in WSO2 including publishing APIs, configuring throttling and rate-limiting policies, managing subscriber access and subscription tiers, and enforcing security policies on the API gateway.
- **Keycloak** — deploying and maintaining Keycloak as the identity and access management platform. Managing multiple realms, configuring OAuth2/OIDC clients, defining roles and role mappings, and maintaining authentication flows for each application.
- **Self-hosted GitLab** — running GitLab as a containerised application within the Azure environment. Responsible for container health, upgrades, backup configuration, repository management, CI/CD runner configuration, and access control.
- **Kestra** — deploying and maintaining Kestra as the workflow orchestration platform for automating data and operational workflows within the project.
- **Kafka** — maintaining the Apache Kafka messaging and event streaming layer — managing brokers, topics, consumer groups, and ensuring reliability of the event pipeline across services.
- **Temporal** — maintaining the Temporal workflow engine — managing workers, namespaces, and monitoring workflow health for durable, fault-tolerant process execution.
- **Database Management** — maintaining all project databases across environments including provisioning, configuration tuning, access control, backup schedules, performance monitoring, and encryption at rest.
- **JIRA Administration** — maintaining JIRA dashboards, boards, and project configurations. Customising dashboards to give project leads and stakeholders clear visibility into delivery progress, sprint health, and issue status.

---

## PROJECT 2 — Novus CBA (AWS)

- Solely maintaining the entire AWS account for the Novus CBA team — responsible for all day-to-day cloud operations, account governance, IAM management, and security compliance.
- Wrote **CloudFormation templates** to define, provision, and manage all AWS infrastructure as code — covering compute, networking, storage, and application resources, making the environment fully reproducible and version-controlled.
- Designed and implemented CI/CD pipelines for all application deployments — applying the same DevOps best practices and pipeline standards established in Novus BiaB.
- **Kong API Manager** — deploying and maintaining Kong as the API gateway. Managing API routes, plugins (authentication, rate limiting, logging), upstream services, and consumer configurations.
- Maintaining **self-hosted GitLab** for the CBA team — managing repositories, access control, branch protection rules, CI/CD runner integrations, and ensuring platform reliability.
- Configured and maintained a cloud-hosted **MongoDB instance from the AWS Marketplace** — handling provisioning, connection configuration, access control, and integration with CBA applications.
- Coordinating with AWS cloud and security teams to ensure all workloads remain compliant with governing policies.
- Actively identifying and resolving security findings and misconfigurations across the AWS environment.

---

## PROJECT 3 — InsightGen (GCP + AWS)

**GitHub Enterprise Administration**

- Administering the entire GitHub Enterprise organisation for InsightGen — managing all code repositories, team structures, and member access across the organisation.
- Configuring and enforcing branch protection rules, CODEOWNERS files, required reviewers, and status checks across all repositories to maintain code quality and prevent unauthorised changes.
- Enforcing repository naming conventions, hygiene standards, and archival policies across the organisation.
- Managing GitHub Actions workflows and integrating them with the broader CI/CD platform.
- Coordinating with the global GIT governance and security teams to understand organisation-wide policies and implementing them correctly within the GitHub Enterprise environment.
- Proactively monitoring for secrets accidentally committed to repositories and executing remediation — including secret rotation, commit history cleanup, and enforcing secret scanning policies.

**GCP — Serverless Infrastructure & Operations**

- Designed and built the entire serverless deployment infrastructure on GCP from scratch — using Cloud Run for containerised API and backend services and Cloud Run Functions for event-driven and lightweight function workloads.
- Implemented automated CI/CD deployment pipelines for all APIs and frontend applications into GCP — covering build, test, containerisation, and deployment to the correct Cloud Run service and environment.
- Maintained fully separate Dev, Staging, and Production environments on GCP — each with independent configurations, service accounts, and access controls to prevent cross-environment impact.
- Collaborated with GCP cloud governance, networking, and the global GIT security teams to understand platform-level policies and implemented them correctly within the GCP project and IAM structure.
- **GCP Cost Optimisation** — continuously monitored GCP billing and resource usage. Identified over-provisioned Cloud Run configurations, idle services, and inefficient resource allocations. **Reduced daily GCP spend from $14/day down to $3/day — saving over $330/month.**

**AWS — AI Workloads, Infrastructure & Operations**

- Deployed the **AIForce application** into InsightGen project environments — set up the deployment pipeline, configured the foundational services AIForce depends on, and integrated those services into the existing InsightGen infrastructure and workflows.
- Working with **AWS Bedrock** — managing AI model access, supporting the team in consuming Bedrock-powered services, and ensuring the infrastructure around AI workloads is reliable and cost-controlled.
- Built **Amazon CloudWatch dashboards** to visualise AWS Bedrock model token usage across all models in use — giving the team real-time visibility into AI consumption patterns and helping forecast costs.
- Created **CloudWatch metric alarms** on Bedrock token consumption to detect and notify on unusual spikes immediately — preventing runaway AI usage from causing unexpected cost overruns.
- **Kong API Manager** — deploying and maintaining Kong as the API gateway for the InsightGen AWS environment, managing API routes, plugins, authentication, rate limiting, and upstream configurations.
- **Keycloak** — maintaining Keycloak for authentication and authorisation across InsightGen AWS environments — managing realms, OIDC clients, role mappings, and access policies for all applications and services.
- Implemented **encryption in transit and at rest** for all databases on AWS — configured TLS connections, enabled storage encryption, and validated that no unencrypted data pathways existed across the environment.
- Replicated the entire InsightGen application stack from GCP onto AWS — maintaining both cloud environments in parallel with consistent deployment standards, configuration patterns, and operational practices.
- Coordinating with AWS cloud, networking, and security teams on governance requirements, compliance standards, and security policy implementation.
- **AWS Cost Optimisation** — continuously monitored AWS billing, Cost Explorer, and resource utilisation. Right-sized compute, cleaned up unused resources, optimised data transfer, and reviewed service configurations. **Reduced monthly AWS spend from $1,500/month to $500/month — saving $1,000/month.**

**AI & Emerging Technology**

- Deployed the **AIForce application** and worked on its foundational services — integrating them into the InsightGen infrastructure so the team could consume AI capabilities reliably across environments.
- Supported a separate team developing an **OpenAI RAG (Retrieval-Augmented Generation) POC** — contributed to the technical implementation, helped work through integration challenges between OpenAI APIs, vector storage, and document retrieval pipelines.
- **Keycloak for the OpenAI team** — configured and maintained Keycloak to provide authentication and authorisation for the OpenAI POC application — setting up the realm, OIDC clients, and access policies required by the team independently.

---

## CROSS-TEAM SUPPORT

**Alphagen Team**

- Provisioned AWS access for the Alphagen team — created IAM users, roles, and permission policies following the principle of least privilege, ensuring the team had exactly the access they needed to operate securely without unnecessary exposure.
- Supported the team through their complete initial AWS environment setup — provisioned and configured EC2 instances, set up networking (VPCs, subnets, security groups), configured storage, and ensured all foundational infrastructure was in place for the team to begin their work.
- Onboarded the Alphagen team into the GitHub Enterprise organisation — created their repositories, configured team structures, assigned appropriate access levels, set up branch protection rules and CODEOWNERS files, and ensured all repos followed organisational standards from day one.
- Deployed Alphagen team applications onto AWS EC2 — set up the server environments, configured application dependencies, established deployment processes, and validated applications were running correctly in the target environment.
- Continued maintaining Alphagen's repositories within the GitHub Enterprise organisation on an ongoing basis — managing access changes, enforcing repository standards, and supporting any repository-level needs as the team grows.

**Thought Machine Team**

- Supporting the Thought Machine team as a standalone project — acting as their DevOps and Cloud point of contact for infrastructure and access needs.
- Managing all cloud access and permissions for the Thought Machine team — provisioning IAM roles and policies, granting environment access, and ensuring access is correctly scoped and reviewed on an ongoing basis.
- Supporting the team with cloud infrastructure setup — advising on architecture decisions, provisioning required cloud resources, and ensuring the environment is correctly configured to support their project activities.
- Helping the team navigate cloud governance requirements and ensuring their infrastructure aligns with organisational policies and security standards.

---

## Across All Projects

**Security & Compliance**
- Independently identified and resolved security vulnerabilities and misconfigurations across Azure, AWS, and GCP — across all projects.
- Enforced least-privilege IAM policies across all cloud accounts, ensuring no over-permissioned roles or service accounts exist.
- Implemented encryption in transit and at rest across all database and data storage layers.
- DevSecOps pipeline controls prevent any insecure build from reaching production across all projects.
- Coordinated with cloud and security governance teams across all three cloud providers to maintain compliance.

**Individual Ownership & Leadership**
- Acting as the sole DevOps and Cloud owner across three simultaneous projects — Novus BiaB, Novus CBA, and InsightGen — while also supporting the Alphagen and Thought Machine teams.
- Every task listed in this document was independently taken on, figured out, and delivered — without waiting for direction or guidance.
- Proactively coordinating across cloud, networking, security, governance, and application teams across Azure, AWS, and GCP.
- Actively guiding and unblocking junior team members — sharing knowledge, reviewing their work, and helping them build capability.

---

## 💰 Total Cost Savings Delivered

| Platform | Before | After | Monthly Saving |
|---|---|---|---|
| Azure — Novus BiaB | — | — | ₹30,000 saved |
| GCP — InsightGen | $14/day (~$420/month) | $3/day (~$90/month) | **~$330/month** |
| AWS — InsightGen | $1,500/month | $500/month | **$1,000/month** |

---

Want me to now export this as a clean Word document you can attach and send to your manager?
