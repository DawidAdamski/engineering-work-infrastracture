# PostgreSQL

PostgreSQL provides the **relational data layer** for the CRM system.  
It stores customer information, orders, and transactional data used for **classical RFM (Recency, Frequency, Monetary)** analysis.

---

## Purpose

- Maintain persistent customer and order records.
- Support SQL-based analytical queries for RFM segmentation.
- Serve as the baseline for comparison with AI-based segmentation (Qdrant embeddings).
- Provide structured data for the n8n workflow to read and enrich with embeddings.

---

## Deployment Overview

|Item|Value|
|:--|:--|
|**Namespace**|`crm-rfm`|
|**Service name**|`postgres`|
|**Port**|`5432`|
|**Persistence**|PVC attached to `/var/lib/postgresql/data`|
|**Credentials**|Stored in a Kubernetes Secret (`postgres-secret`)|

PostgreSQL runs as a **StatefulSet** to preserve data across Pod restarts.

Internal DNS address for in-cluster access:  
postgres.crm-rfm.svc.cluster.local:5432


---

## Schema Concept

The database contains three main tables:

|Table|Description|
|:--|:--|
|`customers`|Basic client information (ID, name, registration date, etc.).|
|`orders`|Each customer’s purchase transactions.|
|`order_items`|Detailed items per order (used to calculate total value).|

---

## Example RFM Calculation (SQL)

```sql
WITH rfm_calc AS (
  SELECT
    c.id AS customer_id,
    DATE_PART('day', NOW() - MAX(o.created_at)) AS recency,
    COUNT(o.id) AS frequency,
    SUM(o.amount) AS monetary
  FROM customers c
  LEFT JOIN orders o ON o.customer_id = c.id
  GROUP BY c.id
)
SELECT *
FROM rfm_calc
ORDER BY recency ASC;
```
This query:
- Calculates how recently each customer made a purchase (recency),
- How often (frequency), and
- How much they spent (monetary).

The CRM API uses this logic to expose classical RFM metrics over HTTP endpoints.

---

## Integration with Other Components

**CRM API** connects directly to PostgreSQL for CRUD and analytics operations.

**n8n embedding workflow** queries PostgreSQL for customers that have no embedding yet:

```sql
SELECT id, name, total_spent, last_order_at
FROM customers
WHERE embedding_status IS NULL
LIMIT 50;
```

**Qdrant** does not interact with PostgreSQL directly — synchronization is handled by n8n.

---

## Configuration Variables

| Variable | Purpose |
|:--|:--|
| `POSTGRES_DB` | Database name (default: `crmdb`) |
| `POSTGRES_USER` | Database username (e.g. `crmuser`) |
| `POSTGRES_PASSWORD` | Database password (in Secret) |
| `PGDATA` | Optional override for data directory path |

Environment variables are injected from a Secret named `postgres-secret`.

---

## Connecting to PostgreSQL

There are several ways to connect to PostgreSQL depending on your use case:

### Method 1: Direct Pod Access (kubectl exec)

Connect directly to the PostgreSQL pod using `kubectl exec`:

```bash
# Interactive psql session
kubectl exec -it postgres-0 -n crm-rfm -- psql -U crmuser -d crmdb
```

**Note:** PostgreSQL runs as a **StatefulSet**, so the pod name is `postgres-0` (not `deploy/postgres`).

Once connected, you can run SQL queries:

```sql
-- List all tables
\dt

-- Query customers
SELECT * FROM customers LIMIT 10;

-- Exit psql
\q
```

### Method 2: Port-Forward (Local Access)

Forward the PostgreSQL port to your local machine:

```bash
# Port-forward PostgreSQL service
kubectl port-forward svc/postgres 5432:5432 -n crm-rfm
```

Then connect from your local machine using any PostgreSQL client:

```bash
# Using psql (if installed locally)
# Replace <username> and <password> with actual values from postgres-secret
psql -h localhost -p 5432 -U <username> -d <database>

# Connection string for applications
# Format: postgresql://<username>:<password>@localhost:5432/<database>
postgresql://crmuser:changeme@localhost:5432/crmdb
```

**Note:** Replace `<username>`, `<password>`, and `<database>` with actual values retrieved from the `postgres-secret` (see "Retrieving Credentials" section below).

