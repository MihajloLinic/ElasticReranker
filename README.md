Elasticsearch + Reranker (Fine‑tuned or Jina) — End‑to‑End Search

Overview
This project retrieves candidates from Elasticsearch (ES) and then reranks them with a cross‑encoder model to return the best results to the customer. In development, we use the multilingual cross‑encoder jinaai/jina-reranker-v2-base-multilingual; in production you can swap in your own fine‑tuned model with the same interface.

Key idea
- Use ES to recall a broad set of relevant candidates quickly (BM25/multi_match or hybrid).
- Build a concise text snippet per candidate (title + a few key fields).
- Score (query, candidate_text) pairs with a cross‑encoder reranker (fine‑tuned or base).
- Sort by reranker score; optionally blend with ES _score.

Repo contents
- sample.ipynb — runnable notebook that:
  - Starts an ES Docker container locally.
  - Indexes the “calmgoose/amazon-product-data-2020” dataset into ES.
  - Implements the reranker (JinaReranker), text building utilities, and a small search UI.
- data/ — placeholder for local data.
- requirements.txt — Python dependencies.

Install
- Python 3.9+ recommended
- PowerShell
  pip install -r requirements.txt

Architecture
1) Retrieve top_k from Elasticsearch using multi_match (50–200 typical).
2) Build candidate_text per hit from key fields (e.g., Product Name, About Product, Product Specification, Technical Details).
3) Rerank with a cross‑encoder model on CPU/GPU.
4) Sort by reranker score (or blend with ES _score) and return top‑N.

Local quick start (dev)
1) Start Elasticsearch in Docker (PowerShell):
   docker run -d --name es-dev -p 9200:9200 -e "discovery.type=single-node" -e "xpack.security.enabled=false" docker.elastic.co/elasticsearch/elasticsearch:8.14.0
2) Open sample.ipynb and run cells in order:
   - It waits for ES to be ready, streams the dataset, and indexes documents via helpers.bulk.
   - Try a query in the UI cell, then run the rerank cell to see improved ordering.

Using the reranker in your code
The implementation lives inside sample.ipynb (class JinaReranker with helpers build_candidate_text and retrieve_and_rerank). For quick experimentation, copy those definitions into your script or import them after exporting to a .py file. Minimal pattern:
- Use es.search(...) to get hits
- ranked = JinaReranker().rank_hits(query, hits, top_n=20, blend_alpha=None)
- Present ranked hits to the user

Connect to a real database and index into Elasticsearch
You can index your production data into ES in several ways.
A) Python bulk indexing (flexible for any source)
- Extract rows/documents from your DB
- Transform to ES documents
- Use elasticsearch.helpers.bulk to index
Example sketch:
from elasticsearch import Elasticsearch, helpers
es = Elasticsearch("http://localhost:9200")  # use https + auth in prod
INDEX = "products"

# optional: create index with mappings/analyzers
es.indices.create(index=INDEX, ignore=400, body={
  "settings": {"analysis": {"analyzer": {"default": {"type": "standard"}}}},
  "mappings": {"properties": {"Product Name": {"type": "text"}, "Category": {"type": "keyword"}}}
})

rows = fetch_from_db()  # your code
actions = ({"_index": INDEX, "_id": r["id"], "_source": r} for r in rows)
helpers.bulk(es, actions)

B) Logstash JDBC (good for relational DBs)
- Install Logstash and a JDBC driver
- Use jdbc input to pull from MySQL/PostgreSQL/etc. and send to Elasticsearch output
Minimal logstash.conf:
input { jdbc { jdbc_driver_library => "path/to/driver.jar" jdbc_driver_class => "org.postgresql.Driver" jdbc_connection_string => "jdbc:postgresql://host:5432/db" jdbc_user => "user" jdbc_password => "pass" statement => "SELECT id, name AS \"Product Name\", description AS \"About Product\" FROM products" schedule => "*/5 * * * * *" } }
filter { }
output { elasticsearch { hosts => ["https://your-es:9200"] index => "products" user => "elastic" password => "<secret>" ssl => true cacert => "/path/ca.crt" } stdout { codec => json_lines } }

C) MongoDB and others
- Consider Monstache (MongoDB→ES), Elastic Agent, Debezium CDC, or custom ETL.

