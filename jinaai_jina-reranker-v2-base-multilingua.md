### What to fine-tune and why
- jinaai/jina-reranker-v2-base-multilingual is a cross-encoder: it takes a pair (query, document) and outputs a relevance score. This is ideal for re-ranking the top N results from Elasticsearch.
- For Serbian product search, you’ll get the biggest gains by fine-tuning on in-domain Serbian data with hard negatives from your ES index.

### Data you need
Prepare one of the following supervision formats (you can use multiple):

1) Pointwise (binary or graded) relevance
- Each sample: {query, document, label}
- label ∈ {0/1} or graded like {0,1,2,3} (e.g., 0=bad, 3=perfect).
- Pros: easy to gather; can use clicks (implicit) or editorial judgments.

2) Pairwise (preference) with hard negatives
- Each sample: {query, pos_doc, neg_doc}
- Train with margin ranking (encourage score(q,pos) > score(q,neg) by a margin).
- Pros: directly optimizes ranking order; works well with mined negatives from ES.

3) Listwise (optional if you have full ranked lists)
- Full lists per query with graded labels; train with approximate listwise losses (e.g., nDCG loss). More involved.

Start with pointwise or pairwise. If you have click logs:
- Derive positives from clicked purchases or long dwell-time clicks.
- Sample negatives from unclicked results in the same impression (hard negatives) plus BM25 random negatives.

### Text fields to use for Serbian products
- Query: the user’s Serbian query text.
- Document: concatenate product_title, important attributes (brand, model), and a short normalized description. Keep max 256–320 wordpiece tokens.
- Normalize Serbian: lowercase consistently, unify diacritics usage, remove boilerplate, preserve key tokens (sizes, model numbers).

### Tokenization and input formatting
- Cross-encoders typically use: "[CLS] {query} [SEP] {document} [SEP]".
- With Hugging Face AutoTokenizer, pass a pair: tokenizer(query, document, truncation=True, max_length=256).
- Set truncation strategy to favor the document (often longer) but ensure the whole query fits.

### Objectives and heads
- Pointwise
  - Use AutoModelForSequenceClassification with num_labels=1 (a single regression/logit head).
  - Loss: BCEWithLogitsLoss for binary labels, or MSE for graded labels (normalize labels to 0–1 or 0–3 scale).
- Pairwise
  - Forward both pairs (q,pos) and (q,neg); compute scores s_pos, s_neg.
  - MarginRankingLoss (e.g., margin=0.1): loss = max(0, margin − (s_pos − s_neg)).

### Hyperparameters that work well
- Learning rate: 1e-5 to 2e-5 for full fine-tuning; 2e-4 to 5e-4 for LoRA adapters.
- Batch size: as large as fits (effective 16–64 with gradient accumulation). For pairwise, each step carries two forward passes.
- Epochs: 1–3 often sufficient for in-domain; watch validation nDCG@10.
- Max length: 256–320 for speed; only go to 512 if needed.
- Warmup: 5–10% of total steps; cosine or linear decay.
- Regularization: weight decay 0.01–0.1; early stopping by nDCG@10.

### Hard negative mining loop
1) Start with BM25 (or your current ES) to get top 50–200 per query.
2) Label positives vs negatives (implicit or editorial). Sample the top non-clicked/high-score items as hard negatives.
3) Train a first pass.
4) Use the trained reranker to re-score candidates; collect new errors (false positives) as harder negatives.
5) Retrain with a mix of old and new hard negatives (curriculum).

### Measuring progress
- Offline metrics on a held-out Serbian set: MRR@10, nDCG@10, Recall@50.
- Use consistent candidate pools (same ES query, top 100) across models.
- For graded labels, nDCG@k is most informative.

### Efficient fine-tuning (resource-friendly)
- Use PEFT/LoRA or QLoRA:
  - Quantize base to 4/8-bit, train low-rank adapters on attention and maybe MLPs.
  - Great when VRAM is limited, often matches full FT performance for reranking.
- Mixed precision (fp16/bf16) if supported.
- Gradient checkpointing to save memory.

### Example: Pointwise training (Hugging Face, PyTorch)
Below shows the critical differences from generic classification: we pass (query, doc) pairs and use a single regression/logit.

```python
from transformers import AutoTokenizer, AutoModelForSequenceClassification
import torch
from torch.utils.data import Dataset, DataLoader
from torch.nn import BCEWithLogitsLoss

model_name = "jinaai/jina-reranker-v2-base-multilingual"
tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForSequenceClassification.from_pretrained(model_name, num_labels=1)

class RerankDS(Dataset):
    def __init__(self, pairs):
        # pairs: list of dicts with keys: query, doc, label (0/1 or float)
        self.pairs = pairs
    def __len__(self):
        return len(self.pairs)
    def __getitem__(self, i):
        ex = self.pairs[i]
        enc = tokenizer(ex["query"], ex["doc"], truncation=True, padding="max_length", max_length=256, return_tensors="pt")
        item = {k: v.squeeze(0) for k,v in enc.items()}
        item["labels"] = torch.tensor([ex["label"]], dtype=torch.float)
        return item

train_loader = DataLoader(RerankDS(train_pairs), batch_size=16, shuffle=True)
val_loader = DataLoader(RerankDS(val_pairs), batch_size=32)

device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
model.to(device)
optimizer = torch.optim.AdamW(model.parameters(), lr=1e-5, weight_decay=0.01)
criterion = BCEWithLogitsLoss()

for epoch in range(2):
    model.train()
    for batch in train_loader:
        batch = {k: v.to(device) for k, v in batch.items()}
        labels = batch.pop("labels")
        out = model(**batch)
        loss = criterion(out.logits.view(-1), labels.view(-1))
        loss.backward()
        optimizer.step(); optimizer.zero_grad()

    # validation: compute scores then nDCG/MRR on your candidate pools
    model.eval()
    # ...

model.save_pretrained("models\\fine_tuned_reranker")
```

