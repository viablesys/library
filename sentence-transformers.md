# Sentence Transformers Reference

State-of-the-art text embeddings for semantic search, classification, and similarity.
Version covered: `sentence-transformers` 3.x / 4.x (PyPI). Python 3.10+, PyTorch 1.11+.

---

## 1. Core Concept

Sentence Transformers (SBERT) fine-tune transformer models with a **siamese network** architecture
so that the cosine distance between two embeddings reflects semantic similarity. The key insight:
raw BERT embeddings averaged over tokens perform poorly for sentence-level similarity — SBERT
fixes this with contrastive training on sentence pairs.

**Output:** dense float32 vectors (embeddings). Two sentences with similar meaning cluster nearby
in embedding space regardless of exact wording. "flight cancellation" and "plane delayed" will
score ~0.8+ cosine similarity; "flight cancellation" and "quantum physics" will score ~0.05.

**Three model types in the library:**
- `SentenceTransformer` — dense embeddings (what this doc covers)
- `CrossEncoder` — reranker, takes a pair and scores it directly (slower, more accurate)
- `SparseEncoder` — SPLADE-style sparse vectors for BM25-like retrieval

---

## 2. Installation

```bash
# Minimal inference
pip install sentence-transformers

# With ONNX runtime (CPU acceleration, ~3x faster)
pip install "sentence-transformers[onnx]"

# With ONNX + GPU support
pip install "sentence-transformers[onnx-gpu]"

# With OpenVINO (Intel acceleration)
pip install "sentence-transformers[openvino]"

# Training / fine-tuning
pip install "sentence-transformers[train]"
```

---

## 3. Basic Usage

### Encode sentences

```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer("all-MiniLM-L6-v2")

sentences = [
    "user complaint: app keeps crashing",
    "feature request: add dark mode",
    "billing issue: wrong charge on invoice",
]

# Returns numpy array shape [N, 384]
embeddings = model.encode(sentences)
print(embeddings.shape)  # (3, 384)
```

### Key `encode()` parameters

```python
embeddings = model.encode(
    sentences,
    batch_size=32,               # default; increase for throughput on GPU
    show_progress_bar=False,
    convert_to_numpy=True,       # default; False gives torch tensors
    convert_to_tensor=False,     # overrides convert_to_numpy; returns single tensor
    normalize_embeddings=False,  # True → unit-length vectors (enables dot product = cosine)
    precision="float32",         # "int8" | "uint8" | "binary" | "ubinary" for quantized output
    device=None,                 # "cuda", "cpu", or list for multi-GPU
    truncate_dim=None,           # truncate to smaller dim (Matryoshka models only)
)
```

**Always set `normalize_embeddings=True`** when using cosine similarity or dot product for search.
Unnormalized magnitudes skew scores — longer sentences can appear artificially more/less similar.

---

## 4. Similarity Computation

### `model.similarity()` — recommended

```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer("all-MiniLM-L6-v2")

sentences_a = ["The cat sat on the mat", "Every dog has its day"]
sentences_b = ["A feline rested on the rug", "The weather is nice today"]

emb_a = model.encode(sentences_a, convert_to_tensor=True)
emb_b = model.encode(sentences_b, convert_to_tensor=True)

# Returns tensor shape [len(a), len(b)]
scores = model.similarity(emb_a, emb_b)
# tensor([[0.8012, 0.0341],
#         [0.1209, 0.0814]])
```

### `util.cos_sim()` / `util.dot_score()`

```python
from sentence_transformers import util

# Cosine similarity — works on unnormalized embeddings
cos_scores = util.cos_sim(emb_a, emb_b)

# Dot product — equivalent to cosine when embeddings are normalized; faster
dot_scores = util.dot_score(emb_a, emb_b)
```

### Pairwise (one-to-one, not cross-product)

```python
# similarity_pairwise: scores[i] = similarity(a[i], b[i])
pairwise = model.similarity_pairwise(emb_a, emb_b)
```

---

## 5. Semantic Search (Top-K Retrieval)