Best practices when indexing
- Use stable IDs so updates overwrite previous versions.
- Normalize and denoise text fields; ensure the fields used for search are indexed as text with the right analyzer (and add keyword subfields when exact filtering is needed).
- Consider synonyms/stemming per language; use language analyzers for multilingual.

Deploy Elasticsearch for production
Options:
- Elastic Cloud: Easiest managed option (TLS, auth, snapshots included).
- Self‑hosted Docker Compose: co‑locate with Kibana and configure security (xpack.security.enabled=true). Expose HTTPS via a reverse proxy.
- Kubernetes: Use the Elastic ECK operator to manage clusters declaratively.
Key production settings
- Security: TLS, API keys or service accounts, firewall/VPC. Don’t run with security disabled.
- Sizing: Start with 2–3 data nodes; memory ~8–32 GB/node; set JVM heap to ~50% of RAM up to 30 GB.
- Backups: Use snapshots to S3/Blob/GCS and schedule them.
- Observability: Enable slow logs and monitor with Kibana/Elastic Observability.

Deploy the reranker model for production
Pattern: run the cross‑encoder behind a stateless HTTP API and call it after ES retrieval.
Example FastAPI service (simplified):
from fastapi import FastAPI
from pydantic import BaseModel
from transformers import AutoTokenizer, AutoModelForSequenceClassification
import torch

MODEL_NAME = "jinaai/jina-reranker-v2-base-multilingual"  # replace with your fine‑tuned model path
app = FastAPI()
_tok = AutoTokenizer.from_pretrained(MODEL_NAME, trust_remote_code=True)
_model = AutoModelForSequenceClassification.from_pretrained(MODEL_NAME, trust_remote_code=True).eval().to("cuda" if torch.cuda.is_available() else "cpu")

class Item(BaseModel):
    query: str
    docs: list[str]

@app.post("/rerank")
def rerank(item: Item):
    # tokenize and score; return scores sorted desc
    # (See sample.ipynb JinaReranker for batching and FP16 usage)
    return {"scores": [0.0 for _ in item.docs]}

Containerize and run
- Build a Docker image with your service and model weights cached (use HF cache in the image layer). Use GPU images on GPU nodes if available.
- Run multiple replicas behind a load balancer; batch requests to improve throughput; enable FP16 on CUDA.
- Consider model optimizations: quantization (bitsandbytes/onnxruntime), smaller batch for low latency.

End‑to‑end flow in production
1) App issues an ES query (top_k=100) to get candidates.
2) App calls the reranker service with the query + candidate texts.
3) Sort by reranker score (or blend with ES _score) and return top‑N.
4) Cache popular queries and precompute reranks if useful.

Using your fine‑tuned model
- Fine‑tune a cross‑encoder on (query, document, label) pairs with a classification head.
- Upload the model to a private Hugging Face repo or store it internally.
- Replace the model name/path used by the service and/or notebook with your fine‑tuned artifact.
- Validate quality with held‑out relevance judgments before rollout. Consider canary release.

Configuration and security
- Use HTTPS and API keys/Service Tokens for ES (never disable security in prod).
- Configure environment variables for endpoints:
  - ES_URL, ES_API_KEY or user/password
  - INDEX name(s)
  - RERANKER_URL or model name

Troubleshooting
- OOM on model: reduce batch_size, use CPU, or move to a smaller/faster model.
- ES startup/auth errors: prefer Elastic Cloud or secure your self‑hosted cluster; check logs.
- Poor recall: widen search_fields, increase top_k, tune analyzers/synonyms.
- Poor precision: tune candidate_text construction and reranker model; try blending with ES.

Notes
- The full reference implementation (including JinaReranker, candidate text builder, and retrieve_and_rerank) is in sample.ipynb. You can copy it into a .py module for reuse in your application.


---

Preventing duplicates by fixing indexing
- We now assign a stable Elasticsearch _id for each document during indexing so that re-running the indexer overwrites existing docs instead of creating duplicates.
- ID priority (first non-empty wins): Uniq Id → UniqId → uniq_id → asin → Product Url → Model Number → Upc Ean Code → Product Name; fallback: SHA1 hash of selected fields.
- Re-index guidance:
  - If your current index already contains duplicates, delete and rebuild it to purge them (e.g., es.indices.delete(index=INDEX, ignore=[400,404]) and then run the indexing cell again).
  - Future indexing runs will overwrite documents with the same _id rather than adding new ones.
