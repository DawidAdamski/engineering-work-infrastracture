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

To view or modify the workflow graphically:

```bash
minikube service n8n -n crm-rfm
```

Or forward the port:

```bash
kubectl port-forward svc/n8n 5678:5678 -n crm-rfm
```

Then open `http://localhost:5678`.

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