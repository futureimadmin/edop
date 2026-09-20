**HIGH LEVEL DESIGN (HLD)**

**Enterprise Decision Orchestration Platform (EDOP)**

*Agentic AI Orchestration for Cross-Functional Decision Making*

Document Version: 1.1 | Date: September 20, 2026 | Status: Draft

## 1. Introduction

This High Level Design (HLD) document defines the architecture, components, and design decisions for the Enterprise Decision Orchestration Platform (EDOP). EDOP is an agentic AI system that enables organizations to coordinate complex, cross-functional decisions across organizational silos while preserving human judgment, authority, and accountability.

The platform is inspired by field research on human-AI collaboration at leading enterprises (Walmart, Amazon, Ericsson, Ramp, Medtronic and others) and addresses the structural challenge of applying AI beyond narrow tasks to enterprise-wide decision orchestration. It combines a Master AI Orchestrator, deterministic Connector layer, specialized task-based agents, and a robust Governance Control Plane.

Target deployment environment: Google Cloud Platform (GCP), Project ID: intelligentmachines, Primary Region: us-central1. Orchestration is built on Google Agent Development Kit (ADK) 2.0.

## 2. Conceptual Architecture

The conceptual architecture of EDOP consists of five core layers plus human and external system interfaces. It is designed so that authority and final decisions remain with humans while the system automates the orchestration of intelligence.

### 2.1 Architecture Layers

1. **Human Decision Makers Layer**: Executives, functional managers (Procurement, Logistics, Finance, Merchandising, Legal), and Governance/Risk/Compliance stakeholders who initiate decisions, supply tacit knowledge, and approve final outcomes.
2. **User Interface Layer**: Master Orchestrator Chat + Decision Workspace, Functional Expert Workspaces, Executive Dashboard & Scenario Explorer, and Governance Console.
3. **Master AI Orchestrator**: Conversational planner built with Google ADK 2.0. Clarifies objectives and constraints, decomposes decisions into modules, routes work, elicits human-only knowledge, and manages the overall decision lifecycle.
4. **Connector Layer (Deterministic)**: Rule-based connectors that perform guardrail validation, output verification, trigger next steps or parallel branches, propagate human knowledge and results, and enforce policies.
5. **Task-Based Specialized Agents**: Domain agents (Procurement, Demand/Merchandising, Logistics/Capacity, Finance/Working Capital & Risk, Legal/Compliance, and others) implemented as ADK agents or workflow nodes.
6. **Perception, State & Knowledge Layer**: Decision state store, vector knowledge store, immutable audit & lineage log, and event bus.
7. **Governance & Control Plane**: Agent registry & inventory, policy & guardrail engine, identity & access, observability (cost/quality/risk), and evaluation & red-teaming.

### 2.2 Conceptual Flow (Tariff Example)

1. Executive initiates a decision (new tariff → explore inventory pull-forward) via the Master Orchestrator UI.
2. Master Orchestrator (ADK 2.0 workflow) clarifies goals/constraints and decomposes the problem into parallel task modules.
3. Connectors launch specialized task agents and enforce guardrails.
4. Task agents produce outputs; humans are prompted only when they hold unique information the system lacks.
5. New human knowledge is immediately propagated through connectors to all relevant agents and re-runs.
6. Senior leaders receive an integrated trade-off view and make the final decision. Full lineage is retained for audit.

### 2.3 Key Design Principles

- Hybrid deterministic + agentic architecture (deterministic Connectors for control; ADK agents for reasoning).
- Humans intervene only when they possess unique context or constraints.
- Modular decision decomposition with standardized inputs, outputs, and assumptions.
- Parallel execution with continuous knowledge propagation.
- Full auditability and governance enforced at runtime via the Control Plane.

## 3. Problem Statements

EDOP addresses four core enterprise challenges:

- **Coordination Bottlenecks**: Cross-functional decisions (tariffs, demand shocks, supply disruptions) require asymmetric information that surfaces late and forces sequential resets.
- **Limitations of Task-Level AI**: Existing tools improve narrow tasks but do not systematically integrate human-only knowledge or reconcile conflicting functional outputs.
- **Unstructured Human-AI Collaboration**: People apply generic trust to AI rather than intervening only when they hold unique information.
- **Governance Gaps**: Without a control plane, multi-agent systems risk policy violations, opaque trails, and loss of human accountability.

## 4. Google Cloud High Level Solution

EDOP is deployed entirely on Google Cloud in project intelligentmachines (us-central1). The solution uses GKE for containerized ADK agent workloads, Vertex AI for Gemini models and evaluation, Cloud Pub/Sub + Eventarc for event-driven communication, Cloud SQL / AlloyDB and Firestore for state, Vertex AI Vector Search for knowledge, and native GCP operations and security services.

