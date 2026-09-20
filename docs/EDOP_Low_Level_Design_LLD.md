**LOW LEVEL DESIGN (LLD)**

**Enterprise Decision Orchestration Platform (EDOP)**

*GCP Deployment Design – Project: intelligentmachines | Region: us-central1*

Document Version: 1.1 | Date: September 20, 2026 | Framework: Google ADK 2.0

## 1. Introduction

This Low Level Design (LLD) provides the detailed technical design to implement and deploy EDOP on Google Cloud Platform. It expands the High Level Design into concrete infrastructure, service configurations, networking, security, data stores, and deployment patterns using Google Agent Development Kit (ADK) 2.0 and exclusively GCP-native services.

Scope: Production-ready deployment of Master Orchestrator, Connector Layer, Task Agents, Governance Control Plane, and supporting data/AI services in project intelligentmachines, primary region us-central1.

## 2. Overview

Workloads run primarily as containers on GKE. State is durable. Communication is event-driven via Cloud Pub/Sub and Eventarc. Governance and observability use only GCP services (Cloud Monitoring, Logging, Trace, Error Reporting, Looker). No third-party observability stacks (e.g., Prometheus) are used.

### 2.1 Deployment Targets

- **GCP Project:** intelligentmachines
- **Primary Region:** us-central1
- **Orchestration Framework:** Google Agent Development Kit (ADK) 2.0
- **Environments:** dev → staging → prod (isolated GKE clusters or strongly isolated namespaces)

## 3. GCP Solution Architecture

The following describes the logical GCP solution architecture for EDOP.

### 3.1 High-Level GCP Topology

Users (Executives, Functional Managers, Governance) access the system via HTTPS Load Balancing + Cloud Armor + Identity-Aware Proxy / Apigee. Requests reach the UI services (Cloud Run or GKE) and the Master Orchestrator API.

The Master Orchestrator and Task Agents run as ADK 2.0 workflows inside GKE pods in private namespaces. They call Vertex AI (Gemini) for reasoning, read/write decision state from Cloud SQL / AlloyDB, publish/consume events via Cloud Pub/Sub, and retrieve semantic knowledge from Vertex AI Vector Search. Document AI and Cloud Storage handle unstructured content.

Deterministic Connectors run on Cloud Run / Cloud Functions (or GKE) and enforce policies via the Governance Control Plane. All agent and human actions are logged to Cloud Logging; metrics and traces flow to Cloud Monitoring and Cloud Trace. Looker provides executive and compliance dashboards.

Secrets are managed exclusively by Secret Manager. Workload Identity maps Kubernetes service accounts to GCP service accounts. Binary Authorization ensures only approved images run. VPC Service Controls and CMEK protect data.

### 3.2 Component-to-Service Mapping

| Logical Component | GCP Service(s) | Notes |
|-------------------|----------------|-------|
| Master Orchestrator (ADK 2.0) | GKE + Vertex AI Gemini | Graph workflows, coordinator agents |
| Task Agents (ADK) | GKE + Vertex AI | Specialized domain agents |
| Connectors (deterministic) | Cloud Run / Cloud Functions + GKE | Guardrails, routing, policy |
| Event Bus | Cloud Pub/Sub + Eventarc | decision.events, agent.outputs, human.knowledge |
| Decision State / Checkpoints | Cloud SQL (PostgreSQL) or AlloyDB | Private IP, CMEK, HA, PITR |
| Session / Real-time State | Firestore (Native) | UI collaboration state |
| Vector Knowledge | Vertex AI Vector Search | Private endpoint preferred |
| Documents & Artifacts | Cloud Storage + Document AI | Contracts, reports, model artifacts |
| Identity | Cloud Identity + IAM + Workload Identity | Humans + agent identities |
| API Edge | API Gateway / Apigee + Cloud Armor | External & internal APIs |
| Observability | Cloud Monitoring, Logging, Trace, Error Reporting | No external Prometheus/Grafana |
| Dashboards | Looker / Looker Studio | Executive & governance views |
| Secrets | Secret Manager | Automatic rotation where available |
| CI/CD | Cloud Build + Cloud Deploy + Artifact Registry | GitOps-friendly |
| IaC | Terraform or Config Connector | VPC, GKE, IAM, data services |

## 4. Detailed Infrastructure Design

### 4.1 Networking
- VPC: edop-vpc (custom) in us-central1 with secondary ranges for GKE pods/services.
- Subnets: edop-gke-subnet, edop-data-subnet, edop-proxy-subnet.
- Private Google Access + Private Service Connect enabled.
- Cloud NAT for controlled egress.
- Least-privilege firewall rules; default deny.

