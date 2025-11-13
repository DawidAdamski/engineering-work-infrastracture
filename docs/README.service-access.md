# Service Access Without Port-Forward

This document explains how to access services in the CRM infrastructure **without using port-forward** and **without Minikube addons**.

Four approaches are available:
1. **NodePort Services** - Direct access via Minikube node IP and assigned ports
2. **LoadBalancer with MetalLB** - Production-like LoadBalancer services (recommended for production-like testing)
3. **Ingress with nginx-ingress** - HTTP/HTTPS routing via nginx Ingress controller (installed manually, not via addon)
4. **Ingress with HAProxy** - High-performance load balancing via HAProxy Ingress controller (alternative to nginx)

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

## Option 2: LoadBalancer with MetalLB (Production-like, Recommended)

**MetalLB** is a load balancer implementation for bare-metal Kubernetes clusters. It provides **LoadBalancer** services without requiring a cloud provider, making it ideal for Minikube and production-like testing.

### Installation

1. **Install MetalLB**:
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.8/config/manifests/metallb-native.yaml
   ```

2. **Wait for MetalLB to be ready**:
   ```bash
   kubectl wait --namespace metallb-system \
     --for=condition=ready pod \
     --selector=app=metallb \
     --timeout=90s
   ```

3. **Get Minikube IP range**:
   ```bash
   minikube ip
   # Example output: 192.168.49.2 or 172.17.0.2
   ```

4. **Configure MetalLB IP pool**:
   - Edit `k8s/metallb-config.yaml` and update the IP range to match your Minikube subnet
   - For Docker driver: typically `172.17.0.100-172.17.0.200`
   - For other drivers: use a range in the same subnet as Minikube IP
   ```bash
   kubectl apply -f k8s/metallb-config.yaml
   ```

5. **Replace NodePort services with LoadBalancer services**:
   ```bash
   kubectl apply -f k8s/services-loadbalancer.yaml -n crm-rfm
   ```

### Accessing Services

After MetalLB assigns IPs, services will have external IPs:

```bash
# Check assigned IPs
kubectl get svc -n crm-rfm

# Example output:
# NAME       TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)
# crm-api    LoadBalancer   10.96.1.2       172.17.0.100     8000:30080/TCP
# n8n        LoadBalancer   10.96.1.3       172.17.0.101     5678:30678/TCP
# qdrant     LoadBalancer   10.96.1.4       172.17.0.102      6333:30333/TCP,6334:30334/TCP
```

Access services directly via their external IPs:

```bash
# CRM API
curl http://172.17.0.100:8000/health

# n8n UI
open http://172.17.0.101:5678

# Qdrant REST API
curl http://172.17.0.102:6333/collections
```

### Optional: Use Standard Ports

You can configure services to use standard ports (80, 443) by modifying the service definitions:

```yaml
spec:
  type: LoadBalancer
  ports:
    - name: http
      port: 80        # External port
      targetPort: 8000  # Internal container port
```

Then access: `http://172.17.0.100` (no port needed)

### Advantages

- ✅ **Production-like** - Uses standard LoadBalancer type (same as cloud providers)
- ✅ **Standard ports** - Can use ports 80, 443, etc. (not limited to 30000+ range)
- ✅ **Direct IP access** - Each service gets its own IP address
- ✅ **No port-forward needed** - Services accessible from host machine
- ✅ **Works with any Kubernetes** - Not Minikube-specific
- ✅ **Industry standard** - MetalLB is widely used in production bare-metal clusters

### Disadvantages

- ⚠️ **Requires MetalLB installation** - Additional component to deploy
- ⚠️ **IP range configuration** - Must configure IP pool matching your network
- ⚠️ **IP management** - Need to manage IP address allocation

### Troubleshooting

**Services stuck in "Pending" state**:
```bash
# Check MetalLB status
kubectl get pods -n metallb-system
kubectl logs -n metallb-system -l app=metallb

# Verify IP pool configuration
kubectl get ipaddresspool -n metallb-system
kubectl describe ipaddresspool default-pool -n metallb-system
```

**Cannot access external IPs**:
- Ensure IP range is in the same subnet as Minikube
- Check firewall rules (if applicable)
- Verify services have external IPs: `kubectl get svc -n crm-rfm`

---

## Option 3: Ingress with nginx-ingress (Production-like)

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

## Option 4: Ingress with HAProxy (High-Performance Alternative)

**HAProxy** is a high-performance, production-grade load balancer that can be used as an Ingress controller. It's an alternative to nginx-ingress, offering better performance for high-traffic scenarios and advanced load balancing features.

### Installation

1. **Install HAProxy Ingress controller**:
   ```bash
   kubectl apply -f k8s/haproxy-ingress-controller.yaml
   ```

2. **Wait for controller to be ready**:
   ```bash
   kubectl wait --namespace haproxy-ingress \
     --for=condition=ready pod \
     --selector=app.kubernetes.io/name=haproxy-ingress \
     --timeout=300s
   ```

3. **Update Ingress to use HAProxy**:
   Edit `k8s/ingress.yaml` and change `ingressClassName` from `nginx` to `haproxy`:
   ```yaml
   spec:
     ingressClassName: haproxy  # Changed from nginx
   ```

4. **Apply Ingress**:
   ```bash
   kubectl apply -f k8s/ingress.yaml -n crm-rfm
   ```

