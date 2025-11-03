# CRM API (Application Layer)

The **CRM API** is the core backend service of the project *“RFM Analysis Using Artificial Intelligence in a CRM Application Deployed on the Kubernetes Platform.”*  
It provides REST endpoints for classical RFM calculations and AI-based segmentation via Qdrant vector search.

This component is developed and deployed from a separate repository:  
➡️ [https://github.com/DawidAdamski/engineering-work-crm](https://github.com/DawidAdamski/engineering-work-crm)

---

## Purpose

- Serve as the **application logic layer** between databases and frontend.  
- Provide endpoints for:
  - CRUD operations on customers and orders.
  - Classical RFM analytics (PostgreSQL).
  - AI-based similarity search (Qdrant).  
- Expose clean REST or JSON endpoints for integration and testing.  
- Run as a self-contained container built using **Source-to-Image (S2I)**.

---

## Deployment Model (S2I)

The CRM API is packaged and deployed using the **Source-to-Image** (S2I) workflow, allowing rapid rebuilds directly from source code.

### S2I build example

```bash
s2i build https://github.com/DawidAdamski/engineering-work-crm \
    registry.access.redhat.com/ubi9/python-311 \
    crm-api:latest
```

This produces a local container image `crm-api:latest` containing:
- Application source code,
- Installed Python dependencies (from `requirements.txt`),
- WSGI entrypoint (e.g. `app:app` for FastAPI/Flask).

You can then deploy it into Kubernetes as part of this infrastructure:

```bash
kubectl apply -f k8s/crm-api.yaml -n crm-rfm
```

---

| Variable | Purpose |
|:--|:--|
| `POSTGRES_HOST` | Hostname of the PostgreSQL service (`postgres.crm-rfm.svc.cluster.local`) |
| `POSTGRES_DB` | Database name (e.g. `crmdb`) |
| `POSTGRES_USER` | Database user (from Secret) |
| `POSTGRES_PASSWORD` | Database password (from Secret) |
| `QDRANT_URL` | Qdrant API endpoint (`http://qdrant.crm-rfm.svc.cluster.local:6333`) |
| `QDRANT_COLLECTION` | Collection name for embeddings (`customers`) |
| `OPENAI_MODEL` | Embedding model name (`text-embedding-3-large`) |
| `LOG_LEVEL` | Optional, defaults to `INFO` |

---

## API Structure

| Endpoint | Method | Description |
|:--|:--|:--|
| `/api/customers` | GET | List all customers. |
| `/api/customers/{id}` | GET | Get a single customer record. |
| `/api/customers` | POST | Add a new customer. |
| `/api/rfm` | GET | Compute RFM metrics from PostgreSQL. |
| `/api/similar/{id}` | GET | Find customers similar to a given one using Qdrant embeddings. |
| `/health` | GET | Basic health check endpoint. |

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
          image: crm-api:latest
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

## Notes

The CRM API is stateless, so it can scale horizontally by increasing replicas.

Built images can be pushed to any registry (quay.io, ghcr.io, Docker Hub, or OpenShift internal registry).

It is designed to run both locally (via Docker Compose or S2I) and in Kubernetes (via Deployment manifest).

Logging is standardized to JSON format for compatibility with future monitoring tools (Grafana Loki, etc.).

The API layer connects the analytical (PostgreSQL) and semantic (Qdrant) data worlds, enabling your RFM + AI comparison experiments.

