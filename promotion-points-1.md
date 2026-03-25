---

# Promotion Case — DevOps & Cloud Engineer

## Executive Summary

Over the past period, I have operated as the **sole DevOps and Cloud owner across three simultaneous live projects** — Novus BiaB on Azure, Novus CBA on AWS, and InsightGen across GCP and AWS — while also supporting two additional teams. Every workstream listed below was independently owned, executed, and delivered without waiting for direction. Alongside delivery, I have consistently supported and guided other engineers through technical challenges, actively unblocking them and helping them grow.

In parallel, I earned three industry certifications while managing all of the below at full capacity.

---

## Certifications

| Certification | Provider |
|---|---|
| AWS Certified DevOps Engineer – Professional | Amazon Web Services |
| AWS Certified AI Practitioner | Amazon Web Services |
| AZ-400: Microsoft Azure DevOps Engineer Expert | Microsoft |

---

## Team Leadership & Peer Support

While I operate as an individual devops/cloud operations member across all three projects, I actively support junior and peer engineers whenever they face technical blockers. This includes troubleshooting issues they are stuck on, explaining concepts and approaches, reviewing their work, and guiding them toward the right solution rather than just handing them answers. My goal is to help the people around me grow their capability, not just get things done. This support spans all three cloud providers and covers infrastructure, CI/CD, security, and platform operations — wherever the team needs help.

---

## Delivered Cost Savings

| Platform | Before | After | Saving |
|---|---|---|---|
| Azure — Novus BiaB | — | — | ₹30,000 |
| GCP — InsightGen | ~$420/month | ~$90/month | ~$330/month |
| AWS — InsightGen | $1,500/month | $500/month | $1,000/month |

All cost optimisation was self-initiated — no direction from management in any case.

---

## Project 1 — Novus BiaB (Azure)

**CI/CD & DevSecOps**

Designed and built Jenkins CI/CD pipelines covering the full release cycle — code checkout, build, test, containerisation, and deployment across all environments. Refactored all existing pipelines using Jenkins Shared Libraries, extracting common logic into reusable functions so every pipeline references the same core code — eliminating duplication and making bug fixes apply everywhere at once.

Implemented parameterised multi-environment builds so a single pipeline definition handles both Dev and Production — no separate pipeline per environment, no configuration drift between them.

Embedded a full DevSecOps security scanning stage into every pipeline: SAST for source code vulnerabilities, SCA for third-party library CVEs, and Trivy for container image scanning at OS and application level. Configured automated build gates — any critical or high-severity finding blocks deployment immediately, enforcing zero-tolerance on known vulnerabilities reaching production.

Implemented HTTPS/TLS encryption across all Novus BiaB applications for both Dev and Production — configured SSL certificates, updated NGINX and application configs, and validated end-to-end encrypted traffic.

**Cloud Infrastructure & IaC**

Acting as the full Azure Administrator for Novus BiaB — solely responsible for all cloud operations including VM provisioning, virtual networking (VNets, subnets, NSGs, peering), storage accounts, Azure Active Directory, RBAC, and resource group governance.

Wrote comprehensive Terraform IaC to fully automate deployment of the entire Novus BiaB environment across multiple Azure accounts — every resource defined in code, version-controlled, and deployable in a single run. Built multi-environment configurations using Terraform workspaces and variable files, making Dev and Production independently deployable with zero configuration drift. Also worked with Azure ARM templates for specific infrastructure provisioning and environment configuration tasks.

**Azure Landing Zone Migration** — independently planned and executed the full migration of the Azure subscription into an Azure Landing Zone. This was one of the most complex and high-risk tasks undertaken:

- Remapped all network IP address spaces and CIDRs as the Landing Zone introduced a new hub-and-spoke network topology.
- Updated all DNS configurations so every service and application remained reachable after the network changes.
- Reconfigured all application-level settings referencing environment-specific IPs, hostnames, and endpoints.
- Updated every CI/CD pipeline to reflect the new subscription structure, resource IDs, and service principal permissions.
- Migrated all Terraform state files to align with the new subscription, resource group, and networking structure.
- Coordinated closely with networking, security, and application teams throughout the migration.
- Carefully sequenced all changes and validated each step — zero unplanned downtime across the entire migration window.
- The result was a more secure, governed, and scalable cloud foundation aligned with enterprise Landing Zone best practices.

**Cost Optimisation**

Independently audited the entire Novus BiaB Azure environment for cost inefficiencies — identified oversized VMs, unused resources, unattached disks, and idle services. Executed rightsizing of compute resources, cleaned up unused infrastructure, and optimised storage tiers — **delivering approximately ₹30,000 in cost savings**.