### Built-in `semantic_search()`

For corpora up to ~1M entries, the built-in utility is sufficient:

```python
from sentence_transformers import SentenceTransformer, util

model = SentenceTransformer("all-MiniLM-L6-v2")

corpus = [
    "user complaint: login fails on mobile",
    "feature request: export to CSV",
    "billing issue: double charged this month",
    "user complaint: slow search results",
    "feature request: keyboard shortcuts",
]
corpus_embeddings = model.encode(corpus, convert_to_tensor=True, normalize_embeddings=True)

query = "app is too slow"
query_embedding = model.encode(query, convert_to_tensor=True, normalize_embeddings=True)

hits = util.semantic_search(
    query_embedding,
    corpus_embeddings,
    top_k=3,
    score_function=util.dot_score,  # use dot_score when normalized
)
# hits[0] → list of {"corpus_id": int, "score": float}
for hit in hits[0]:
    print(f"{hit['score']:.4f}  {corpus[hit['corpus_id']]}")
```

### FAISS for large corpora (>100k entries)

```python
import faiss
import numpy as np
from sentence_transformers import SentenceTransformer

model = SentenceTransformer("all-MiniLM-L6-v2")

# Encode corpus — must be float32 and normalized for IndexFlatIP (cosine)
corpus_embeddings = model.encode(corpus, normalize_embeddings=True).astype(np.float32)

dim = corpus_embeddings.shape[1]  # 384 for MiniLM

# IndexFlatIP = exact inner product search (== cosine when normalized)
index = faiss.IndexFlatIP(dim)
index.add(corpus_embeddings)

# Query
query_emb = model.encode([query], normalize_embeddings=True).astype(np.float32)
scores, indices = index.search(query_emb, k=5)

for score, idx in zip(scores[0], indices[0]):
    print(f"{score:.4f}  {corpus[idx]}")
```

**FAISS index types by scale:**

| Index | Exact? | Scale | Notes |
|-------|--------|-------|-------|
| `IndexFlatIP` | Yes | <500k | Brute-force cosine; best accuracy |
| `IndexFlatL2` | Yes | <500k | Euclidean; normalize first for cosine |
| `IndexIVFFlat` | No | 1M-50M | Cluster-based ANN; tune `nlist` |
| `IndexHNSWFlat` | No | >1M | Graph-based; best speed/accuracy trade-off |

---

## 6. Classification with Embeddings

Embeddings are a powerful feature representation for sklearn classifiers. Two patterns:

### Pattern A: Embed then classify (zero/few-shot style)

Use label embeddings as anchors; classify by nearest-label similarity.

```python
from sentence_transformers import SentenceTransformer, util
import numpy as np

model = SentenceTransformer("all-MiniLM-L6-v2")

# Label descriptions — more descriptive = better signal
labels = {
    "billing":   "billing issue, wrong charge, invoice, payment",
    "bug":       "crash, error, broken, not working, app fails",
    "feature":   "request, add, would be nice, improve, suggestion",
    "account":   "login, password, cannot access, locked out",
}

label_names = list(labels.keys())
label_embeddings = model.encode(
    list(labels.values()),
    normalize_embeddings=True,
    convert_to_tensor=True,
)

def classify(text: str) -> str:
    emb = model.encode(text, normalize_embeddings=True, convert_to_tensor=True)
    scores = util.dot_score(emb, label_embeddings)[0]
    return label_names[scores.argmax().item()]

print(classify("can't log in to my account"))   # → account
print(classify("app crashes when I open it"))    # → bug
```

### Pattern B: Embed then fit sklearn classifier

For labeled data (even small amounts), fit a logistic regression on embeddings:

