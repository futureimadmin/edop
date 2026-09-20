# EDOP Runbook

**Enterprise Decision Orchestration Platform**  
Project: `intelligentmachines` | Region: `us-central1` | Framework: **Google ADK 2.0**

This README is the operational runbook for setting up, deploying, and operating EDOP on Google Cloud Platform.

---

## 1. Prerequisites

- Google Cloud account with billing enabled
- `gcloud` CLI (latest) authenticated
- `kubectl`, `terraform` (≥ 1.5), Docker, Python 3.11+, Node 20+
- Access to project `intelligentmachines` (or create it)
- GitHub access to this repository

```bash
gcloud auth login
gcloud config set project intelligentmachines
gcloud config set compute/region us-central1
```

---

## 2. High-Level Setup Sequence

1. Enable required APIs
2. Create networking (VPC, subnets, Private Google Access)
3. Provision data services (Cloud SQL / AlloyDB, Firestore, GCS, Vector Search)
4. Create GKE cluster with Workload Identity + Binary Authorization
5. Configure IAM & Workload Identity bindings
6. Deploy ADK 2.0 agent workloads + Connectors
7. Configure Pub/Sub topics, Eventarc triggers, Secret Manager
8. Set up observability (Monitoring, Logging, Trace, Looker)
9. Deploy UI (Next.js) and Governance Console
10. Run smoke tests and first decision pilot

---

## 3. Enable GCP APIs

```bash
gcloud services enable \
  container.googleapis.com \
  compute.googleapis.com \
  sqladmin.googleapis.com \
  firestore.googleapis.com \
  aiplatform.googleapis.com \
  pubsub.googleapis.com \
  eventarc.googleapis.com \
  secretmanager.googleapis.com \
  cloudfunctions.googleapis.com \
  run.googleapis.com \
  artifactregistry.googleapis.com \
  cloudbuild.googleapis.com \
  clouddeploy.googleapis.com \
  monitoring.googleapis.com \
  logging.googleapis.com \
  cloudtrace.googleapis.com \
  documentai.googleapis.com \
  servicenetworking.googleapis.com \
  networkmanagement.googleapis.com
```

---

## 4. Networking (Terraform recommended)

Create VPC `edop-vpc` in `us-central1` with:

- Subnet for GKE (`edop-gke-subnet`)
- Subnet for data services
- Secondary ranges for pods and services
- Private Google Access enabled
- Cloud NAT for egress
- Firewall rules (default deny + explicit allows)

Apply:

```bash
cd infra/terraform
terraform init
terraform plan -var="project_id=intelligentmachines" -var="region=us-central1"
terraform apply
```

---

## 5. GKE Cluster

```bash
gcloud container clusters create-auto edop-gke-us-central1 \
  --region=us-central1 \
  --release-channel=regular \
  --enable-private-nodes \
  --network=edop-vpc \
  --subnetwork=edop-gke-subnet \
  --workload-pool=intelligentmachines.svc.id.goog \
  --enable-binauthz
```

Get credentials:

```bash
gcloud container clusters get-credentials edop-gke-us-central1 --region=us-central1
```

Create namespaces:

```bash
kubectl create namespace edop-orchestrator
kubectl create namespace edop-agents
kubectl create namespace edop-connectors
kubectl create namespace edop-governance
kubectl create namespace edop-monitoring
```

---

## 6. Database Connections

### Cloud SQL / AlloyDB (Decision State & ADK Checkpoints)

1. Create instance (private IP, CMEK recommended):

```bash
gcloud sql instances create edop-state-db \
  --database-version=POSTGRES_15 \
  --tier=db-custom-2-8192 \
  --region=us-central1 \
  --network=projects/intelligentmachines/global/networks/edop-vpc \
  --no-assign-ip
```

2. Create database and user:

```bash
gcloud sql databases create edop_decision --instance=edop-state-db
gcloud sql users create edop_app --instance=edop-state-db --password=<strong-password>
```

3. Store connection string in Secret Manager:

```bash
echo -n "postgresql://edop_app:<password>@<private-ip>:5432/edop_decision" | \
  gcloud secrets create edop-db-connection --data-file=-
```

4. In ADK / FastAPI services, inject via Workload Identity + Secret Manager.

Example (Python):

```python
from google.cloud import secretmanager

def get_db_url():
    client = secretmanager.SecretManagerServiceClient()
    name = "projects/intelligentmachines/secrets/edop-db-connection/versions/latest"
    response = client.access_secret_version(request={"name": name})
    return response.payload.data.decode("UTF-8")
```

### Firestore

```bash
gcloud firestore databases create --location=us-central1 --type=firestore-native
```

### Vertex AI Vector Search

Create index and endpoint via Vertex AI console or SDK. Grant the GKE service account the necessary Vertex AI roles.

---

## 7. Pub/Sub Topics & Eventarc

```bash
gcloud pubsub topics create decision.events
gcloud pubsub topics create agent.outputs
gcloud pubsub topics create human.knowledge
gcloud pubsub topics create governance.alerts

gcloud pubsub subscriptions create agent-outputs-sub --topic=agent.outputs
```

---

## 8. Deploying ADK 2.0 Agents

1. Install ADK:

```bash
pip install google-adk
```

2. Build & push images to Artifact Registry:

```bash
gcloud artifacts repositories create edop-agents --repository-format=docker --location=us-central1
docker build -t us-central1-docker.pkg.dev/intelligentmachines/edop-agents/master-orchestrator:latest .
docker push us-central1-docker.pkg.dev/intelligentmachines/edop-agents/master-orchestrator:latest
```

3. Deploy to GKE with Workload Identity, secret mounts, resource limits, and HPA.

---

## 9. Observability

- Use Cloud Monitoring, Logging, Trace, Error Reporting (native).
- Instrument ADK / FastAPI with OpenTelemetry → Cloud Trace / Monitoring.
- Dashboards in Looker or Cloud Monitoring for decision latency, intervention rate, agent success, cost.

---

## 10. Security Checklist

- [ ] Workload Identity enabled and bound for all agent service accounts
- [ ] Binary Authorization policy active
- [ ] CMEK on Cloud SQL, GCS, and sensitive Vertex resources
- [ ] VPC Service Controls perimeter defined
- [ ] Cloud Armor on external load balancers
- [ ] No long-lived service account keys
- [ ] Least-privilege IAM roles

---

## 11. Useful Commands

```bash
# Cluster status
kubectl get pods -A -l app=edop

# Logs
gcloud logging read 'resource.type="k8s_container" AND resource.labels.namespace_name="edop-orchestrator"' --limit=50

# Secrets
gcloud secrets list
gcloud secrets versions access latest --secret=edop-db-connection

# Pub/Sub
gcloud pubsub topics list
```

---

## 12. Documentation

| Document | Location |
|----------|----------|
| High Level Design (HLD) | [docs/EDOP_High_Level_Design_HLD.md](docs/EDOP_High_Level_Design_HLD.md) (Word version with diagram available in design package) |
| Low Level Design (LLD) | [docs/EDOP_Low_Level_Design_LLD.md](docs/EDOP_Low_Level_Design_LLD.md) (Word version with GCP diagram available in design package) |
| Conceptual Architecture | Embedded in HLD |
| GCP Solution Architecture | Embedded in LLD |

---

## 13. Support & Ownership

- Architecture & Governance: Review any change to modules, guardrails, or agent boundaries.
- Platform / SRE: Own GKE, networking, data services, CI/CD.
- Domain teams: Own their specialized task agents and human knowledge capture points.

---

*EDOP Runbook – Project intelligentmachines – September 2026*