**Observability & Monitoring**

Designed and implemented a full observability stack from scratch covering all Azure VMs and all containerised workloads:

- **ELK Stack** — centralised log aggregation with Logstash pipelines parsing and indexing logs, Kibana dashboards for structured log search and analysis across the entire platform.
- **Prometheus + Node Exporter + cAdvisor** — system-level metrics collection across all VMs and container-level resource visibility across all container hosts.
- **Grafana** — comprehensive operational dashboards giving the team real-time visibility into system health, resource utilisation, and application performance at both infrastructure and container level.
- **SigNoz** — APM and distributed tracing for end-to-end request visibility across microservices, identifying service-to-service latency, slow queries, and performance bottlenecks.

Configured alerting in Prometheus Alertmanager and Grafana covering CPU and memory pressure, container crash loops, HTTP error rate spikes, and disk exhaustion — all routed to a dedicated Microsoft Teams channel for immediate team notification.

**Platform Operations**

Maintaining all platform components across the Novus BiaB project:

- **NGINX Reverse Proxy** — onboarding new API routes, updating upstream configurations, managing SSL termination, and applying configuration changes with zero-downtime reloads.
- **WSO2 API Manager** — managing the full API lifecycle including publishing APIs, configuring throttling and rate-limiting policies, managing subscriber access and subscription tiers, and enforcing security policies on the API gateway.
- **Keycloak** — deploying and maintaining Keycloak as the identity and access management platform, managing multiple realms, configuring OAuth2/OIDC clients, defining roles and role mappings, and maintaining authentication flows for each application.
- **Self-hosted GitLab** — running GitLab as a containerised application within the Azure environment, responsible for container health, upgrades, backup configuration, repository management, and access control.
- **Kestra** — deploying and maintaining Kestra as the workflow orchestration platform for automating data and operational workflows.
- **Kafka** — maintaining the Apache Kafka messaging and event streaming layer — managing brokers, topics, consumer groups, and ensuring reliability of the event pipeline across services.
- **Temporal** — maintaining the Temporal workflow engine — managing workers, namespaces, and monitoring workflow health for durable, fault-tolerant process execution.
- **Database Management** — maintaining all project databases across Dev and Production including provisioning, configuration tuning, access control, backup schedules, performance monitoring, and encryption at rest.
- **JIRA Administration** — maintaining JIRA dashboards, boards, and project configurations, customising dashboards to give project leads and stakeholders clear visibility into delivery progress, sprint health, and issue status.

---

## Project 2 — Novus CBA (AWS)

Solely maintaining the entire AWS account for the Novus CBA team — responsible for all day-to-day cloud operations, account governance, IAM management, and security compliance. Wrote CloudFormation templates to define, provision, and manage all AWS infrastructure as code — covering compute, networking, storage, and application resources, making the environment fully reproducible and version-controlled.

Designed and implemented CI/CD pipelines for all application deployments — applying the same DevOps best practices and pipeline standards established in Novus BiaB.

Managing **Kong API Gateway** — deploying and maintaining Kong as the API gateway, managing API routes, plugins (authentication, rate limiting, logging), upstream services, and consumer configurations.

Maintaining **self-hosted GitLab** for the CBA team — managing repositories, access control, branch protection rules, CI/CD runner integrations, and ensuring platform reliability.

Configured and maintained a cloud-hosted **MongoDB instance from the AWS Marketplace** — handling provisioning, connection configuration, access control, and integration with CBA applications.

Coordinating with AWS cloud and security teams to ensure all workloads remain compliant with governing policies. Actively identifying and resolving security findings and misconfigurations across the AWS environment.

---

## Project 3 — InsightGen (GCP + AWS)

**GitHub Enterprise Administration**

Administering the entire GitHub Enterprise organisation for InsightGen — managing all code repositories, team structures, and member access across the organisation. Configuring and enforcing branch protection rules across all repositories to maintain code quality and prevent unauthorised changes.

Enforcing repository naming conventions, hygiene standards, and archival policies across the organisation. Managing GitHub Actions workflows and integrating them with the broader CI/CD platform. Coordinating with the global GIT governance and security teams to understand organisation-wide policies and implementing them correctly within the GitHub Enterprise environment.

Proactively monitoring for secrets accidentally committed to repositories and executing full remediation — including secret rotation, commit history cleanup, and enforcing secret scanning policies.

**GCP — Serverless Infrastructure & AI**