```python
from sentence_transformers import SentenceTransformer
from sklearn.linear_model import LogisticRegression
from sklearn.preprocessing import normalize
import numpy as np

model = SentenceTransformer("all-MiniLM-L6-v2")

# Training data — even 10-50 examples per class gives strong results
train_texts = [
    "can't log in", "password reset broken", "locked out of my account",
    "app crashes on startup", "gets an error when saving", "broken feature",
    "please add dark mode", "would love export to PDF", "could you add search?",
]
train_labels = ["account", "account", "account", "bug", "bug", "bug", "feature", "feature", "feature"]

# Encode — normalize so logistic regression sees unit vectors
train_emb = model.encode(train_texts, normalize_embeddings=True)

clf = LogisticRegression(max_iter=500, C=1.0)
clf.fit(train_emb, train_labels)

# Inference
test_texts = ["I keep getting 404 errors", "add keyboard shortcuts please"]
test_emb = model.encode(test_texts, normalize_embeddings=True)
predictions = clf.predict(test_emb)
probas = clf.predict_proba(test_emb)

print(predictions)    # ['bug', 'feature']
print(probas.max(axis=1))  # confidence per prediction
```

**Other sklearn classifiers that work well on embeddings:**

```python
from sklearn.svm import SVC, LinearSVC
from sklearn.neighbors import KNeighborsClassifier
from sklearn.neural_network import MLPClassifier

# LinearSVC — fast, good on linearly separable embeddings
clf = LinearSVC(C=1.0, max_iter=2000)

# KNN — no training, good for prototyping
clf = KNeighborsClassifier(n_neighbors=5, metric="cosine")

# MLP — most flexible; use when class boundaries are nonlinear
clf = MLPClassifier(hidden_layer_sizes=(256,), max_iter=500)
```

---

## 7. SetFit: Few-Shot Fine-Tuning

SetFit (HuggingFace) combines sentence-transformers with contrastive fine-tuning and a sklearn
head. Achieves high accuracy from 8-32 labeled examples per class.

```bash
pip install setfit
```

### Training pipeline

```python
from datasets import Dataset
from setfit import SetFitModel, Trainer, TrainingArguments

# Build a tiny labeled dataset (8 examples per class)
train_data = Dataset.from_dict({
    "text": [
        "app won't start", "keeps crashing", "runtime error on open",
        "add CSV export", "dark mode please", "need bulk actions",
        "charged twice", "wrong amount billed", "invoice is wrong",
    ],
    "label": [0, 0, 0, 1, 1, 1, 2, 2, 2],
})

# Default head: LogisticRegression from sklearn
model = SetFitModel.from_pretrained(
    "sentence-transformers/paraphrase-mpnet-base-v2",
    labels=["bug", "feature", "billing"],
)

args = TrainingArguments(
    num_epochs=1,               # contrastive fine-tuning epochs
    batch_size=16,
    num_iterations=20,          # number of sentence pair iterations
    body_learning_rate=1e-5,
    head_learning_rate=1e-2,
)

trainer = Trainer(
    model=model,
    args=args,
    train_dataset=train_data,
)
trainer.train()

# Inference
preds = model(["can't log in", "please add search"])
print(preds)  # tensor([0, 1]) — bug, feature
```

### Custom sklearn head

```python
from setfit import SetFitModel
from sklearn.svm import LinearSVC
from sentence_transformers import SentenceTransformer

body = SentenceTransformer("BAAI/bge-small-en-v1.5")
head = LinearSVC(C=0.5)
model = SetFitModel(body, head)
```

### Custom LogisticRegression parameters

```python
model = SetFitModel.from_pretrained(
    "BAAI/bge-small-en-v1.5",
    head_params={"solver": "liblinear", "max_iter": 500, "C": 0.5},
)
```

**SetFit vs raw embedding + sklearn:**

| | SetFit | Raw embed + sklearn |
|---|---|---|
| Data needed | 8-32 per class | 20-100+ per class |
| Training time | Minutes | Seconds |
| Accuracy (few-shot) | Higher | Lower |
| No fine-tuning needed | No | Yes |
| Extra dependency | `setfit` | None |

---

## 8. Model Comparison

### General-purpose models (English)

