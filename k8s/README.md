# Kubernetes Manifests

This directory contains all Kubernetes manifests for the CRM-RFM infrastructure.

## File Structure

| File | Description |
|:--|:--|
| `namespace.yaml` | Namespace `crm-rfm` for all application components |
| `configmaps.yaml` | Configuration maps for CRM API, PostgreSQL initialization |
| `secrets.yaml` | Secrets for PostgreSQL and CRM (credentials, API keys) |
| `postgres.yaml` | PostgreSQL StatefulSet and Service (NodePort) |
| `qdrant.yaml` | Qdrant StatefulSet and Service (NodePort) |
| `n8n.yaml` | n8n Deployment and Service (NodePort) |
| `crm-api.yaml` | CRM API Deployment and Service (NodePort) |
| `ingress.yaml` | Ingress for external access to CRM API |
| `psql-utility.yaml` | Utility Pod for PostgreSQL access (optional) |
| `metallb-config.yaml` | MetalLB IP pool configuration for LoadBalancer services |
| `services-loadbalancer.yaml` | LoadBalancer service definitions (for use with MetalLB) |
| `nginx-ingress-controller.yaml` | Manual nginx-ingress controller installation (not via addon) |
| `haproxy-ingress-controller.yaml` | HAProxy Ingress controller installation (high-performance alternative) |

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

## Service Access Options

Services are configured as **NodePort** by default for direct access without port-forward.  
For production-like setups, you can use:

- **MetalLB LoadBalancer** - Install MetalLB and use `services-loadbalancer.yaml` (see `docs/README.service-access.md`)
- **Ingress with nginx** - Install nginx-ingress controller and use `ingress.yaml` (see `docs/README.service-access.md`)
- **Ingress with HAProxy** - Install HAProxy Ingress controller for high-performance load balancing (see `docs/README.service-access.md`)

See [Service Access Guide](../docs/README.service-access.md) for detailed instructions on all access methods.

