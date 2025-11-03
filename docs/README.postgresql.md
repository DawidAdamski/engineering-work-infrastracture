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
kubectl exec -it deploy/postgres -n crm-rfm -- \
  pg_dump -U crmuser crmdb > backup.sql
```

### To Import

```bash
kubectl exec -i deploy/postgres -n crm-rfm -- \
  psql -U crmuser -d crmdb < backup.sql
```

---

## Notes

- PostgreSQL is the authoritative source of CRM data.
- Classical RFM logic remains fully interpretable and auditable here.
- Qdrant and OpenAI integrations never alter core transactional data — they only extend it semantically.
- In Minikube, data persists locally in the mounted PVC volume between restarts.