### Accessing Services via HAProxy

The HAProxy controller is exposed as a **NodePort** service:

| Service | NodePort | Access URL |
|:--|:--|:--|
| **HAProxy HTTP** | `30081` | `http://crm.local` (requires `/etc/hosts` entry) |
| **HAProxy HTTPS** | `30444` | `https://crm.local` (requires `/etc/hosts` entry) |
| **HAProxy Stats** | `31024` | `http://$(minikube ip):31024/stats` (monitoring) |

### Configure Local DNS

Add to `/etc/hosts` (Linux/macOS) or `C:\Windows\System32\drivers\etc\hosts` (Windows):

```
<minikube-ip>  crm.local
```

### Optional: Use MetalLB with HAProxy

For LoadBalancer access, you can combine HAProxy with MetalLB:

1. Install MetalLB (see Option 2)
2. Change HAProxy service type to LoadBalancer:
   ```bash
   kubectl patch svc haproxy-ingress -n haproxy-ingress \
     --type='json' \
     -p='[{"op": "replace", "path": "/spec/type", "value":"LoadBalancer"}]'
   ```
3. HAProxy will get an external IP from MetalLB

### Advantages

- ✅ **High performance** - Optimized for high-traffic scenarios
- ✅ **Advanced load balancing** - Multiple algorithms (round-robin, leastconn, etc.)
- ✅ **Built-in monitoring** - Statistics dashboard on port 1024
- ✅ **Production-grade** - Widely used in enterprise environments
- ✅ **SSL/TLS termination** - Full support for HTTPS
- ✅ **Path-based routing** - Same Ingress features as nginx
- ✅ **Can combine with MetalLB** - Use LoadBalancer type for external IPs

### Disadvantages

- ⚠️ **Requires additional component** - Must install HAProxy Ingress controller
- ⚠️ **Requires DNS configuration** - Need to edit `/etc/hosts` or use DNS
- ⚠️ **More complex setup** - Additional deployment step
- ⚠️ **Less common than nginx** - Fewer examples and community resources

### When to Use HAProxy vs nginx-ingress

**Use HAProxy** when:
- You need maximum performance and throughput
- You require advanced load balancing algorithms
- You want built-in statistics and monitoring
- You're familiar with HAProxy configuration

**Use nginx-ingress** when:
- You prefer a more common, well-documented solution
- You need extensive annotation support
- You want easier configuration and more examples

---

## Comparison

| Feature | NodePort | MetalLB LoadBalancer | nginx Ingress | HAProxy Ingress |
|:--|:--|:--|:--|:--|
| **Setup Complexity** | Simple | Moderate | Moderate | Moderate |
| **Additional Components** | None | MetalLB | nginx-ingress | HAProxy Ingress |
| **Access Method** | IP + Port | IP + Port | Domain name | Domain name |
| **Port Range** | 30000-32767 | Any (80, 443, etc.) | Configurable | Configurable |
| **Production Ready** | Limited | Yes | Yes | Yes |
| **Path Routing** | No | No | Yes | Yes |
| **SSL/TLS** | No | Yes (with service config) | Yes | Yes |
| **Service Type** | NodePort | LoadBalancer | ClusterIP + Ingress | ClusterIP + Ingress |
| **IP Assignment** | Node IP | Dedicated IP per service | Via Ingress IP | Via Ingress IP |
| **Performance** | Good | Good | Very Good | Excellent |
| **Monitoring** | Basic | Basic | Good | Built-in stats |
| **Load Balancing** | Basic | Basic | Good | Advanced algorithms |

---

## Recommended Approach

### For Development/Testing

Use **NodePort** (Option 1) for simplicity:
- No additional setup required
- Immediate access after service deployment
- Easy to script and automate

### For Production-like Testing

Use **MetalLB LoadBalancer** (Option 2) for realistic testing:
- Most production-like approach (standard LoadBalancer type)
- Each service gets dedicated IP address
- Can use standard ports (80, 443)
- Mirrors cloud provider behavior

### For Domain-based Routing

Use **Ingress** (Option 3 or 4) for domain-based access:
- Better for testing domain-based routing
- Single entry point for multiple services
- Supports SSL/TLS configuration
- Can combine with MetalLB (Ingress controller as LoadBalancer)

**Choose nginx-ingress** (Option 3) for:
- More common, well-documented solution
- Extensive annotation support
- Easier configuration

**Choose HAProxy** (Option 4) for:
- Maximum performance and throughput
- Advanced load balancing algorithms
- Built-in statistics dashboard
- Enterprise-grade features

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

- **All methods can coexist** - You can use NodePort, LoadBalancer, and Ingress simultaneously
- **MetalLB + Ingress** - You can use MetalLB to provide LoadBalancer IP for the Ingress controller (nginx or HAProxy), combining both approaches
- **HAProxy vs nginx** - Both are excellent Ingress controllers; choose based on your performance needs and familiarity
- **Security considerations** - NodePort and LoadBalancer expose services directly; Ingress provides additional routing and security features
- **Minikube tunnel** - Alternative to MetalLB for LoadBalancer services (not covered here as it requires additional setup)
- **Service types** - Services are configured as NodePort by default; they can be changed to LoadBalancer (with MetalLB) or ClusterIP (with Ingress)

This approach provides flexible access to services without requiring port-forward or Minikube addons.
