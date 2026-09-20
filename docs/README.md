# EDOP Documentation

This folder contains the formal design documents and architecture diagrams for the Enterprise Decision Orchestration Platform.

## Design Documents

- **EDOP_High_Level_Design_HLD.docx** – High Level Design (conceptual architecture, problems, GCP solution overview, decisions, tech stack).
- **EDOP_Low_Level_Design_LLD.docx** – Low Level Design focused on GCP deployment in project `intelligentmachines`, region `us-central1`, using Google ADK 2.0 and native GCP services.

## Architecture Diagrams

- **architecture/conceptual-architecture.jpg** – Conceptual layered architecture (Humans → UI → Master Orchestrator → Connectors → Task Agents → Data & Governance).
- **architecture/gcp-solution-architecture.jpg** – Detailed GCP solution architecture (GKE, Vertex AI, Pub/Sub, data services, security, observability).

Please refer to the HLD for design rationale and the LLD for implementation and deployment details.