**Note:** Keep the port-forward terminal session running while you use the connection.

### Method 3: NodePort (Direct External Access)

PostgreSQL is exposed as a **NodePort** service on port `30432`:

```bash
# Get Minikube IP
minikube ip

# Connect using psql (replace <minikube-ip> with actual IP)
psql -h $(minikube ip) -p 30432 -U crmuser -d crmdb
```

**Connection string:**
```
# Format: postgresql://<username>:<password>@$(minikube ip):30432/<database>
postgresql://crmuser:changeme@$(minikube ip):30432/crmdb
```

**Note:** Replace credentials with actual values from your `postgres-secret`.

### Method 4: From Within Kubernetes Cluster

Applications running inside the cluster can connect using the internal DNS:

**Connection string:**
```
# Format: postgresql://<username>:<password>@postgres.crm-rfm.svc.cluster.local:5432/<database>
postgresql://crmuser:changeme@postgres.crm-rfm.svc.cluster.local:5432/crmdb
```

**Environment variables** (use actual values from `postgres-secret`):
- `POSTGRES_HOST=postgres.crm-rfm.svc.cluster.local`
- `POSTGRES_PORT=5432`
- `POSTGRES_USER=<username>` (default: `crmuser`)
- `POSTGRES_PASSWORD=<password>` (default: `changeme`)
- `POSTGRES_DB=<database>` (default: `crmdb`)

### Retrieving Credentials from Kubernetes Secret

**⚠️ Important:** Always verify the actual credentials from your deployed secret, as they may differ from the defaults.

To retrieve the actual credentials from your cluster:

```bash
# Get username
kubectl get secret postgres-secret -n crm-rfm \
  -o jsonpath="{.data.POSTGRES_USER}" | base64 -d && echo

# Get password
kubectl get secret postgres-secret -n crm-rfm \
  -o jsonpath="{.data.POSTGRES_PASSWORD}" | base64 -d && echo

# Get database name
kubectl get secret postgres-secret -n crm-rfm \
  -o jsonpath="{.data.POSTGRES_DB}" | base64 -d && echo
```

**Default credentials** (as defined in `k8s/secrets.yaml` - may not match your actual deployment):
- **Username:** `crmuser`
- **Password:** `changeme`
- **Database:** `crmdb`

**Note:** If you've customized the secret or it was generated differently, use the commands above to retrieve the actual values from your cluster.

### Using a PostgreSQL Client Tool

If you prefer a GUI client (e.g., DBeaver, pgAdmin, TablePlus), use **Method 2 (Port-Forward)** and connect with:

- **Host:** `localhost`
- **Port:** `5432`
- **Database:** Retrieve from `postgres-secret` (default: `crmdb`)
- **Username:** Retrieve from `postgres-secret` (default: `crmuser`)
- **Password:** Retrieve from `postgres-secret` (default: `changeme`)

**⚠️ Always verify actual credentials using the commands in the "Retrieving Credentials" section above.**

---

## Initialization Script

If you want to bootstrap schema automatically, you can mount an init SQL file in /docker-entrypoint-initdb.d/ or use an initContainer.

Example contents:

CREATE TABLE customers (
  id SERIAL PRIMARY KEY,
  name TEXT,
  registered_at TIMESTAMP DEFAULT NOW(),
  last_order_at TIMESTAMP,
  total_spent NUMERIC,
  embedding_status TEXT
);

CREATE TABLE orders (
  id SERIAL PRIMARY KEY,
  customer_id INT REFERENCES customers(id),
  created_at TIMESTAMP DEFAULT NOW(),
  amount NUMERIC
);
```

---

## Backup and Restore (Local Testing)

### To Export Data

```bash
kubectl exec -it postgres-0 -n crm-rfm -- \
  pg_dump -U crmuser crmdb > backup.sql
```

### To Import

```bash
kubectl exec -i postgres-0 -n crm-rfm -- \
  psql -U crmuser -d crmdb < backup.sql
```

---

## Notes

- PostgreSQL is the authoritative source of CRM data.
- Classical RFM logic remains fully interpretable and auditable here.
- Qdrant and OpenAI integrations never alter core transactional data — they only extend it semantically.
- In Minikube, data persists locally in the mounted PVC volume between restarts.