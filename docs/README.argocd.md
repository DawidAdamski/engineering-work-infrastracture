# ArgoCD

ArgoCD is a **GitOps continuous delivery tool** used for managing and deploying all application components in the CRM infrastructure.  
It ensures declarative, version-controlled, and automated deployments aligned with the Git repository state.

---

## Purpose

- Manage all application deployments declaratively using Git as the single source of truth.
- Automatically synchronize Kubernetes resources with Git repository manifests.
- Provide visibility into application state and deployment status.
- Enable automated rollbacks and self-healing deployments.
- Simplify deployment management across multiple services (PostgreSQL, Qdrant, n8n, CRM API).

---

## Deployment Overview

| Item | Value |
|:--|:--|
| **Namespace** | `argocd` (system namespace, separate from `crm-rfm`) |
| **Service name** | `argocd-server` |
| **Port** | `443` (HTTPS) / `80` (HTTP) |
| **UI Access** | `http://localhost:8080` (via port-forward) or Ingress |
| **Git Repository** | This repository or dedicated GitOps repository |

ArgoCD runs as a **Deployment** in its own namespace and monitors Git repositories for changes.

Internal DNS for in-cluster access:  
`argocd-server.argocd.svc.cluster.local:443`

---

## GitOps Workflow

### Repository Structure

ArgoCD monitors the Git repository (this infrastructure repo) for Kubernetes manifests:

```
engineering-work-infrastracture/
├── k8s/
│   ├── postgres.yaml
│   ├── qdrant.yaml
│   ├── n8n.yaml
│   ├── crm-api.yaml
│   ├── configmaps.yaml
│   ├── secrets.yaml
│   └── ingress.yaml
└── argocd/
    └── applications/
        ├── postgres-app.yaml
        ├── qdrant-app.yaml
        ├── n8n-app.yaml
        └── crm-api-app.yaml
```

### Application Definitions

Each application component is defined as an ArgoCD `Application` resource:

**Example: PostgreSQL Application**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: postgres
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/DawidAdamski/engineering-work-infrastracture
    targetRevision: main
    path: k8s/postgres.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: crm-rfm
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

---

## Synchronization Policy

### Automated Sync

Applications can be configured for automatic synchronization:

- **Auto-sync**: ArgoCD automatically applies changes when Git repository is updated.
- **Self-heal**: Automatically reverts manual changes to match Git state.
- **Prune**: Removes resources that no longer exist in Git.

### Manual Sync

For critical deployments, manual sync allows for review and approval:

```bash
kubectl patch application postgres -n argocd \
  --type merge -p '{"spec":{"syncPolicy":{"automated":null}}}'
```

---

## Accessing the UI

There are several ways to access ArgoCD UI without port-forward:

### Method 1: LoadBalancer with MetalLB (Recommended)

If you have MetalLB installed, expose ArgoCD as a LoadBalancer service:

```bash
# Apply LoadBalancer service for ArgoCD
kubectl apply -f k8s/argocd-loadbalancer.yaml -n argocd

# Check the external IP
kubectl get svc argocd-server -n argocd

# Example output:
# NAME            TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)
# argocd-server   LoadBalancer   10.97.133.222   192.168.0.203   80:30080/TCP,443:30443/TCP
```

**Access ArgoCD:**
- **HTTPS:** `https://<EXTERNAL-IP>` (or `http://<EXTERNAL-IP>` if using HTTP)
- **Example:** `https://192.168.0.203`

**Note:** ArgoCD may require HTTPS. If you get certificate errors, you can:
- Accept the self-signed certificate in your browser
- Or access via HTTP if ArgoCD is configured to allow it

### Method 2: NodePort (Direct Access)

Change the ArgoCD server service to NodePort:

```bash
# Patch the service to NodePort
kubectl patch svc argocd-server -n argocd -p '{"spec":{"type":"NodePort"}}'

# Get the NodePort
kubectl get svc argocd-server -n argocd

# Access via Minikube IP
minikube ip
# Then access: https://$(minikube ip):<NODEPORT>
```

### Method 3: Port-Forward (Local Access)

For local development, use port-forward:

```bash
kubectl port-forward svc/argocd-server 8080:443 -n argocd
```

Access the UI at: `https://localhost:8080`

### Method 4: Ingress (Domain-based Access)

For domain-based routing, configure an Ingress resource:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/backend-protocol: "GRPC"
spec:
  ingressClassName: nginx
  rules:
    - host: argocd.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  number: 80