Designed and built the entire serverless deployment infrastructure on GCP from scratch — Cloud Run for containerised API and backend services, Cloud Run Functions for event-driven and lightweight function workloads. Configured **Serverless Network Endpoint Groups (NEGs)** and **GCP Load Balancers** to route traffic reliably to Cloud Run services — handling path-based routing, SSL termination, and traffic distribution across the environment.

Worked with **Vertex AI** — managing AI model deployments and supporting the team in consuming Vertex AI services within the InsightGen platform.

Implemented automated CI/CD deployment pipelines for all APIs and frontend applications — covering build, test, containerisation, and deployment to the correct Cloud Run service and environment. Maintained fully separate Dev and Int environments on GCP — each with independent configurations to prevent cross-environment impact. Collaborated with GCP cloud governance, networking, and the global security teams to implement platform-level policies correctly within the GCP project structure.

Continuously monitored GCP billing and resource usage — identified over-provisioned Cloud Run configurations, idle services, and inefficient resource allocations. **Reduced daily GCP spend from $14/day to $3/day, saving over $330/month.**

**AWS — AI Workloads, CI/CD & Infrastructure**

Built and maintained full CI/CD pipelines on AWS using **CodePipeline, CodeBuild, and CodeDeploy** — covering automated build, test, containerisation, and deployment workflows for all InsightGen applications on AWS.

Managed deployments across multiple compute targets:

- **Lambda-based deployments** — packaging and deploying serverless functions as part of the InsightGen application stack.
- **EC2 container deployments** — deploying and managing containerised workloads on EC2 instances.
- **Amazon ECR** — managing container image repositories, image lifecycle policies, and integrating ECR into all deployment pipelines.

Deployed the **AIForce application** into InsightGen environments, configured its foundational services, and integrated them into the existing infrastructure and workflows. Managing **AWS Bedrock** model access and ensuring AI workload infrastructure is reliable and cost-controlled.

Built **CloudWatch dashboards** to visualise Bedrock token usage across all models in use — giving the team real-time visibility into AI consumption patterns and helping forecast costs. Configured **CloudWatch metric alarms** on token consumption to detect and alert on unusual spikes immediately — preventing runaway AI costs.

Managing **Kong API Gateway** and **Keycloak** across InsightGen AWS environments — maintaining API routes, plugins, authentication flows, realms, OIDC clients, role mappings, and access policies for all applications and services.

Implemented **encryption in transit and at rest** across all databases — configured TLS connections, enabled storage encryption, and validated that no unencrypted data pathways exist across the environment.

Replicated the entire InsightGen application stack from GCP onto AWS — maintaining both cloud environments in parallel with consistent deployment standards, configuration patterns, and operational practices. Maintained Dev and Int environments on AWS aligned with the GCP environment structure.

**Continuously monitored AWS billing — reduced monthly spend from $1,500/month to $500/month, saving $1,000/month.**

**AI & Emerging Technology**

Deployed the AIForce application and worked on its foundational services — integrating them into the InsightGen infrastructure so the team could consume AI capabilities reliably across environments.

Supported a separate team developing an **OpenAI RAG (Retrieval-Augmented Generation) POC** — contributed to the technical implementation and helped resolve integration challenges between OpenAI APIs, vector storage, and document retrieval pipelines. Independently configured and maintained Keycloak for the OpenAI team's application — setting up the realm, OIDC clients, and access policies required by the team.

---

## Cross-Team Support

**Alphagen Team**

Supporting the Alphagen team as a standalone project — acting as their dedicated DevOps and Cloud point of contact for infrastructure and access needs. Managing all cloud access and permissions, supporting infrastructure setup, advising on architecture decisions, and helping the team navigate cloud governance requirements and organisational policies.

Onboarded the Alphagen team into the GitHub Enterprise organisation — created their repositories, configured team structures, assigned appropriate access levels, set up branch protection rules. Deployed Alphagen team applications onto AWS EC2 and continue maintaining their repositories within the GitHub Enterprise organisation on an ongoing basis.

**Thought Machine Team**

Supporting the Thought Machine team as a standalone project — acting as their dedicated DevOps and Cloud point of contact for infrastructure and access needs. Managing all cloud access and permissions, supporting infrastructure setup, advising on architecture decisions, and helping the team navigate cloud governance requirements and organisational policies.

---

## Standards Upheld Across All Projects

- Least-privilege IAM enforced across Azure and AWS — no over-permissioned roles or service accounts.
- Proactively drove cost optimisation across all three cloud providers — reducing GCP spend by over $330/month, AWS spend by $1,000/month, and delivering $320 in Azure savings.
- Active identification and remediation of security misconfigurations across Azure, AWS, and GCP.
- Ongoing coordination with cloud, networking, security, and governance teams across all three cloud providers.

---