- Optional: add keyword subfields for the chosen unique identifier so you can filter or collapse by it later if needed.


---

Server-side deduplication (collapse_key; no client-side dedup)
- The notebook now prevents duplicate hits directly at the Elasticsearch layer using field collapsing on a looser grouping key.
- Each document keeps a strict uniq_key (also used as _id) for upserts, and additionally stores collapse_key (keyword) derived from a normalized Product Name (lowercased, non-alphanumeric removed, whitespace collapsed). If no title is present we fall back to uniq_key.
- All search requests use collapse: {"field": "collapse_key"} so ES returns at most one hit per normalized-title group.
- Why this matters: Near-duplicates with different URLs/IDs won't appear multiple times; re-indexing still overwrites by _id via uniq_key.
- How to apply on an existing index:
  1) If you previously indexed without collapse_key, delete the index and re-run the indexing cell to populate collapse_key (PowerShell in notebook context):
     es.indices.delete(index=INDEX, ignore=[400,404])
     # then run the indexing cell again
  2) Alternatively, reindex into a fresh index that has collapse_key mapped as keyword.
- Advanced:
  - To influence which representative doc is returned per group, set an explicit sort and/or use inner_hits with collapse to fetch more docs from a group.
  - If you have true variants that should not collapse, adjust how collapse_key is derived (e.g., include model/SKU).


---

Updated 2025-09-27

Project structure
- README.md — overview, setup, deployment, and updates (this file)
- guidelines.md — deployment, operations, and contributing guidelines
- jinaai_jina-reranker-v2-base-multilingua.md — detailed fine-tuning guidance (pointwise/pairwise, hard negatives, LoRA)
- requirements.txt — Python dependencies
- sample.ipynb — end-to-end indexing, retrieval, and reranking demo
- models\ — place local models here (e.g., models\fine_tuned_reranker)

Serbian-only fine-tuning quick guide
- Goal: specialize the cross‑encoder reranker for Serbian without changing the task.
- Start with supervised Serbian pairs (query, document, label). Prefer hard negatives from ES SERPs.
- Conservative hyperparameters for an “extra” specialization pass:
  - LR 5e-6 to 1e-5, 1–2 epochs, effective batch size 16–64, max_length 256–320, warmup ~5%.
  - Objective: num_labels=1 with BCEWithLogitsLoss (binary) or MSE (graded); or pairwise MarginRankingLoss.
- Keep tokenizer; do not change vocab. Optionally train LoRA adapters to preserve base multilingual ability.
- Full details and code snippets: see jinaai_jina-reranker-v2-base-multilingua.md

Using a locally fine‑tuned model
- Save your fine‑tuned model under: models\fine_tuned_reranker
- Replace the model name/path wherever you load the model:
  - PowerShell / Python example:
    from transformers import AutoTokenizer, AutoModelForSequenceClassification
    import torch
    MODEL_PATH = "models\\fine_tuned_reranker"
    tok = AutoTokenizer.from_pretrained(MODEL_PATH, trust_remote_code=True)
    model = AutoModelForSequenceClassification.from_pretrained(MODEL_PATH, trust_remote_code=True).eval().to("cuda" if torch.cuda.is_available() else "cpu")
- In your reranking code, pass pairs: tokenizer([query]*len(docs), docs, truncation=True, padding=True, max_length=256)

Evaluation (recommended)
- Build a fixed candidate pool per query (e.g., top 100 from ES using the same query/settings across models).
- Metrics: nDCG@10 (graded labels), MRR@10 (binary), Recall@50. Track latency for reranking.
- Validate baseline (ES only) → ES + pretrained reranker → ES + Serbian‑fine‑tuned reranker.
- Use the same pool and random seed to compare fairly; early stop on Serbian nDCG@10.

Serbian robustness tips
- Latin and Cyrillic: expose both scripts during training; optionally transliterate a fraction of samples.
- Diacritics: include with and without diacritics to match real user queries.
- Ekavian/Ijekavian variants: include both forms when relevant.
- Preserve model numbers/brands; avoid over‑normalization of alphanumerics and hyphens.

Where to find the fine‑tuning playbook
- See jinaai_jina-reranker-v2-base-multilingua.md for end‑to‑end guidance (data formats, losses, hyperparameters, hard negative mining, PEFT/LoRA setup, ES integration patterns, and evaluation).
