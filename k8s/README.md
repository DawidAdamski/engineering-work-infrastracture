# Kubernetes Manifests

This directory contains all Kubernetes manifests for the CRM-RFM infrastructure.

## File Structure

| File | Description |
|:--|:--|
| `namespace.yaml` | Namespace `crm-rfm` for all application components |
| `configmaps.yaml` | Configuration maps for CRM API, PostgreSQL initialization |
| `secrets.yaml` | Secrets for PostgreSQL and CRM (credentials, API keys) |
| `postgres.yaml` | PostgreSQL StatefulSet and Service |
| `qdrant.yaml` | Qdrant StatefulSet and Service |
| `n8n.yaml` | n8n Deployment and Service (embedding worker) |
| `crm-api.yaml` | CRM API Deployment and Service |
| `ingress.yaml` | Ingress for external access to CRM API |
| `psql-utility.yaml` | Utility Pod for PostgreSQL access (optional) |

## Deployment Order

When deploying manually (not via ArgoCD), apply manifests in this order:

1. `namespace.yaml`
2. `configmaps.yaml`
3. `secrets.yaml`
4. `postgres.yaml`
5. `qdrant.yaml`
6. `n8n.yaml`
7. `crm-api.yaml`
8. `ingress.yaml`
9. `psql-utility.yaml` (optional)

## ArgoCD Deployment

When using ArgoCD, apply the Application manifest in `../argocd/applications/crm-infrastructure-app.yaml` which will automatically deploy all manifests in this directory in the correct order.

## Image Versions

- **PostgreSQL**: `postgres:18-alpine`
- **Qdrant**: `qdrant/qdrant:v1.15.5`
- **n8n**: `n8nio/n8n:1.118.1`
- **CRM API**: `crm-api:latest` (custom S2I-built image)

All image versions are locked as per `.cursor/rules/images.mdc`.

