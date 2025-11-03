# ArgoCD Applications

This directory contains ArgoCD Application manifests for GitOps-based deployment.

## Applications

| File | Description |
|:--|:--|
| `crm-infrastructure-app.yaml` | Main ArgoCD Application that manages all Kubernetes manifests in the `k8s/` directory |

## Deployment

To deploy via ArgoCD:

```bash
# Apply the ArgoCD Application
kubectl apply -f argocd/applications/crm-infrastructure-app.yaml

# Or use ArgoCD CLI
argocd app create -f argocd/applications/crm-infrastructure-app.yaml
```

## Application Details

The `crm-infrastructure` application:
- Monitors the `k8s/` directory in the Git repository
- Deploys all manifests to the `crm-rfm` namespace
- Uses automated sync with self-healing
- Creates the namespace automatically if it doesn't exist

## Repository Configuration

Make sure to update the `repoURL` and `targetRevision` in the Application manifest to match your Git repository URL and branch.

