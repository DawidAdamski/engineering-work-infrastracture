# ArgoCD Applications

This directory contains ArgoCD Application manifests for GitOps-based deployment.

## Applications

| File | Description | Network | Branch |
|:--|:--|:--|:--|
| `crm-infrastructure-app.yaml` | Default ArgoCD Application for network `192.168.0.x` | `192.168.0.x` | `main` (HEAD) |
| `crm-infrastructure-app-192.168.10.x.yaml` | ArgoCD Application for network `192.168.10.x` | `192.168.10.x` | `network-192.168.10.x` |

## Network Configuration

This repository supports different network configurations:

- **Default (192.168.0.x)**: For most users with standard network setup
- **192.168.10.x**: For users with specific network requirements (e.g., Windows Hyper-V with custom subnet)

Each network configuration uses a different Git branch:
- `main` branch: Contains MetalLB config with IP range `192.168.0.200-192.168.0.210`
- `network-192.168.10.x` branch: Contains MetalLB config with IP range `192.168.10.200-192.168.10.210`

## Deployment

### Using Default Configuration (192.168.0.x)

```bash
# Apply the default ArgoCD Application
kubectl apply -f argocd/applications/crm-infrastructure-app.yaml

# Or use ArgoCD CLI
argocd app create -f argocd/applications/crm-infrastructure-app.yaml
```

### Using Network 192.168.10.x Configuration

```bash
# First, ensure the branch exists in Git
git checkout -b network-192.168.10.x
# (Make sure k8s/metallb-config.yaml has IP range 192.168.10.200-192.168.10.210)
git push -u origin network-192.168.10.x

# Apply the network-specific ArgoCD Application
kubectl apply -f argocd/applications/crm-infrastructure-app-192.168.10.x.yaml

# Or use ArgoCD CLI
argocd app create -f argocd/applications/crm-infrastructure-app-192.168.10.x.yaml
```

## Switching Between Configurations

To switch from one network configuration to another:

```bash
# Option 1: Delete current Application and apply new one
kubectl delete application crm-infrastructure -n argocd
kubectl apply -f argocd/applications/crm-infrastructure-app-192.168.10.x.yaml

# Option 2: Patch existing Application to change branch
kubectl patch application crm-infrastructure -n argocd --type merge -p '{"spec":{"source":{"targetRevision":"network-192.168.10.x"}}}'

# To switch back to default:
kubectl patch application crm-infrastructure -n argocd --type merge -p '{"spec":{"source":{"targetRevision":"HEAD"}}}'
```

## Application Details

The `crm-infrastructure` application:
- Monitors the `k8s/` directory in the Git repository
- Deploys all manifests to the `crm-rfm` namespace
- Uses automated sync with self-healing
- Creates the namespace automatically if it doesn't exist
- **Network-specific**: Points to different Git branches based on network configuration

## Important Notes

⚠️ **Only one Application can be active at a time** - both files define the same Application name (`crm-infrastructure`). Use only one configuration file at a time.

⚠️ **Branch must exist in Git** - Before using `crm-infrastructure-app-192.168.10.x.yaml`, ensure the `network-192.168.10.x` branch exists and contains the correct MetalLB configuration.

## Repository Configuration

Make sure to update the `repoURL` and `targetRevision` in the Application manifest to match your Git repository URL and branch.