| Model | Dims | Params | Disk | Max tokens | Speed (CPU) | Quality |
|-------|------|--------|------|------------|-------------|---------|
| `all-MiniLM-L6-v2` | 384 | ~22M | ~90MB | 256 | Very fast | Good |
| `all-MiniLM-L12-v2` | 384 | ~33M | ~120MB | 256 | Fast | Better |
| `all-mpnet-base-v2` | 768 | ~110M | ~420MB | 384 | Moderate | Best |
| `paraphrase-MiniLM-L6-v2` | 384 | ~22M | ~90MB | 128 | Very fast | Good |
| `paraphrase-mpnet-base-v2` | 768 | ~110M | ~420MB | 128 | Moderate | Best |

### Modern/SOTA models

| Model | Dims | Notes |
|-------|------|-------|
| `BAAI/bge-small-en-v1.5` | 384 | Fast, MTEB-competitive; good MiniLM replacement |
| `BAAI/bge-base-en-v1.5` | 768 | Strong accuracy, MTEB top tier for base size |
| `BAAI/bge-large-en-v1.5` | 1024 | Best accuracy at large size |
| `BAAI/bge-m3` | 1024 | Multi-lingual (100+ langs), dense+sparse, 8192 tokens |
| `nomic-ai/nomic-embed-text-v1.5` | 768 | Matryoshka, truncatable, 8192 tokens |
| `mixedbread-ai/mxbai-embed-large-v1` | 1024 | MTEB leaderboard top, 512 tokens |

### Multilingual models

| Model | Dims | Languages | Max tokens |
|-------|------|-----------|------------|
| `paraphrase-multilingual-MiniLM-L12-v2` | 384 | 50+ | 128 |
| `paraphrase-multilingual-mpnet-base-v2` | 768 | 50+ | 128 |
| `BAAI/bge-m3` | 1024 | 100+ | 8192 |

### Choosing a model for short text (20-200 chars)

Short text fits comfortably in any model's token budget. The differentiation is quality vs speed:

- **Prototype / high-throughput**: `all-MiniLM-L6-v2` — 5x faster than mpnet, 384-dim, good enough
- **Production accuracy**: `BAAI/bge-small-en-v1.5` — similar speed to MiniLM, better MTEB scores
- **Best accuracy, cost flexible**: `all-mpnet-base-v2` or `BAAI/bge-base-en-v1.5`
- **Multilingual**: `paraphrase-multilingual-MiniLM-L12-v2` — max 128 tokens but short text is fine

---

## 9. Efficiency: GPU, ONNX, Quantization

### GPU

```python
import torch
from sentence_transformers import SentenceTransformer

# Auto-detects CUDA if available
model = SentenceTransformer("all-MiniLM-L6-v2")

# Explicit device
model = SentenceTransformer("all-MiniLM-L6-v2", device="cuda")
model = SentenceTransformer("all-MiniLM-L6-v2", device="cpu")

# Encode on GPU — increase batch_size for throughput
embeddings = model.encode(sentences, batch_size=128, device="cuda")
```

### Multi-GPU encoding (large corpora)

```python
from sentence_transformers import SentenceTransformer

def encode_large_corpus(model_name, sentences):
    model = SentenceTransformer(model_name)

    # Option A: pass device list directly (simpler, for one encode call)
    embeddings = model.encode(sentences, device=["cuda:0", "cuda:1"])
    return embeddings

def encode_with_pool(model_name, sentences):
    model = SentenceTransformer(model_name)

    # Option B: reusable pool (more efficient for repeated encodes)
    pool = model.start_multi_process_pool(target_devices=["cuda:0", "cuda:1"])
    embeddings = model.encode(sentences, pool=pool)
    model.stop_multi_process_pool(pool)
    return embeddings

if __name__ == "__main__":  # required for multiprocessing
    result = encode_with_pool("all-MiniLM-L6-v2", sentences)
```

### ONNX backend (CPU: ~3x speedup, GPU: ~1.9x speedup)

