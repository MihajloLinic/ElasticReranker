Guidelines: Deploying and Operating Elasticsearch + Reranker Search

Purpose
This document explains how to deploy Elasticsearch, connect it to real production data, and deploy the reranker model service for production usage. It complements the README overview with deeper, actionable details.

1. System overview
- Retrieval: Elasticsearch (ES) returns a broad set of candidates quickly using BM25/multi_match (optionally hybrid).
- Reranking: A cross‑encoder model (e.g., jinaai/jina-reranker-v2-base-multilingual or your fine‑tuned model) scores (query, candidate_text) pairs and reorders the results.
- Output: Top‑N results returned to the application. Optionally blend ES score and reranker score.

2. Environments
- Development:
  - Run ES locally via Docker.
  - Use sample.ipynb to ingest sample data and test queries + reranking.
- Staging/Production:
  - Elastic Cloud (managed) or self‑hosted ES (Docker/Kubernetes/VMs) with security enabled.
  - Reranker deployed as a stateless HTTP service, horizontally scalable, with batching and observability.

3. Elasticsearch deployment
3.1 Local (development) via Docker (PowerShell)
- One‑liner (insecure Dev only):
  docker run -d --name es-dev -p 9200:9200 -e "discovery.type=single-node" -e "xpack.security.enabled=false" docker.elastic.co/elasticsearch/elasticsearch:8.14.0
- Health check: curl http://127.0.0.1:9200

3.2 Production options
- Elastic Cloud: Easiest path, managed TLS/auth/snapshots.
- Self‑hosted Docker Compose: Enable security, configure passwords, mount persistent volumes, expose HTTPS behind a reverse proxy.
- Kubernetes with Elastic ECK operator: Recommended for larger deployments and declarative ops.

3.3 Example: Docker Compose (single node, security on)
version: "3.8"
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.14.0
    container_name: es-prod
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=true
      - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
      - ES_JAVA_OPTS=-Xms4g -Xmx4g
    ports:
      - "9200:9200"
    volumes:
      - esdata:/usr/share/elasticsearch/data
volumes:
  esdata:
Notes:
- Provide ELASTIC_PASSWORD via an environment variable or secret store.
- Front with a reverse proxy that terminates TLS (or configure TLS in ES directly).

3.4 Kubernetes (ECK) sketch
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: es-prod
spec:
  version: 8.14.0
  nodeSets:
  - name: default
    count: 3
    config:
      node.store.allow_mmap: false
      xpack.security.enabled: true
    podTemplate:
      spec:
        containers:
        - name: elasticsearch
          resources:
            requests: { memory: 8Gi, cpu: "2" }
            limits:   { memory: 16Gi, cpu: "4" }
    volumeClaimTemplates:
    - metadata: { name: elasticsearch-data }
      spec:
        accessModes: [ "ReadWriteOnce" ]
        resources: { requests: { storage: 200Gi } }

3.5 Production settings and best practices
- Security: Always enable TLS and use API keys/service accounts. Restrict network access.
- Sizing: Start with 2–3 data nodes. Heap ~50% of RAM (up to 30 GB).
- Backups: Configure snapshot repositories (S3/Blob/GCS) and schedule automated snapshots.
- Observability: Enable slow logs and use Kibana/Elastic Observability.
- Index lifecycle: Use ILM for hot/warm/cold tiers if data grows large.

4. Index design and analyzers
4.1 Mappings example (products)
PUT products
{
  "settings": {
    "analysis": {
      "analyzer": {
        "default": { "type": "standard" }
      }
    }
  },
  "mappings": {
    "properties": {
      "Product Name": { "type": "text", "fields": { "keyword": { "type": "keyword" } } },
      "About Product": { "type": "text" },
      "Product Specification": { "type": "text" },
      "Technical Details": { "type": "text" },
      "Category": { "type": "keyword" },
      "price": { "type": "float" },
      "updated_at": { "type": "date" }
    }
  }
}

4.2 Multilingual
- Consider language analyzers per language (or multi‑fields with language‑specific analyzers).
- Synonyms: curate synonym sets for head terms; use search‑time synonyms for recall.

5. Connecting to a real database and indexing into ES
5.1 Python bulk indexing pattern
from elasticsearch import Elasticsearch, helpers
import psycopg2

