# Service Access Without Port-Forward

This document explains how to access services in the CRM infrastructure **without using port-forward** and **without Minikube addons**.

Two approaches are available:
1. **NodePort Services** - Direct access via Minikube node IP and assigned ports
2. **Ingress with Manual nginx-ingress** - HTTP/HTTPS routing via Ingress controller (installed manually, not via addon)

---

## Option 1: NodePort Services (Simplest)

All services are configured as **NodePort** type, exposing them directly on the Minikube node.

### Service Ports

| Service | Internal Port | NodePort | Access URL |
|:--|:--|:--|:--|
| **CRM API** | `8000` | `30080` | `http://$(minikube ip):30080` |
| **n8n** | `5678` | `30678` | `http://$(minikube ip):30678` |
| **Qdrant REST** | `6333` | `30333` | `http://$(minikube ip):30333` |
| **Qdrant gRPC** | `6334` | `30334` | `grpc://$(minikube ip):30334` |
| **PostgreSQL** | `5432` | `30432` | `postgresql://$(minikube ip):30432` |

### Accessing Services

1. **Get Minikube IP**:
   ```bash
   minikube ip
   ```

2. **Access Services**:
   ```bash
   # CRM API
   curl http://$(minikube ip):30080/health
   
   # n8n UI
   open http://$(minikube ip):30678
   
   # Qdrant REST API
   curl http://$(minikube ip):30333/collections
   
   # PostgreSQL (using psql)
   psql -h $(minikube ip) -p 30432 -U crmuser -d crmdb
   ```

### Advantages

- ✅ **No additional components** - Works immediately after service deployment
- ✅ **Simple and direct** - Access services directly via IP and port
- ✅ **No port-forward needed** - Services accessible from host machine
- ✅ **Works with any Kubernetes distribution** - Not Minikube-specific

### Disadvantages

- ⚠️ **Requires knowing Minikube IP** - Must query `minikube ip` or use fixed IP
- ⚠️ **Ports in 30000+ range** - Not standard HTTP ports (80, 443)
- ⚠️ **No domain names** - Must use IP addresses
- ⚠️ **All services exposed** - Less secure for production

---

## Option 2: Ingress with Manual nginx-ingress (Production-like)

Use **Ingress** resources with a manually installed nginx-ingress controller (not via Minikube addon).

### Installation

1. **Install nginx-ingress controller**:
   ```bash
   kubectl apply -f k8s/nginx-ingress-controller.yaml
   ```

2. **Wait for controller to be ready**:
   ```bash
   kubectl wait --namespace ingress-nginx \
     --for=condition=ready pod \
     --selector=app.kubernetes.io/component=controller \
     --timeout=300s
   ```

3. **Verify installation**:
   ```bash
   kubectl get pods -n ingress-nginx
   kubectl get svc ingress-nginx-controller -n ingress-nginx
   ```

### Accessing Services via Ingress

The ingress controller is exposed as a **NodePort** service:

| Service | NodePort | Access URL |
|:--|:--|:--|
| **Ingress HTTP** | `30000` | `http://crm.local` (requires `/etc/hosts` entry) |
| **Ingress HTTPS** | `30443` | `https://crm.local` (requires `/etc/hosts` entry) |

### Configure Local DNS

Add to `/etc/hosts` (Linux/macOS) or `C:\Windows\System32\drivers\etc\hosts` (Windows):

```
<minikube-ip>  crm.local
```

Replace `<minikube-ip>` with the output of `minikube ip`.

### Access Services

```bash
# CRM API via Ingress
curl http://crm.local/health

# Or directly via NodePort (no DNS needed)
curl http://$(minikube ip):30000/health
```

### Ingress Configuration

The existing `k8s/ingress.yaml` defines routing rules:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: crm-ingress
  namespace: crm-rfm
spec:
  ingressClassName: nginx
  rules:
    - host: crm.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: crm-api
                port:
                  number: 8000
```

### Advantages

- ✅ **Domain-based routing** - Use friendly hostnames (e.g., `crm.local`)
- ✅ **Standard HTTP/HTTPS ports** - Can configure to use ports 80/443
- ✅ **Path-based routing** - Route multiple services through single entry point
- ✅ **Production-ready** - Similar to production Kubernetes setups
- ✅ **SSL/TLS support** - Can configure certificates for HTTPS

### Disadvantages

- ⚠️ **Requires additional component** - Must install nginx-ingress controller
- ⚠️ **Requires DNS configuration** - Need to edit `/etc/hosts` or use DNS
- ⚠️ **More complex setup** - Additional deployment step

---

## Comparison

| Feature | NodePort | Ingress |
|:--|:--|:--|
| **Setup Complexity** | Simple | Moderate |
| **Additional Components** | None | nginx-ingress controller |
| **Access Method** | IP + Port | Domain name |
| **Port Range** | 30000-32767 | Configurable (default 30000) |
| **Production Ready** | Limited | Yes |
| **Path Routing** | No | Yes |
| **SSL/TLS** | No | Yes |

---

## Recommended Approach

### For Development/Testing

Use **NodePort** (Option 1) for simplicity:
- No additional setup required
- Immediate access after service deployment
- Easy to script and automate

### For Production-like Testing

Use **Ingress** (Option 2) for realistic testing:
- Mirrors production Kubernetes setups
- Better for testing domain-based routing
- Supports SSL/TLS configuration

---

## Troubleshooting

### NodePort Not Accessible

1. **Check Minikube is running**:
   ```bash
   minikube status
   ```

2. **Verify service is NodePort**:
   ```bash
   kubectl get svc -n crm-rfm
   ```

3. **Check firewall rules** (if applicable):
   ```bash
   # Linux
   sudo ufw status
   ```

4. **Test connectivity**:
   ```bash
   curl -v http://$(minikube ip):30080/health
   ```

### Ingress Not Working

1. **Verify nginx-ingress controller is running**:
   ```bash
   kubectl get pods -n ingress-nginx
   ```

2. **Check Ingress resource**:
   ```bash
   kubectl get ingress -n crm-rfm
   kubectl describe ingress crm-ingress -n crm-rfm
   ```

3. **Verify DNS entry**:
   ```bash
   # Should resolve to Minikube IP
   ping crm.local
   ```

4. **Check ingress controller logs**:
   ```bash
   kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller
   ```

---

## Examples

### Accessing CRM API

**NodePort**:
```bash
export MINIKUBE_IP=$(minikube ip)
curl http://$MINIKUBE_IP:30080/api/customers
```

**Ingress**:
```bash
# After adding to /etc/hosts
curl http://crm.local/api/customers
```

### Accessing n8n UI

**NodePort**:
```bash
open http://$(minikube ip):30678
```

**Ingress** (requires additional Ingress rule):
```yaml
# Add to k8s/ingress.yaml
- path: /n8n
  pathType: Prefix
  backend:
    service:
      name: n8n
      port:
        number: 5678
```

Then access: `http://crm.local/n8n`

### Accessing Qdrant

**NodePort**:
```bash
curl http://$(minikube ip):30333/collections
```

**Ingress** (requires additional Ingress rule for Qdrant service)

---

## Notes

- **Both methods can coexist** - Services remain accessible via NodePort even if Ingress is installed
- **Security considerations** - NodePort exposes services directly; Ingress provides additional routing and security features
- **Minikube tunnel** - Alternative to NodePort for LoadBalancer services (not covered here as it requires additional setup)
- **Service types** - Services are configured as NodePort; they can be changed to ClusterIP if only using Ingress

This approach provides flexible access to services without requiring port-forward or Minikube addons.