Key contrasts with a generic classifier:
- num_labels=1 (not multi-class)
- Input is query-document pair, not a single text string
- Loss is ranking-appropriate (BCE or MSE for pointwise)

### Example: Pairwise training with MarginRankingLoss
```python
from torch.nn import MarginRankingLoss

margin_loss = MarginRankingLoss(margin=0.1)

def score_batch(model, tokenizer, queries, docs):
    enc = tokenizer(queries, docs, truncation=True, padding=True, max_length=256, return_tensors="pt").to(device)
    with torch.cuda.amp.autocast(enabled=torch.cuda.is_available()):
        out = model(**enc)
    return out.logits.view(-1)  # higher is better

for epoch in range(2):
    model.train()
    for batch in pairwise_loader:  # each batch contains lists: q, pos, neg
        s_pos = score_batch(model, tokenizer, batch["q"], batch["pos"])
        s_neg = score_batch(model, tokenizer, batch["q"], batch["neg"])
        y = torch.ones_like(s_pos, device=device)
        loss = margin_loss(s_pos, s_neg, y)
        loss.backward(); optimizer.step(); optimizer.zero_grad()
```

### Using PEFT/LoRA (optional but recommended)
```python
from peft import LoraConfig, get_peft_model

peft_cfg = LoraConfig(
    r=16, lora_alpha=32, lora_dropout=0.05,
    target_modules=["query", "key", "value", "dense"],  # depends on the backbone
    bias="none",
)
model = get_peft_model(model, peft_cfg)
```
Train as usual; save adapters for lightweight deployment.

### Candidate generation from Elasticsearch
- Use Elasticsearch to retrieve top N (e.g., 100) candidates with BM25 or a hybrid (BM25 + dense, RRF).
- Rerank client-side with the fine-tuned cross-encoder and return top k (e.g., 10).
- Keep N modest (50–200) to control latency.

Python sketch:
```python
from elasticsearch import Elasticsearch
from transformers import AutoTokenizer, AutoModelForSequenceClassification
import torch

es = Elasticsearch(["http://localhost:9200"])  # auth as needed
index = "products"

model = AutoModelForSequenceClassification.from_pretrained("models\\fine_tuned_reranker").eval().to("cuda")
tokenizer = AutoTokenizer.from_pretrained("models\\fine_tuned_reranker")

def rerank(query, hits):
    docs = [h["_source"]["title"] + " " + h["_source"].get("attrs", "") for h in hits]
    enc = tokenizer([query]*len(docs), docs, truncation=True, padding=True, max_length=256, return_tensors="pt").to(model.device)
    with torch.no_grad():
        scores = model(**enc).logits.view(-1).cpu().tolist()
    for h, s in zip(hits, scores):
        h["rerank_score"] = float(s)
    return sorted(hits, key=lambda x: x["rerank_score"], reverse=True)

def search_and_rerank(query):
    res = es.search(index=index, size=100, query={"multi_match": {"query": query, "fields": ["title^3", "description"]}})
    hits = res["hits"]["hits"]
    return rerank(query, hits)[:10]
```

Tip: If latency matters, batch requests, use fp16/bf16, and serve with ONNX/TensorRT where possible.

### Evaluation pipeline
- Build a fixed benchmark of queries with labeled candidates (top 100 from ES), in Serbian.
- Evaluate baseline (ES-only), then ES+pretrained reranker, then ES+fine-tuned reranker.
- Report nDCG@10, MRR@10, Recall@50, average latency.
- Do ablations: only title vs title+attrs vs title+desc; max_length 256 vs 320.

### Data quality and Serbian-specific tips
- Normalize brand and model tokens; avoid splitting model numbers.
- Maintain diacritics consistently; if user queries often omit them, include both forms during training for robustness.
- Add synonyms and morphology variants in your negative/positive pools to reduce overfitting to exact token matches.
- Balance categories so popularity doesn’t dominate (downsample overrepresented classes).

### Practical checklist
- [ ] Prepare query-document pairs in Serbian with labels; mix easy and hard negatives.
- [ ] Choose objective: start with pointwise BCE (binary) or pairwise margin ranking.
- [ ] Tokenize as pairs; set num_labels=1.
- [ ] Tune LR 1e-5–2e-5 (full FT) or LoRA with higher LR; 1–3 epochs; max_length 256–320.
- [ ] Validate on held-out queries, track nDCG@10; early stop.
- [ ] Integrate by reranking top 100 ES hits client-side.
- [ ] Iterate hard-negative mining and refresh the dataset.

### If you’re adapting your current notebook
From your snippet, it looks like you’re training a multi-class classifier on single texts. For reranking:
- Switch the dataset to produce tokenizer(query, doc, ...), not a single string.
- Set num_labels=1.
- Replace the loss with BCEWithLogitsLoss (binary) or implement pairwise margin ranking.
- Compute evaluation metrics on candidate lists (MRR/nDCG), not accuracy.

If you share a small sample of your Serbian data schema, I can tailor an exact collator and training loop for you (pointwise or pairwise), and help set up the ES reranking endpoint with batching and latency considerations.