def fetch_rows():
    # Replace with your DB access; ensure pagination/streaming
    conn = psycopg2.connect("dbname=app user=app password=*** host=host port=5432")
    cur = conn.cursor(name="products_cursor")  # server-side cursor
    cur.itersize = 1000
    cur.execute("SELECT id, name, description, spec, details, category FROM products")
    for row in cur:
        yield {
            "_id": row[0],
            "_source": {
                "Product Name": row[1],
                "About Product": row[2],
                "Product Specification": row[3],
                "Technical Details": row[4],
                "Category": row[5],
            }
        }

ES_URL = "https://your-es:9200"
es = Elasticsearch(ES_URL, api_key=("id", "secret"))
INDEX = "products"
helpers.bulk(es, ({"_index": INDEX, **doc} for doc in fetch_rows()))

Best practices
- Use stable IDs to upsert.
- Stream in batches; avoid loading entire tables in memory.
- Normalize/clean text fields; ensure consistent field names used by search_fields.

5.2 Logstash JDBC (relational DBs)
input {
  jdbc {
    jdbc_driver_library => "path/to/driver.jar"
    jdbc_driver_class => "org.postgresql.Driver"
    jdbc_connection_string => "jdbc:postgresql://host:5432/db"
    jdbc_user => "user"
    jdbc_password => "pass"
    statement => "SELECT id, name AS \"Product Name\", description AS \"About Product\" FROM products"
    schedule => "*/5 * * * *"
  }
}
filter { }
output {
  elasticsearch {
    hosts => ["https://your-es:9200"]
    index => "products"
    user => "elastic"
    password => "<secret>"
    ssl => true
    cacert => "/path/ca.crt"
  }
}

5.3 CDC and non‑SQL sources
- Debezium (Postgres/MySQL/Mongo) to stream changes to Kafka → Logstash/Elastic Agent → ES.
- Monstache for MongoDB → ES replication.

6. Application queries and reranking
Search fields example (align with your mappings):
search_fields = [
  "Product Name",
  "About Product",
  "Product Specification",
  "Technical Details",
  "Category",
]

- Perform multi_match with top_k=50–200.
- Build candidate_text per hit (title + 1–3 key fields, trimmed).
- Call the reranker to score and sort; optionally blend with ES _score (min‑max normalized).

7. Deploying the reranker model
7.1 Service pattern
- Stateless HTTP API that accepts { query, docs: [candidate_text...] } and returns scores.
- Runs on CPU or GPU; batch requests for throughput; use FP16 when on CUDA.

7.2 FastAPI service (simplified)
from fastapi import FastAPI
from pydantic import BaseModel
from transformers import AutoTokenizer, AutoModelForSequenceClassification
import torch

MODEL_NAME = "jinaai/jina-reranker-v2-base-multilingual"  # replace with your fine‑tuned model
DEVICE = "cuda" if torch.cuda.is_available() else "cpu"
BATCH_SIZE = int(os.getenv("BATCH_SIZE", 16))

app = FastAPI()
_tok = AutoTokenizer.from_pretrained(MODEL_NAME, trust_remote_code=True)
_model = AutoModelForSequenceClassification.from_pretrained(MODEL_NAME, trust_remote_code=True).eval().to(DEVICE)

class Item(BaseModel):
  query: str
  docs: list[str]

@app.post("/rerank")
def rerank(item: Item):
  # Tokenize in batches; compute scores (see sample.ipynb for scoring details)
  return {"scores": [0.0 for _ in item.docs]}

7.3 Containerization (Dockerfile example)
FROM python:3.10-slim
ENV DEBIAN_FRONTEND=noninteractive
RUN pip install --no-cache-dir fastapi uvicorn transformers torch --extra-index-url https://download.pytorch.org/whl/cpu
# For CUDA builds, use a CUDA base image and install the matching torch wheels.
ENV MODEL_NAME=jinaai/jina-reranker-v2-base-multilingual
# Optional: pre-download model to Docker layer
RUN python - <<"PY"
from transformers import AutoTokenizer, AutoModelForSequenceClassification
m = "${MODEL_NAME}"
AutoTokenizer.from_pretrained(m, trust_remote_code=True)
AutoModelForSequenceClassification.from_pretrained(m, trust_remote_code=True)
PY
COPY app.py /app/app.py
WORKDIR /app
EXPOSE 8000
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]