```python
# Load with ONNX backend — drop-in replacement, same API
model = SentenceTransformer("all-MiniLM-L6-v2", backend="onnx")

# Optimize ONNX model (requires optimum)
from sentence_transformers import export_optimized_onnx_model

export_optimized_onnx_model(
    model=model,
    optimization_config="O3",   # O1..O4 from HuggingFace Optimum; O3 = aggressive
    model_name_or_path="./optimized-model",
)
# Load the optimized version
model = SentenceTransformer("./optimized-model", backend="onnx")
```

### Dynamic quantization (CPU int8, ~2x speedup)

```python
from sentence_transformers import SentenceTransformer, export_dynamic_quantized_onnx_model

# Must load with ONNX backend first
model = SentenceTransformer("all-MiniLM-L6-v2", backend="onnx")

export_dynamic_quantized_onnx_model(
    model=model,
    quantization_config="avx512_vnni",  # options: "arm64", "avx2", "avx512", "avx512_vnni"
    model_name_or_path="./quantized-model",
)

# Load quantized model
model_q = SentenceTransformer("./quantized-model", backend="onnx")
```

**Quantization config by CPU:**

| Config | CPU Architecture |
|--------|-----------------|
| `arm64` | Apple Silicon, ARM servers |
| `avx2` | Intel Haswell+ / AMD Zen+ |
| `avx512` | Intel Skylake-X+ |
| `avx512_vnni` | Intel Ice Lake+ (best for vnni-capable chips) |

Check your CPU: `lscpu | grep -o 'avx[0-9_]*'`

### Precision parameter (embedding quantization)

```python
# Output quantized embeddings — smaller storage, faster similarity search
embeddings_int8 = model.encode(sentences, precision="int8")     # 4x smaller
embeddings_binary = model.encode(sentences, precision="binary")  # 32x smaller, Hamming distance

# Note: model.similarity() only supports fp32; use raw numpy for quantized
```

### Speed summary

| Setup | Relative speed | Notes |
|-------|---------------|-------|
| CPU fp32 (baseline) | 1x | PyTorch default |
| CPU ONNX fp32 | ~1.5x | Free accuracy gain |
| CPU ONNX int8 quantized | ~2-3x | Minimal accuracy loss |
| GPU fp32 | ~5-20x | Depends on GPU/model |
| GPU fp16 | ~1.9x vs GPU fp32 | `model.half()` |
| GPU bf16 | ~1.8x vs GPU fp32 | A100/H100 |
| Multi-GPU | Linear with GPU count | Pool required |

---

## 10. Matryoshka Embeddings (Truncatable Dimensions)

Some models (nomic-embed, bge with MRL training) produce embeddings where truncating to a smaller
dimension still gives useful results. Enables speed/accuracy trade-offs at inference time.

```python
from sentence_transformers import SentenceTransformer

# Load with truncation — only 64 dims computed
model = SentenceTransformer(
    "nomic-ai/nomic-embed-text-v1.5",
    trust_remote_code=True,
    truncate_dim=64,
)

embeddings = model.encode(sentences)
assert embeddings.shape[-1] == 64
```

**Matryoshka dimension trade-off (nomic-embed-text-v1.5, STSBenchmark):**

| Dims | Spearman score | vs full |
|------|---------------|---------|
| 768 | 0.856 | 100% |
| 256 | 0.849 | 99.2% |
| 64  | 0.832 | 97.2% |
| 16  | 0.791 | 92.4% |

Use 64-128 dims for fast retrieval, full dims for final reranking.

---

## 11. Training / Fine-tuning

### InputExample format

```python
from sentence_transformers import InputExample

# Pair with similarity score (0.0-1.0) — for CosineSimilarityLoss, CoSENTLoss
example = InputExample(texts=["sentence A", "sentence B"], label=0.9)

# Pair with binary label — for ContrastiveLoss, SoftmaxLoss
example = InputExample(texts=["same topic", "same topic"], label=1)

# Triplet — for TripletLoss
example = InputExample(texts=["anchor", "positive", "negative"])
```

### Common loss functions

