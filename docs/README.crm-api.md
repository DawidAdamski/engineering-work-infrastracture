# CRM API (Application Layer)

The **CRM API** is the core backend service of the project *"RFM Analysis Using Artificial Intelligence in a CRM Application Deployed on the Kubernetes Platform."*  
It provides REST endpoints for classical RFM calculations and AI-based segmentation via n8n workflows.

This component is developed and deployed from a separate repository:  
➡️ [https://github.com/DawidAdamski/engineering-work-crm](https://github.com/DawidAdamski/engineering-work-crm)

---

## Purpose

- Serve as the **application logic layer** between databases and frontend.  
- Provide endpoints for:
  - CRUD operations on customers and orders.
  - Classical RFM analytics (PostgreSQL).
  - AI-based similarity search (via n8n workflows).  
- Expose clean REST or JSON endpoints for integration and testing.  
- Run as a self-contained container built using standard Docker build process.

---

## Deployment Model

The CRM API is packaged as a standard Docker container image, built from the application source code.

### Docker Image

The CRM API image is available on Docker Hub:
- **Image:** `anihilat/crm-api`
- **Registry:** [Docker Hub](https://hub.docker.com/r/anihilat/crm-api)

### Pull and Use the Image

```bash
# Pull the latest image from Docker Hub
docker pull anihilat/crm-api:latest

# Or pull a specific version
docker pull anihilat/crm-api:v1.0.0
```

### Build from Source (Optional)

If you want to build the image locally from source:

```bash
# Build the image from the CRM repository
cd /path/to/engineering-work-crm
docker build -t anihilat/crm-api:latest .

# Push to Docker Hub (if you have access)
docker push anihilat/crm-api:latest
```

The container image contains:
- Application source code,
- Installed Python dependencies (from `requirements.txt`),
- WSGI entrypoint (e.g. `app:app` for FastAPI/Flask).

### Deploy to Kubernetes

```bash
kubectl apply -f k8s/crm-api.yaml -n crm-rfm
```

---

## Configuration Variables

| Variable | Purpose |
|:--|:--|
| `POSTGRES_HOST` | Hostname of the PostgreSQL service (`postgres.crm-rfm.svc.cluster.local`) |
| `POSTGRES_DB` | Database name (e.g. `crmdb`) |
| `POSTGRES_USER` | Database user (from Secret) |
| `POSTGRES_PASSWORD` | Database password (from Secret) |
| `N8N_URL` | n8n API endpoint for similarity search workflows (`http://n8n.crm-rfm.svc.cluster.local:5678`) |
| `LOG_LEVEL` | Optional, defaults to `INFO` |

**Note:** CRM API does **not** connect directly to Qdrant. Instead, it communicates with **n8n workflows** that handle Qdrant operations. This separation of concerns allows n8n to manage all vector database interactions.

---

## API Structure

| Endpoint | Method | Description |
|:--|:--|:--|
| `/api/customers` | GET | List all customers. |
| `/api/customers/{id}` | GET | Get a single customer record. |
| `/api/customers` | POST | Add a new customer. |
| `/api/rfm` | GET | Compute RFM metrics from PostgreSQL. |
| `/api/similar/{id}` | GET | Find customers similar to a given one via n8n workflow (which queries Qdrant). |
| `/health` | GET | Basic health check endpoint. |

### Integration with n8n for Similarity Search

The CRM API delegates AI-based similarity search to **n8n workflows**:

1. **CRM API receives request** → `/api/similar/{id}`
2. **CRM API calls n8n webhook/API** → Triggers similarity search workflow
3. **n8n workflow** → Queries Qdrant for similar customers
4. **n8n returns results** → Similar customer IDs and scores
5. **CRM API formats response** → Returns JSON to client

### Example Query

```bash
curl http://crm.local/api/similar/42
```

Response:

```json
{
  "base_customer": 42,
  "similar_customers": [
    { "id": 88, "score": 0.91 },
    { "id": 97, "score": 0.88 }
  ]
}
```

**Internal flow:**
- CRM API calls n8n webhook: `POST http://n8n.crm-rfm.svc.cluster.local:5678/webhook/similarity-search`
- n8n workflow queries Qdrant and returns results
- CRM API formats and returns the response

---

## Deployment Manifest (Example)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: crm-api
  namespace: crm-rfm
spec:
  replicas: 1
  selector:
    matchLabels:
      app: crm-api
  template:
    metadata:
      labels:
        app: crm-api
    spec:
      containers:
        - name: crm-api
          image: anihilat/crm-api:latest
          ports:
            - containerPort: 8000
          envFrom:
            - configMapRef:
                name: crm-config
            - secretRef:
                name: crm-secrets
```

---

```yaml
apiVersion: v1
kind: Service
metadata:
  name: crm-api
  namespace: crm-rfm
spec:
  type: ClusterIP
  ports:
    - port: 8000
      targetPort: 8000
  selector:
    app: crm-api
```

---

For the full system architecture, see [README.architecture.md](README.architecture.md).

---

## Integration Architecture

```
┌─────────────┐
│   Client    │
└──────┬──────┘
       │ HTTP Request
       ▼
┌─────────────────┐
│    CRM API      │
│  (FastAPI/Flask)│
└──────┬──────────┘
       │
       ├──► PostgreSQL (RFM queries)
       │
       └──► n8n Workflow (similarity search)
                │
                └──► Qdrant (vector search)
```

**Key Points:**
- CRM API connects **directly** to PostgreSQL for classical RFM calculations
- CRM API connects to **n8n** (not directly to Qdrant) for AI-based similarity search
- n8n workflows handle all Qdrant interactions, keeping the architecture clean and maintainable
- This separation allows n8n to manage embedding generation, storage, and retrieval independently

---

## Notes

The CRM API is stateless, so it can scale horizontally by increasing replicas.

Built images can be pushed to any registry (Docker Hub, ghcr.io, quay.io, etc.).

It is designed to run both locally (via Docker Compose) and in Kubernetes (via Deployment manifest).

Logging is standardized to JSON format for compatibility with future monitoring tools (Grafana Loki, etc.).

The API layer connects the analytical (PostgreSQL) and semantic (Qdrant via n8n) data worlds, enabling your RFM + AI comparison experiments.