```

Then add to `/etc/hosts`:
```
<minikube-ip>  argocd.local
```

Access at: `http://argocd.local`

### Default Credentials

Default credentials (for initial setup):
- **Username:** `admin`
- **Password:** Retrieve with:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

**⚠️ Security Note:** Change the default password after first login in production environments!

---

## Application Management

### Creating Applications

Applications can be created via:

1. **ArgoCD UI** – Use the web interface to create applications.
2. **kubectl** – Apply Application manifests directly.
3. **ArgoCD CLI** – Use `argocd` command-line tool.

### CLI Example

```bash
argocd app create postgres \
  --repo https://github.com/DawidAdamski/engineering-work-infrastracture \
  --path k8s/postgres.yaml \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace crm-rfm \
  --sync-policy automated \
  --self-heal
```

### Monitoring Applications

Check application status:

```bash
argocd app list
argocd app get postgres
```

View application details in UI:
- **Health Status**: Healthy, Degraded, Progressing, Suspended, Unknown
- **Sync Status**: Synced, OutOfSync, Unknown
- **Resource Tree**: Visual representation of deployed resources

---

## Configuration Variables

| Variable | Purpose |
|:--|:--|
| `ARGOCD_SERVER_ADDR` | ArgoCD server address (`argocd-server.argocd.svc.cluster.local:443`) |
| `ARGOCD_INSECURE` | Skip TLS verification (for local development, default: `false`) |
| `ARGOCD_AUTH_TOKEN` | Authentication token for API/CLI access |
| `ARGOCD_GRPC_WEB` | Enable gRPC-Web for UI (default: `true`) |

---

## Integration with Other Components

| Component | Interaction |
|:--|:--|
| **PostgreSQL** | ArgoCD deploys and manages PostgreSQL StatefulSet, Service, and PVC. |
| **Qdrant** | ArgoCD deploys and manages Qdrant Deployment and Service. |
| **n8n** | ArgoCD deploys and manages n8n Deployment and Service. |
| **CRM API** | ArgoCD deploys and manages CRM API Deployment and Service. |
| **ConfigMaps/Secrets** | ArgoCD manages application configuration via Git-sourced manifests. |
| **Ingress** | ArgoCD can deploy Ingress resources for external access. |

---

## Deployment Flow with ArgoCD

1. **Install ArgoCD** in the cluster:
   ```bash
   kubectl create namespace argocd
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   ```

2. **Configure Application Definitions**:
   - Create Application manifests in `argocd/applications/`
   - Or configure via UI

3. **ArgoCD Syncs Applications**:
   - Monitors Git repository for changes
   - Detects changes in Kubernetes manifests
   - Automatically applies updates (if auto-sync enabled)

4. **Monitor Deployment Status**:
   - Check UI dashboard
   - Use CLI to verify sync status
   - Review logs for troubleshooting

---

## Best Practices

- **Separate Git Repository**: Consider using a dedicated GitOps repository for production.
- **Application Isolation**: Use ArgoCD projects to organize applications by team or environment.
- **Sync Windows**: Configure sync windows for production applications to prevent unexpected deployments.
- **Resource Hooks**: Use pre/post sync hooks for database migrations or backup operations.
- **RBAC**: Configure Role-Based Access Control to limit who can modify applications.
- **Health Checks**: Define custom health checks for application-specific health verification.

---

## Troubleshooting

| Issue | Possible Cause | Fix |
|:--|:--|:--|
| Application stuck in Progressing | Resource waiting for condition | Check resource events and logs |
| OutOfSync detected | Manual changes or Git update | Sync application via UI or CLI |
| Application health Unknown | Health check not configured | Add custom health check or fix resource |
| Sync fails | Invalid YAML or missing dependencies | Validate manifests, check ArgoCD logs |
| UI not accessible | Service or Ingress misconfigured | Verify Service and port-forward |

### Common Commands

```bash
# Get application status
argocd app get <app-name>

# Sync application manually
argocd app sync <app-name>

# Force refresh
argocd app get <app-name> --refresh

# View application events
argocd app history <app-name>
```

---

## Notes

- ArgoCD follows GitOps principles: **Git is the source of truth** for all deployments.
- All manual changes to managed resources will be reverted by ArgoCD (if self-heal is enabled).
- ArgoCD provides a unified view of all application deployments across the cluster.
- For production environments, enable authentication and RBAC for ArgoCD access.
- ArgoCD significantly simplifies deployment management compared to manual `kubectl apply` commands.

This GitOps approach ensures reproducible, traceable, and automated deployments for the thesis infrastructure.