| Loss | Input | Use Case |
|------|-------|----------|
| `CosineSimilarityLoss` | (sent_a, sent_b, score) | STS scores (0-1) |
| `MultipleNegativesRankingLoss` | (anchor, positive) | Retrieval, NLI pairs |
| `ContrastiveLoss` | (sent_a, sent_b, label) | Binary similar/dissimilar |
| `TripletLoss` | (anchor, pos, neg) | Triplet data |
| `CoSENTLoss` | (sent_a, sent_b, score) | Better STS than cosine loss |
| `MatryoshkaLoss` | (wraps other loss) | Train truncatable models |

### Minimal fine-tuning example

```python
from sentence_transformers import SentenceTransformer, SentenceTransformerTrainer, losses
from datasets import Dataset

model = SentenceTransformer("all-MiniLM-L6-v2")

train_dataset = Dataset.from_dict({
    "anchor": ["ticket: app crashes", "billing: double charge"],
    "positive": ["the application fails to start", "I was charged twice"],
})

loss = losses.MultipleNegativesRankingLoss(model)

trainer = SentenceTransformerTrainer(
    model=model,
    train_dataset=train_dataset,
    loss=loss,
)
trainer.train()
model.save_pretrained("./my-finetuned-model")
```

---

## 12. Complete Classification Pipeline

Full pipeline: embed → cache corpus embeddings → classify new inputs.

```python
import numpy as np
from pathlib import Path
from sentence_transformers import SentenceTransformer, util
from sklearn.linear_model import LogisticRegression
import pickle

# --- Setup ---
MODEL_NAME = "all-MiniLM-L6-v2"
model = SentenceTransformer(MODEL_NAME)

# --- Training ---
train_texts = [...]   # list of strings
train_labels = [...]  # list of string or int labels

train_emb = model.encode(
    train_texts,
    normalize_embeddings=True,
    batch_size=64,
    show_progress_bar=True,
)

clf = LogisticRegression(max_iter=500, C=1.0, solver="lbfgs", multi_class="multinomial")
clf.fit(train_emb, train_labels)

# --- Save ---
with open("classifier.pkl", "wb") as f:
    pickle.dump(clf, f)

# --- Inference ---
def predict(texts: list[str]) -> list[str]:
    emb = model.encode(texts, normalize_embeddings=True)
    return clf.predict(emb).tolist()

def predict_with_confidence(texts: list[str]) -> list[tuple[str, float]]:
    emb = model.encode(texts, normalize_embeddings=True)
    labels = clf.predict(emb)
    confidences = clf.predict_proba(emb).max(axis=1)
    return list(zip(labels, confidences))
```

---

## 13. Pitfalls

### Silent truncation

Every model has a `max_seq_length`. Inputs longer than this are silently cut. Two documents that
differ only beyond the cutoff will produce identical embeddings.

```python
# Check the limit
print(model.max_seq_length)  # e.g. 256 for all-MiniLM-L6-v2, 384 for all-mpnet-base-v2

# If you might exceed it, chunk:
def chunk_encode(model, text, max_chars=500):
    chunks = [text[i:i+max_chars] for i in range(0, len(text), max_chars)]
    embeddings = model.encode(chunks, normalize_embeddings=True)
    return embeddings.mean(axis=0)  # average chunk embeddings
```

### Not normalizing before cosine similarity

```python
# Wrong — dot product of unnormalized vectors != cosine similarity
emb = model.encode(texts)                          # magnitudes vary
scores = emb @ emb.T                               # biased by magnitude

# Correct
emb = model.encode(texts, normalize_embeddings=True)
scores = emb @ emb.T                               # now == cosine similarity
```

### Mismatched normalization between corpus and query

```python
# Both must be normalized or neither — never mix
corpus_emb = model.encode(corpus, normalize_embeddings=True)   # normalized
query_emb  = model.encode(query)                               # NOT normalized — broken

# Fix: normalize both
query_emb  = model.encode(query, normalize_embeddings=True)
```

### Wrong model for asymmetric tasks