7.4 Kubernetes deployment (GPU optional)
apiVersion: apps/v1
kind: Deployment
metadata: { name: reranker }
spec:
  replicas: 2
  selector: { matchLabels: { app: reranker } }
  template:
    metadata: { labels: { app: reranker } }
    spec:
      containers:
      - name: reranker
        image: yourrepo/reranker:latest
        ports: [ { containerPort: 8000 } ]
        env:
        - { name: MODEL_NAME, value: "your-finetuned-model" }
        resources:
          requests: { cpu: "500m", memory: "1Gi" }
          limits:   { cpu: "2", memory: "4Gi" }
      # For GPU nodes:
      #   resources:
      #     limits:
      #       nvidia.com/gpu: 1
---
apiVersion: v1
kind: Service
metadata: { name: reranker }
spec:
  selector: { app: reranker }
  ports: [ { port: 80, targetPort: 8000 } ]

7.5 Scaling and performance
- Batching: collect small batches (e.g., up to 16–32 docs) for higher throughput; bound by latency SLOs.
- Mixed precision: enable FP16 on CUDA for speed.
- Quantization: consider 8‑bit (bitsandbytes) or ONNX Runtime for CPU.
- Timeouts: ES query timeout (client‑side) and reranker service timeout safeguards.
- Fallbacks: if reranker is unavailable, return ES‑only ranking with a warning.

8. Configuration and security
- ES: Use API keys or service accounts; TLS; restricted firewall/VPC; regular snapshot backups.
- Reranker API: Protect with mTLS or JWT/API keys; rate limiting; request size limits.
- Secrets: Inject via environment variables or secret managers (Kubernetes Secrets, Vault), never hardcode.
- HF private models: Use HUGGINGFACE_HUB_TOKEN to pull private fine‑tuned models.

9. Observability and QA
- Metrics: QPS, latency per stage (ES, reranker), cache hit ratio, error rates.
- Logs: Structured JSON logs with correlation IDs across ES and reranker calls.
- Quality: Track precision@K/nDCG@K on a labeled set; alert on regressions.
- Tracing: OpenTelemetry instrumentation for end‑to‑end spans.

10. Rollout and CI/CD
- Canary: Route a small % of traffic to the new model; compare metrics.
- A/B testing: Compare ranking quality before switching fully.
- CI: Lint/type‑check service; unit tests for candidate_text building and ranking pipeline.
- CD: Build image, run smoke tests against a staging ES, deploy progressively.

11. Adapting the code to your dataset
- Map your schema’s title/description/spec fields to the candidate_text builder inputs.
- Update search_fields to match your index mappings.
- Ensure IDs are stable; choose which fields to highlight; adjust max_chars_per_field and max_total_chars for your content.

12. End‑to‑end request flow (production)
1) App issues ES query for top_k candidates.
2) App builds candidate_text for each hit.
3) App calls reranker API with query + candidate_text list.
4) Sort by reranker score (or blended score) and return top‑N.
5) Log, trace, and monitor metrics; fallback to ES‑only if reranker errors.

References
- sample.ipynb: Contains runnable reference implementation of ES retrieval and reranking utilities.
- Elastic docs: https://www.elastic.co/guide/
- ECK operator: https://www.elastic.co/guide/en/cloud-on-k8s/current/index.html
- Transformers: https://huggingface.co/docs/transformers/index


---

Contributing, Privacy, and Training Guidelines (Updated 2025-09-27)

1. Contribution workflow
- Branching
  - Use short, descriptive names: feature/<topic>, fix/<topic>, docs/<topic>, ops/<topic>.
- Pull Requests
  - PR description: what changed, why, and how to validate.
  - Include screenshots or metric tables when relevant (e.g., nDCG/MRR deltas).
  - Link to any notebooks or scripts used for evaluation; ensure they run on Windows (PowerShell) with the provided requirements.txt.
- Commit messages (Conventional Commits)
  - feat: add Jina reranker integration to service
  - fix: handle empty description when building candidate_text
  - docs: add Serbian-only fine-tuning quick guide
  - refactor: extract ES client init
  - perf: batch reranker requests