### 4.1 GCP Service Mapping

| EDOP Component | GCP Service(s) | Purpose |
|----------------|----------------|--------|
| Master Orchestrator & Task Agents | GKE + Vertex AI (Gemini) + Google ADK 2.0 | Stateful multi-agent workflows, LLM inference |
| Connector Layer | GKE + Cloud Run + Cloud Functions | Deterministic rule evaluation and routing |
| Event Bus | Cloud Pub/Sub + Eventarc | Reliable inter-agent messaging and triggers |
| Decision State & Checkpoints | Cloud SQL (PostgreSQL) / AlloyDB | Durable state and audit metadata |
| Vector Knowledge Store | Vertex AI Vector Search | Semantic retrieval of decisions & knowledge |
| Documents / Unstructured Data | Cloud Storage + Document AI | Contracts, reports, knowledge bases |
| Identity & Access | Cloud Identity + IAM + Workload Identity | SSO, human & agent identities |
| API Gateway | Apigee or API Gateway | Secure external and internal APIs |
| Observability | Cloud Monitoring, Logging, Trace, Error Reporting | Metrics, logs, distributed tracing, alerting |
| Governance Console | Cloud Run / GKE + Looker | Policy management, dashboards, audit views |
| Secrets & Config | Secret Manager + Config Connector | Credentials and configuration |
| CI/CD & GitOps | Cloud Build + Cloud Deploy + Artifact Registry | Build, test, deploy agent images |

### 4.2 Network & Security Posture

- Private GKE clusters with Workload Identity.
- VPC-native networking and Private Service Connect.
- Cloud Armor for edge protection; Binary Authorization for container images.
- Customer-managed encryption keys (CMEK) for sensitive data stores.
- VPC Service Controls for data exfiltration protection.

## 5. Architectural Decisions

### 5.1 Orchestration Framework
**Decision:** Google Agent Development Kit (ADK) 2.0 as the primary multi-agent orchestration framework.
**Rationale:** ADK 2.0 provides graph-based deterministic workflows, collaborative multi-agent architectures, built-in human-in-the-loop primitives, task delegation, state management, and native optimization for Gemini / Vertex AI. It is the Google-recommended code-first framework for production agent systems on GCP.

### 5.2 Hybrid Deterministic-Agentic Pattern
**Decision:** Use deterministic Connectors for control flow, validation, and policy enforcement; reserve ADK agents for reasoning-heavy steps. ADK 2.0 Workflow runtime supports this hybrid model natively.

### 5.3 Human-in-the-Loop Model
**Decision:** Structured intervention — humans supply missing information or adjust outputs only when they possess unique context. ADK 2.0 provides first-class HITL support.

### 5.4 Compute & Runtime
**Decision:** Primary runtime on GKE for long-running stateful ADK workflows; Cloud Run / Cloud Functions for lightweight connectors and webhooks.

### 5.5 Model Strategy
**Decision:** Vertex AI Gemini family (and partner models via Model Garden) for Master Orchestrator and complex reasoning; lighter models for narrow task agents. Multi-model routing for cost and latency control.

## 6. Tools, Languages, Frameworks Needed to Implement

### 6.1 Languages
- **Python 3.11+**: Primary language for ADK agents, data processing, and backend services.
- **TypeScript / JavaScript**: Frontend (Next.js / React) and selected Node services.
- **Go (optional)**: ADK Go 2.0 support for high-performance components if required.

### 6.2 Core Frameworks & Libraries
- **Google Agent Development Kit (ADK) 2.0**: Graph-based workflows, multi-agent collaboration, HITL, task API.
- **FastAPI + Pydantic**: API layer and validation.
- **Next.js + React + Tailwind**: Modern UI surfaces.
- **Vertex AI SDK / google-cloud libraries**: Model access, Vector Search, Document AI, etc.

### 6.3 GCP Native Services (No Third-Party Observability Stacks)
- Cloud Monitoring, Cloud Logging, Cloud Trace, Error Reporting for full observability.
- Cloud Build, Cloud Deploy, Artifact Registry for CI/CD.
- Secret Manager, Cloud KMS, Binary Authorization, VPC Service Controls.
- Terraform or Config Connector for Infrastructure as Code.
- Looker / Looker Studio for executive and governance dashboards.

## 7. Document Control

This HLD is a living document. Related document: Low Level Design (LLD) – EDOP GCP Deployment (intelligentmachines / us-central1). Major architectural changes require Architecture and Governance review.

— End of High Level Design Document —