Some models are trained for **symmetric** similarity (both texts are the same "type") and others
for **asymmetric** (short query vs. long document). Using the wrong type hurts retrieval quality.

```python
# Symmetric: both sides are same type (e.g., FAQ matching, paraphrase detection)
model = SentenceTransformer("all-MiniLM-L6-v2")

# Asymmetric: short query → long document (RAG, passage retrieval)
model = SentenceTransformer("msmarco-distilbert-base-v4")  # trained on MS MARCO
model = SentenceTransformer("BAAI/bge-base-en-v1.5")       # also good for retrieval

# BGE models use a query prefix for asymmetric tasks
query_embedding = model.encode("Represent this query for retrieval: " + query)
```

### Cosine similarity thresholds are domain-specific

A score of 0.7 means very different things across domains. Calibrate on labeled data:

```python
from sklearn.metrics import precision_recall_curve

# Find the threshold that maximizes F1 on a held-out set
scores = [util.cos_sim(q_emb, d_emb).item() for q_emb, d_emb in pairs]
labels = [1, 0, 1, ...]   # ground truth

precision, recall, thresholds = precision_recall_curve(labels, scores)
f1 = 2 * precision * recall / (precision + recall + 1e-8)
best_threshold = thresholds[f1.argmax()]
```

### Embedding dilution in large chunks

Research shows 256-512 tokens is the sweet spot. A 4000-token chunk produces an embedding that
averages so many concepts the cosine similarity to any specific query is diluted. For short text
(20-200 chars) this is not an issue — no chunking needed.

### Model loaded on every call (serverless / hook contexts)

Loading a model takes 0.5-3s. In hot paths or per-request contexts, load once and reuse:

```python
# Module-level singleton
_model = None

def get_model():
    global _model
    if _model is None:
        _model = SentenceTransformer("all-MiniLM-L6-v2")
    return _model
```

### Multiprocessing and `start_multi_process_pool`

Must run under `if __name__ == "__main__":` guard. Omitting this causes fork bombs on Linux
and errors on Windows/macOS.

---

## 14. Similarity Score Interpretation

Scores depend heavily on the model and domain. Rough guides for general-purpose models:

| Score | Interpretation |
|-------|---------------|
| > 0.90 | Near-duplicate or paraphrase |
| 0.70-0.90 | Strongly related / same topic |
| 0.50-0.70 | Related but distinct |
| 0.30-0.50 | Weakly related |
| < 0.30 | Unrelated |

These are rough ranges. Calibrate with your own data before using fixed thresholds in production.

---

## 15. Quick Reference

```python
# Load
from sentence_transformers import SentenceTransformer, util
model = SentenceTransformer("all-MiniLM-L6-v2")

# Encode
emb = model.encode(texts, normalize_embeddings=True)

# Similarity matrix
scores = model.similarity(emb_a, emb_b)           # [N, M] tensor
scores = util.cos_sim(emb_a, emb_b)               # same

# Top-K search
hits = util.semantic_search(query_emb, corpus_emb, top_k=10)

# Check token limit
print(model.max_seq_length)

# ONNX backend (drop-in)
model = SentenceTransformer("all-MiniLM-L6-v2", backend="onnx")

# Multi-GPU
embeddings = model.encode(texts, device=["cuda:0", "cuda:1"])

# Matryoshka truncation
model = SentenceTransformer("nomic-ai/nomic-embed-text-v1.5",
                             trust_remote_code=True, truncate_dim=128)
```

---

## References

- [SBERT documentation](https://www.sbert.net/)
- [Pretrained models list](https://www.sbert.net/docs/sentence_transformer/pretrained_models.html)
- [Efficiency / ONNX docs](https://sbert.net/docs/sentence_transformer/usage/efficiency.html)
- [SetFit: few-shot fine-tuning](https://huggingface.co/blog/setfit)
- [MTEB leaderboard](https://huggingface.co/spaces/mteb/leaderboard)
- [GitHub: huggingface/sentence-transformers](https://github.com/huggingface/sentence-transformers)