- Code style and structure
  - Python 3.9+. Prefer type hints and clear docstrings.
  - Keep notebook cells small and reusable. If logic grows, extract a .py helper and import into the notebook.
  - Use Windows-friendly paths in docs and examples (e.g., models\fine_tuned_reranker).
  - Do not introduce hard-coded credentials or file system paths; use environment variables.

2. Secrets and data privacy
- Never commit secrets
  - Keep credentials (Elasticsearch, API keys, tokens) only in environment variables or secret stores.
  - Suggested environment variables: ES_URL, ES_API_KEY_ID, ES_API_KEY_SECRET, RERANKER_MODEL, BATCH_SIZE.
  - Local-only .env files must not be committed.
- Product data governance
  - Treat product data and any user interaction logs as confidential.
  - Do not commit raw datasets. Use small, sanitized samples for examples.
  - Ensure logs and metrics do not contain PII.
- Models and large artifacts
  - Do not commit large model files or HF caches. Store fine-tuned models outside the repo (e.g., models\fine_tuned_reranker locally, artifact storage remotely).

3. Model training and evaluation policy
- Task and models
  - Cross-encoder reranking with jinaai/jina-reranker-v2-base-multilingual as the default base.
  - For Serbian specialization, prefer supervised fine-tuning on Serbian pairs (pointwise or pairwise) with hard negatives.
- Training choices
  - Full fine-tuning: LR 1e-5–2e-5, epochs 1–3, weight decay 0.01, warmup 5–10%.
  - Adapter (LoRA) fine-tuning: LR 2e-4–5e-4, r≈16, alpha≈32, dropout≈0.05; train attention Q/K/V (+optional output dense).
  - For an extra Serbian specialization pass: LR 5e-6–1e-5, 1–2 epochs.
- Evaluation requirements
  - Maintain a fixed candidate pool per query (e.g., top 100 from ES) for fair comparisons.
  - Report at least nDCG@10 and MRR@10; include Recall@50 if labels allow.
  - Track average and P95 reranking latency for your batch size and hardware.
  - Fix a random seed; document dataset split versions and label sources.
- Acceptance criteria (guideline)
  - New models should not regress prior best by more than 1% absolute on nDCG@10; otherwise, provide justification.
  - Include a short markdown or CSV with metrics in the PR (do not commit large artifacts).

4. Serbian-specific robustness guidelines
- Script and diacritics
  - Cover both Latin and Cyrillic scripts during training/validation.
  - Include diacritics-on and diacritics-off variants to mirror user behavior.
- Morphology and variants
  - Include Ekavian/Ijekavian where relevant.
- Token preservation
  - Preserve model numbers/brands and hyphenated tokens; avoid over-normalization.

5. Documentation policy
- When you change how indexing, retrieval, reranking, or evaluation works:
  - Update README.md (append an “Updated YYYY-MM-DD” section) with user-facing changes.
  - If training guidance changes, update jinaai_jina-reranker-v2-base-multilingua.md.
  - Keep Windows PowerShell examples and backslash paths for local usage.
- Keep examples minimal and copy-paste runnable.

6. Release and deployment notes
- Reranker service
  - Expose a simple /rerank endpoint that accepts { query, docs } and returns scores.
  - Batch inputs for throughput; enable FP16 on CUDA where available.
  - Configure via env vars (MODEL path/name, BATCH_SIZE, device selection); never hard-code secrets.
- Elasticsearch
  - In production, enable TLS and auth; prefer API keys. Configure snapshots for backups.

7. Manual QA checklist (before merging)
- Run sample.ipynb end-to-end on Windows with ES running (Docker or Elastic Cloud).
- Verify that reranking improves the top-10 ordering against ES-only for a small labeled set.
- Confirm that no secrets are exposed in code, configs, or outputs.
- Ensure README.md and this guidelines file reflect any behavior changes (date-stamped updates).

See also
- README.md for project overview, quickstart, evaluation guidance, and Serbian specialization quick guide.
- jinaai_jina-reranker-v2-base-multilingua.md for detailed fine-tuning playbook (pointwise/pairwise, hard negatives, PEFT/LoRA, and evaluation).