### 4.2 GKE Cluster
- Cluster Name: edop-gke-us-central1
- Mode: GKE Autopilot (preferred) or Standard
- Release Channel: Regular
- Locations: us-central1-a / b / c
- Workload Identity: Enabled
- Binary Authorization: Enabled – only signed images from Artifact Registry
- Network Policy: Dataplane V2 / Calico
- Namespaces: edop-orchestrator, edop-agents, edop-connectors, edop-governance, edop-monitoring
- Autoscaling: HPA + VPA enabled

### 4.3 Data Services
- Cloud SQL (PostgreSQL) or AlloyDB: Primary store for decision state, ADK session/checkpoint data, agent registry, audit metadata. Private IP, CMEK, regional HA, point-in-time recovery.
- Firestore: Lightweight real-time session and collaboration state for UI.
- Vertex AI Vector Search: Semantic search over past decisions, tacit knowledge summaries, and contracts.
- Cloud Storage: Documents, evaluation datasets, model artifacts; CMEK and lifecycle policies.

### 4.4 Messaging
Cloud Pub/Sub topics (examples): decision.events, agent.outputs, human.knowledge, governance.alerts. Dead-letter topics and exponential backoff configured. Eventarc routes selected events to Cloud Run/Functions.

### 4.5 AI Platform
Vertex AI provides Gemini models (and Model Garden partners) for ADK agents. ADK 2.0 runs inside GKE and calls Vertex AI endpoints using Workload Identity. Vertex AI evaluation and pipelines support continuous agent quality assessment. Document AI extracts structured data from contracts and reports.

### 4.6 Security & Governance Controls
- IAM + Workload Identity for all human and agent identities.
- VPC Service Controls perimeter around sensitive resources.
- Binary Authorization attestors for container supply-chain security.
- Cloud Armor WAF/DDoS protection on external load balancers.
- CMEK for Cloud SQL, GCS, and selected Vertex resources.
- Cloud Audit Logs (Admin, Data Access, System) exported for long-term retention and analysis.

### 4.7 Observability (GCP Native Only)
- Cloud Monitoring – metrics, uptime checks, alerting policies, custom dashboards.
- Cloud Logging – structured logs with decision-id correlation.
- Cloud Trace – distributed tracing of ADK workflows and agent calls.
- Error Reporting – automatic error aggregation.
- Looker / Looker Studio – executive KPIs, cost attribution, intervention rates, compliance views.

### 4.8 CI/CD
- Source control → Cloud Build (build & test) → Artifact Registry.
- Cloud Deploy (or Config Connector + GitOps) promotes to staging then prod with approval gates.
- Binary Authorization policies enforced at deploy time.

## 5. Architectural Decisions

**AD-01: Google ADK 2.0 as Orchestration Runtime**  
ADK 2.0 supplies graph-based workflows, collaborative multi-agent patterns, first-class human-in-the-loop, task delegation, and native Vertex AI / Gemini integration. It is the preferred Google framework for production agent systems on GCP.

**AD-02: GKE for Stateful ADK Workloads**  
Long-running, stateful ADK graphs with checkpointing and HITL pauses are best hosted on GKE rather than pure serverless.

**AD-03: Workload Identity Only**  
No long-lived service-account keys. All agent-to-GCP authentication uses Workload Identity.

**AD-04: GCP-Native Observability Only**  
Cloud Monitoring, Logging, Trace, Error Reporting, and Looker replace any third-party monitoring stacks for consistency, security, and reduced operational surface.

**AD-05: Hybrid Pub/Sub + Synchronous Calls**  
Pub/Sub for asynchronous knowledge propagation and fan-out; synchronous calls inside a single ADK workflow when low latency is required.

## 6. Tools, Languages, Frameworks

### 6.1 Languages
- Python 3.11+ (ADK agents, FastAPI services)
- TypeScript / Node (Next.js frontend)
- Go (optional ADK Go 2.0 components)

### 6.2 Core Frameworks
- Google ADK 2.0: Workflow runtime, multi-agent collaboration, HITL, Task API.
- FastAPI + Pydantic, Next.js + React + Tailwind.
- Vertex AI Python / Node SDKs, google-cloud client libraries.

### 6.3 GCP Tooling
- Terraform or Config Connector, Cloud Build, Cloud Deploy, Artifact Registry.
- gcloud, kubectl, Skaffold for local development loops.
- Cloud Monitoring, Logging, Trace, Error Reporting, Looker – no external Prometheus/Grafana.

## 7. Next Steps

1. Finalize detailed ADK 2.0 workflow graphs and connector rule definitions.
2. Author Terraform modules for VPC, GKE, Cloud SQL/AlloyDB, Pub/Sub, IAM, and Vertex resources.
3. Establish evaluation harness using Vertex AI evaluation capabilities.
4. Complete threat model and security review.
5. Pilot one high-value decision module (e.g., tariff response) in the staging environment.

— End of Low Level Design Document —
