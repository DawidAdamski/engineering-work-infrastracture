# n8n Embedding Worker

The **n8n embedding worker** automates the process of generating and storing customer embeddings.  
It connects **PostgreSQL**, **OpenAI**, and **Qdrant**, enabling continuous synchronization between classical RFM data and AI-driven segmentation.

---

## Purpose

- Periodically read customer data from PostgreSQL.  
- Call the **OpenAI Embedding API** (or another compatible model endpoint).  
- Store resulting vectors in **Qdrant** along with relevant RFM payloads.  
- Mark processed customers in PostgreSQL to avoid duplication.

---

## Deployment Overview

|Item|Value|
|:--|:--|
|**Namespace**|`crm-rfm`|
|**Service name**|`n8n`|
|**Port**|`5678`|
|**Persistence**|Optional – local workflow storage at `/home/node/.n8n`|
|**Access**|Internal ClusterIP or Ingress for UI access (optional)|

Internal URL for in-cluster services:  
`http://n8n.crm-rfm.svc.cluster.local:5678`

---

## Workflow Logic

### 🧠 Step-by-step overview

1. **Trigger**  
   - Cron node runs periodically (e.g., every hour).  
   - Alternatively, a Webhook node can trigger the flow manually.

2. **Fetch customers**  
   - PostgreSQL node executes:
     ```sql
     SELECT id, name, total_spent, last_order_at
     FROM customers
     WHERE embedding_status IS NULL
     LIMIT 50;
     ```

3. **Build profile text**  
   - Function node concatenates attributes into a single text string, e.g.:
     ```
     Customer John Doe spent 1200 USD across 5 orders, last seen 45 days ago.
     ```

4. **Generate embedding**  
   - HTTP Request node → OpenAI API endpoint:
     ```
     POST https://api.openai.com/v1/embeddings
     Headers:
       Authorization: Bearer {{ $env.OPENAI_API_KEY }}
     Body:
       {
         "input": {{$json["profile_text"]}},
         "model": "text-embedding-3-large"
       }
     ```
   - Extract the returned vector from the response.

5. **Upsert to Qdrant**  
   - HTTP Request node → Qdrant:
     ```
     POST http://qdrant.crm-rfm.svc.cluster.local:6333/collections/customers/points
     Body:
       {
         "points": [{
           "id": {{$json["id"]}},
           "vector": {{$json["embedding"]}},
           "payload": {
             "customer_id": {{$json["id"]}},
             "rfm_recency": {{$json["recency"]}},
             "rfm_frequency": {{$json["frequency"]}},
             "rfm_monetary": {{$json["monetary"]}}
           }
         }]
       }
     ```

6. **Update PostgreSQL**  
   - PostgreSQL node marks processed rows:
     ```sql
     UPDATE customers
     SET embedding_status = 'done'
     WHERE id = {{ $json["id"] }};
     ```

7. **Loop or Stop**  
   - Loop until all unprocessed customers are embedded.

---

## Configuration Variables

|Environment Variable|Purpose|
|:--|:--|
|`POSTGRES_HOST`|Internal PostgreSQL host (`postgres.crm-rfm.svc.cluster.local`)|
|`POSTGRES_DB`|Database name|
|`POSTGRES_USER`|Database username|
|`POSTGRES_PASSWORD`|Database password|
|`QDRANT_URL`|Qdrant endpoint (default: `http://qdrant.crm-rfm.svc.cluster.local:6333`)|
|`QDRANT_COLLECTION`|Collection name (`customers`)|
|`OPENAI_API_KEY`|API key for embedding generation|
|`OPENAI_MODEL`|Embedding model name (`text-embedding-3-large`)|
|`N8N_BASIC_AUTH_USER`|Optional username for n8n UI|
|`N8N_BASIC_AUTH_PASSWORD`|Optional password for n8n UI|

---

## Accessing the UI

### Method 1: Port-Forward (Recommended - Easiest)

**Port-forward is the simplest and most reliable method** for accessing n8n:

```bash
kubectl port-forward svc/n8n 5678:5678 -n crm-rfm
```

Then open in your browser: **`http://localhost:5678`**

**Note:** 
- Keep the port-forward terminal session running while you use n8n
- Press `Ctrl+C` to stop the port-forward when done
- Works from any machine that can run `kubectl` commands

### Method 2: LoadBalancer (External IP)

If n8n is deployed as a **LoadBalancer** service (with MetalLB), it will have an external IP:

```bash
# Check the external IP
kubectl get svc n8n -n crm-rfm

# Example output:
# NAME   TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)
# n8n    LoadBalancer   10.104.239.39   192.168.0.201   5678:30569/TCP
```

**Access n8n:**
- **URL:** `http://<EXTERNAL-IP>:5678`
- **Example:** `http://192.168.0.201:5678`

**Note:** LoadBalancer IPs may not be accessible from all networks. If the external IP doesn't work, use port-forward instead.

### Method 3: NodePort (Direct External Access)

If n8n is exposed as a **NodePort** service:

```bash
# Get Minikube IP
minikube ip

# Access n8n (replace <minikube-ip> with actual IP)
open http://$(minikube ip):30678
```

### Method 4: From Within Kubernetes Cluster

Applications running inside the cluster can connect using the internal DNS:

**URL:**
```
http://n8n.crm-rfm.svc.cluster.local:5678
```

**Environment variables:**
- `N8N_URL=http://n8n.crm-rfm.svc.cluster.local:5678`
- `N8N_HOST=n8n.crm-rfm.svc.cluster.local`
- `N8N_PORT=5678`
- `N8N_PROTOCOL=http`

### Authentication & First-Time Setup

**Default behavior:** n8n does **not require authentication** by default on first access.

**First-time access:**
1. Open the n8n URL in your browser (e.g., `http://192.168.0.201:5678`)
2. You will be prompted to **create your first user account**
3. Enter your email and password to create the admin account
4. After creating the account, you'll be logged in automatically

**Subsequent logins:**
- Use the email and password you created during first-time setup
- If you forgot your password, you can reset it (if email is configured) or recreate the n8n instance

### Optional: Basic Authentication

To enable basic authentication (username/password) before accessing n8n, set these environment variables in the deployment:

```yaml
env:
  - name: N8N_BASIC_AUTH_USER
    value: "admin"
  - name: N8N_BASIC_AUTH_PASSWORD
    valueFrom:
      secretKeyRef:
        name: crm-secrets
        key: N8N_BASIC_AUTH_PASSWORD
```

**Note:** Basic authentication is **optional** and not configured by default. If not set, n8n uses its own user management system (created on first access).

---

## Security Notes
Store secrets (OpenAI API key, DB credentials) in Kubernetes Secrets, not ConfigMaps.

Limit external access to n8n — keep it internal during development.

Use HTTPS or network policies for sensitive deployments.

For the full system architecture, see [README.architecture.md](README.architecture.md).

---

## Notes
n8n acts as a bridge between traditional relational data and AI-based representations.

The workflow can be exported/imported as JSON (.n8n file) for version control.

This worker design allows you to easily swap OpenAI for another embedding model (Gemini, local model server, etc.).

It represents the AI automation layer of your engineering thesis — the component that demonstrates cloud-native, asynchronous data enrichment.