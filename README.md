# Enterprise Decision Orchestration Platform (EDOP)

**Agentic AI Orchestration for Cross-Functional Decision Making**

EDOP is an enterprise platform that uses **Google Agent Development Kit (ADK) 2.0** to orchestrate AI agents across organizational silos. It enables faster, higher-quality cross-functional decisions while keeping final authority and accountability with humans.

**GCP Project:** `intelligentmachines`  
**Primary Region:** `us-central1`  
**Orchestration Framework:** Google ADK 2.0  
**Runtime:** GKE + Vertex AI (Gemini)

## Key Features

- **Master AI Orchestrator** – Conversational planner that decomposes enterprise decisions, elicits human-only (tacit) knowledge, and coordinates work.
- **Deterministic Connector Layer** – Guardrails, validation, routing, and policy enforcement.
- **Specialized Task Agents** – Domain agents for Procurement, Logistics, Finance, Merchandising, Legal, etc.
- **Human-in-the-Loop** – Humans intervene only when they hold unique context the AI cannot access.
- **Governance Control Plane** – Agent registry, policies, identity, full audit lineage, and observability (GCP-native).

## Architecture

### Conceptual Architecture
See `docs/architecture/conceptual-architecture.jpg`

### GCP Solution Architecture
See `docs/architecture/gcp-solution-architecture.jpg`

## Documentation

| Document | Description |
|----------|-------------|
| [High Level Design (HLD)](docs/EDOP_High_Level_Design_HLD.docx) | Conceptual architecture, problem statements, GCP high-level solution, architectural decisions, tools & frameworks |
| [Low Level Design (LLD)](docs/EDOP_Low_Level_Design_LLD.docx) | Detailed GCP deployment (GKE, Vertex AI, Pub/Sub, Cloud SQL/AlloyDB, security, observability) for project `intelligentmachines` |

## Tech Stack

- **Orchestration:** Google ADK 2.0 (graph workflows, multi-agent, HITL)
- **Compute:** GKE (Autopilot/Standard), Cloud Run, Cloud Functions
- **AI:** Vertex AI (Gemini family)
- **Data:** Cloud SQL / AlloyDB, Firestore, Vertex AI Vector Search, Cloud Storage, Document AI
- **Events:** Cloud Pub/Sub + Eventarc
- **Security:** Workload Identity, Binary Authorization, VPC Service Controls, CMEK, Cloud Armor, Secret Manager
- **Observability:** Cloud Monitoring, Cloud Logging, Cloud Trace, Error Reporting, Looker
- **CI/CD:** Cloud Build, Cloud Deploy, Artifact Registry
- **Frontend:** Next.js / React
- **Backend APIs:** FastAPI (Python)

## Getting Started

1. Review the HLD and LLD in the `docs/` folder.
2. Provision the GCP landing zone in project `intelligentmachines` (us-central1) following the LLD.
3. Deploy ADK 2.0 agent workloads to the GKE cluster `edop-gke-us-central1`.
4. Configure governance policies and human knowledge capture points.

## Repository Structure

```
edop/
├── README.md
├── docs/
│   ├── EDOP_High_Level_Design_HLD.docx
│   ├── EDOP_Low_Level_Design_LLD.docx
│   └── architecture/
│       ├── conceptual-architecture.jpg
│       └── gcp-solution-architecture.jpg
└── (future source code, IaC, agent definitions)
```

## License

Proprietary / Internal use – Intelligent Machines project.

---
*Generated as part of the EDOP design initiative – September 2026*
