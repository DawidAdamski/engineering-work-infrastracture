# Infrastructure for Engineering Thesis  
### *“RFM Analysis Using Artificial Intelligence in a CRM Application Deployed on the Kubernetes Platform”*

This repository contains **only the infrastructure configuration and deployment manifests** used to run the engineering thesis project:

> **RFM Analysis Using Artificial Intelligence in a CRM Application Deployed on the Kubernetes Platform**

The infrastructure is designed to provide a reproducible, cloud-native environment for hosting the CRM system, databases, and AI components used in the analysis.

---

## 📘 Related Application Repository
The actual CRM application (FastAPI backend, data models, and UI) is maintained separately here:  
👉 **[engineering-work-crm](https://github.com/DawidAdamski/engineering-work-crm)**

That repository handles:
- customer and order data models,  
- REST API for classical RFM queries,  
- AI-based similarity endpoints (Qdrant integration),  
- and front-end communication.

This current repo focuses exclusively on **infrastructure provisioning and orchestration**.

---

## 🧩 Repository Structure

|Path|Description|
|:--|:--|
|`k8s/`|All Kubernetes YAML manifests for databases, services, and supporting components|
|`argocd/`|ArgoCD Application definitions for GitOps-based deployment management|
|`docs/`|Documentation for each infrastructure object (`README.architecture.md`, `README.postgresql.md`, `README.argocd.md`, etc.)|
|`worker/`|n8n embedding workflow and worker configuration|
|`testing/`|Test scripts and validation tools|


---

## ☁️ Infrastructure Components

|Component|Purpose|
|:--|:--|
|**ArgoCD**|GitOps continuous delivery tool managing all application deployments and configurations declaratively|
|**PostgreSQL**|Stores CRM transactional data and classical RFM tables|
|**Qdrant**|Vector database used for storing customer embeddings|
|**n8n (Embedding Worker)**|Automates creation and synchronization of AI embeddings|
|**CRM API**|Backend service providing REST endpoints for classical RFM and AI-based segmentation|
|**Namespace `crm-rfm`**|Isolated environment for application services (PostgreSQL, Qdrant, n8n, CRM API)|
|**Namespace `argocd`**|Isolated environment for ArgoCD deployment management|
|**ConfigMaps / Secrets**|Configuration and credential management|
|**(optional) Ingress**|Expose the CRM API to external users (e.g., via Minikube host)|

---

## 🐳 Required Docker Images

All container images are locked to specific versions to ensure reproducible deployments. **Do not use `latest` tags** (except for UBI10 base image where specified).  
**Primary repository: [Docker Hub](https://hub.docker.com/)** — most images are pulled from Docker Hub.

| Component | Image (Docker Hub) | Version | Purpose |
|:--|:--|:--|:--|
| **PostgreSQL** | [`postgres`](https://hub.docker.com/_/postgres) | `18-alpine` | PostgreSQL 18 database for storing CRM transactional data |
| **Qdrant** | [`qdrant/qdrant`](https://hub.docker.com/r/qdrant/qdrant) | `v1.15.5` | Vector database for embeddings and similarity search |
| **n8n** | [`n8nio/n8n`](https://hub.docker.com/r/n8nio/n8n) | `1.118.1` | Workflow automation platform for embedding generation |
| **ArgoCD Server** | `quay.io/argoproj/argocd` | `v3.1.9` | GitOps continuous delivery server *(hosted on Quay.io)* |
| **ArgoCD Repo Server** | `quay.io/argoproj/argocd-repo-server` | `v3.1.9` | Git repository synchronization *(hosted on Quay.io)* |
| **ArgoCD Application Controller** | `quay.io/argoproj/argocd-applicationset-controller` | `v3.1.9` | Application controller for ArgoCD *(hosted on Quay.io)* |
| **ArgoCD Redis** | `quay.io/argoproj/argocd` | `v3.1.9` (redis-ha) | Redis cache for ArgoCD *(hosted on Quay.io)* |
| **CRM API Base Image** | `registry.access.redhat.com/ubi10/python-311` | `latest` | Base image for S2I builds *(Red Hat Container Registry)* |
| **CRM API** | `crm-api` | Custom (S2I-built) | Custom application image built from source |

**Notes:** 
- **Docker Hub** is the primary repository for application images (PostgreSQL, Qdrant, n8n).
- PostgreSQL uses the official `postgres` image version 18 — plain PostgreSQL without vector extensions.
- Version tags are locked to ensure consistency across deployments.
- Update versions only after testing in a development environment.
- For ArgoCD, all components use the same version (`v3.1.9`) and are hosted on Quay.io.
- CRM API image is built using Source-to-Image (S2I) from the Red Hat UBI10 base image listed above.
- Verify versions on [Docker Hub](https://hub.docker.com/) before updating.

---

## 🧱 Deployment Target

The default target environment is **Minikube** for local testing and demonstration purposes.  
All manifests, services, and internal DNS references are compatible with:
- **Minikube**,  
- **K3s**, or  
- any standard **Kubernetes 1.28+ cluster**.

---

## ⚙️ Usage Overview

```bash
# Start local cluster
minikube start

# Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Create application namespace
kubectl create namespace crm-rfm

# Deploy applications via ArgoCD (after configuring Application definitions)
# See docs/README.argocd.md for detailed ArgoCD setup instructions

# Check running pods
kubectl get pods -n crm-rfm
kubectl get pods -n argocd

# (Optional) Access ArgoCD UI
kubectl port-forward svc/argocd-server 8080:443 -n argocd
# Open https://localhost:8080 in your browser

# (Optional) Expose services for local access
kubectl port-forward svc/qdrant 6333:6333 -n crm-rfm
kubectl port-forward svc/postgres 5432:5432 -n crm-rfm
kubectl port-forward svc/crm-api 8000:8000 -n crm-rfm
```

### Detailed Documentation

For comprehensive deployment instructions, see:
- **[Deployment Guide](docs/README.deployment.md)** – Step-by-step deployment instructions
- **[ArgoCD Documentation](docs/README.argocd.md)** – GitOps deployment management
- **[Architecture Overview](docs/README.architecture.md)** – System architecture and components
