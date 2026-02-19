# SetFit Reference

Efficient few-shot fine-tuning for text classification using Sentence Transformers. Current version: **1.1.x**.

Docs: <https://huggingface.co/docs/setfit>
GitHub: <https://github.com/huggingface/setfit>
Paper: <https://arxiv.org/abs/2209.11055>

---

## Table of Contents

1. [How SetFit Works](#1-how-setfit-works)
2. [Installation](#2-installation)
3. [Model Selection](#3-model-selection)
4. [Training API](#4-training-api)
5. [Inference](#5-inference)
6. [Dataset Preparation](#6-dataset-preparation)
7. [Multi-label Classification](#7-multi-label-classification)
8. [Zero-shot Mode](#8-zero-shot-mode)
9. [Hyperparameter Tuning](#9-hyperparameter-tuning)
10. [Knowledge Distillation](#10-knowledge-distillation)
11. [Performance Characteristics](#11-performance-characteristics)
12. [Comparison with Alternatives](#12-comparison-with-alternatives)
13. [Pitfalls](#13-pitfalls)

---

## 1. How SetFit Works

SetFit trains in two separate phases with no prompts or verbalizers.

**Phase 1 — Contrastive embedding fine-tuning**

Given labeled examples, SetFit generates sentence pairs: positive pairs (same class) and negative pairs (different classes). With N examples per class and C classes, the pair count grows combinatorially — 8 examples across 2 classes yields 28 positive + 64 negative = 92 pairs. This is what makes few-shot learning viable: a small labeled set explodes into a large contrastive training set.

The Sentence Transformer body is fine-tuned on these pairs using a contrastive loss (default: cosine similarity loss), so that same-class texts cluster in embedding space and different-class texts repel.

**Phase 2 — Classification head training**

After embedding fine-tuning, the frozen body encodes all labeled examples. A lightweight classifier (default: logistic regression from scikit-learn) is trained on those embeddings to map them to class labels. The head trains from scratch on the raw labeled examples — not pairs.

**Inference**

New text passes through the fine-tuned body, producing an embedding, which the classification head maps to a class label. Inference is fast: just a sentence transformer forward pass plus a logistic regression lookup.

**Key properties:**

- No prompts or templates required
- Works with 8–20 examples per class (50–200 total is comfortable)
- Scales combinatorially: more examples = exponentially more training pairs
- Body and head are trained independently — different learning rates, epochs, losses are valid per phase
- Some `TrainingArguments` accept tuples `(phase1_value, phase2_value)` for per-phase control

---

## 2. Installation

```bash
pip install setfit
# with GPU support (CUDA):
pip install setfit torch torchvision
# datasets library for data loading:
pip install datasets
```

Core dependencies: `sentence-transformers`, `scikit-learn`, `torch`, `transformers`, `datasets`.

Python 3.8+. GPU optional but recommended for faster training; CPU training with small models takes a few minutes.

---

## 3. Model Selection

SetFit uses any Sentence Transformer as its body. The body determines embedding quality — pick based on your accuracy vs. speed tradeoff.

### Recommended models for short text classification

| Model | Size | Speed | Best for |
|-------|------|-------|---------|
| `BAAI/bge-small-en-v1.5` | 33M | Fastest | Latency-critical, CPU deployment |
| `BAAI/bge-base-en-v1.5` | 109M | Fast | General-purpose, best accuracy/speed balance |
| `sentence-transformers/all-MiniLM-L6-v2` | 22M | Fastest | Very resource-constrained; ~5–8% accuracy loss vs. bge-base |
| `sentence-transformers/paraphrase-mpnet-base-v2` | 109M | Moderate | Strong semantic understanding, older but reliable |
| `answerdotai/ModernBERT-embed-base` | 149M | Fast | Long context (up to 8192 tokens), complex tasks |

**For short text (20–200 chars) with 2–4 classes and ~100 examples:**
`BAAI/bge-small-en-v1.5` or `BAAI/bge-base-en-v1.5` are the practical defaults. Start with `bge-small` and upgrade to `bge-base` if accuracy is insufficient.

**Domain-specific:** Use a domain-specific Sentence Transformer if available (e.g., biomedical, legal). Domain pre-training advantage is strongest in few-shot regimes and diminishes as training set grows.

**ModernBERT note:** `modernbert-embed-base` outperforms bge-base on long-context and complex tasks (92.7% vs 82.5% on ArXiv with 8 shots), but performs poorly as a pure transfer learning feature extractor without fine-tuning. Use it when your texts have rich semantic content or are longer than 512 tokens.

**Multilingual:** Any multilingual Sentence Transformer works — e.g., `sentence-transformers/paraphrase-multilingual-mpnet-base-v2`.

---

## 4. Training API

### Minimal training loop

```python
from datasets import Dataset
from setfit import SetFitModel, Trainer, TrainingArguments

# Your labeled data — must have "text" and "label" columns
train_data = {
    "text": [
        "cancel my subscription",
        "how do I cancel",
        "billing issue with my account",
        "charge on my card",
        "where is my order",
        "track my shipment",
    ],
    "label": [0, 0, 1, 1, 2, 2],  # or string labels
}
train_dataset = Dataset.from_dict(train_data)

# Initialize model with string labels for readable predictions
model = SetFitModel.from_pretrained(
    "BAAI/bge-small-en-v1.5",
    labels=["cancellation", "billing", "shipping"],
)

args = TrainingArguments(
    batch_size=16,
    num_epochs=4,
)

trainer = Trainer(
    model=model,
    args=args,
    train_dataset=train_dataset,
)
trainer.train()
```

### TrainingArguments — key parameters

```python
args = TrainingArguments(
    # Phase 1: contrastive embedding fine-tuning
    batch_size=16,          # or (16, 32) — tuple for (phase1, phase2)
    num_epochs=4,           # or (4, 20) — more epochs in phase2 often helps
    body_learning_rate=2e-5,  # LR for sentence transformer body
    num_iterations=20,      # pairs per sentence per class; default 20

    # Phase 2: classification head training
    head_learning_rate=1e-2,  # LR for the sklearn or pytorch head (ignored by sklearn LogReg)

    # Evaluation and saving
    eval_strategy="epoch",  # or "no"
    save_strategy="epoch",
    load_best_model_at_end=True,

    # Misc
    seed=42,
    max_steps=-1,           # override num_epochs with fixed step count
    warmup_proportion=0.1,  # fraction of training steps for LR warmup
    samples_per_label=2,    # examples per label per iteration in contrastive phase
)
```

`num_epochs` and `batch_size` can both be scalars (applied to both phases) or 2-tuples `(phase1, phase2)`.

### Trainer with evaluation dataset and column mapping

```python
from datasets import Dataset
from setfit import SetFitModel, Trainer, TrainingArguments, sample_dataset

# column_mapping when your dataset uses different column names
trainer = Trainer(
    model=model,
    args=args,
    train_dataset=train_dataset,
    eval_dataset=eval_dataset,
    metric="accuracy",                                # or "f1", or custom callable
    column_mapping={"sentence": "text", "cat": "label"},  # map to required text/label names
)

trainer.train()
metrics = trainer.evaluate()
print(metrics)  # => {'accuracy': 0.87}
```

### Save and load

```python
# Save locally
model.save_pretrained("./my-setfit-model")

# Load from local directory
model = SetFitModel.from_pretrained("./my-setfit-model")

# Push to Hugging Face Hub (requires HF account)
model.push_to_hub("your-username/my-setfit-model")

# Load from Hub
model = SetFitModel.from_pretrained("your-username/my-setfit-model")
```

### Sampling from larger datasets

```python
from setfit import sample_dataset

# Select num_samples examples per class from a larger labeled dataset
train_dataset = sample_dataset(
    full_dataset["train"],
    label_column="label",
    num_samples=16,    # 16 per class
    seed=42,
)
```

---

## 5. Inference

```python
# Single prediction
pred = model.predict(["cancel my account please"])
print(pred)   # => ["cancellation"]

# Batch prediction
preds = model.predict([
    "I was charged twice",
    "where is my package",
    "I want to unsubscribe",
])
print(preds)  # => ["billing", "shipping", "cancellation"]

# Probability scores (confidence)
probs = model.predict_proba([
    "cancel this immediately",
    "not sure what happened with my order",
])
print(probs)  # => array of shape (n_samples, n_classes) with probabilities

# Move model to specific device
model.to("cuda")   # or "cpu", "mps" for Apple Silicon
```

`model.predict()` returns string labels if `labels` was set on `from_pretrained`, otherwise returns integer tensor indices.

`model.predict_proba()` returns a numpy array of class probabilities — useful for threshold-based filtering or uncertainty estimation.

---

## 6. Dataset Preparation

### From a pandas DataFrame

```python
import pandas as pd
from datasets import Dataset

df = pd.DataFrame({
    "text": ["example one", "example two", "example three"],
    "label": ["class_a", "class_b", "class_a"],
})

# String labels work directly — SetFit encodes them internally
dataset = Dataset.from_pandas(df)
```

### Label encoding

SetFit handles string labels natively when you pass `labels=` to `from_pretrained`. Integer labels also work. The two must be consistent between training and inference.

```python
# String labels — most readable
model = SetFitModel.from_pretrained("BAAI/bge-small-en-v1.5", labels=["spam", "ham"])
model.predict(["buy now!!!"])  # => ["spam"]

# Integer labels — slightly less overhead
model = SetFitModel.from_pretrained("BAAI/bge-small-en-v1.5")
# predictions return tensor([1, 0, 0])
```

### Stratified train/test split

```python
from sklearn.model_selection import train_test_split
import pandas as pd
from datasets import Dataset

df = pd.read_csv("labeled.csv")  # columns: text, label

train_df, test_df = train_test_split(
    df, test_size=0.2, stratify=df["label"], random_state=42
)

train_dataset = Dataset.from_pandas(train_df.reset_index(drop=True))
test_dataset = Dataset.from_pandas(test_df.reset_index(drop=True))
```

Always stratify splits so each class is represented proportionally in train and test.

### With ~100 examples across 4 classes

At 100 examples / 4 classes = 25 per class, SetFit has more than enough for strong performance. Phase 1 will generate 25*24/2 = 300 positive pairs per class and many more negative pairs. Typical accuracy with 25 examples per class on a well-separated 4-class problem: 85–92%.

Diminishing returns kick in around 50–100 examples per class. Beyond that, standard fine-tuning (full BERT/RoBERTa) is worth considering.

---

## 7. Multi-label Classification

When each text can belong to multiple categories simultaneously:

```python
import numpy as np
from datasets import Dataset
from setfit import SetFitModel, Trainer, TrainingArguments

# Labels as multi-hot vectors: [is_billing, is_shipping, is_cancellation]
train_data = {
    "text": ["refund and cancel my order", "where is my bill"],
    "label": [
        [1, 1, 1],   # billing + shipping + cancellation
        [1, 0, 0],   # billing only
    ],
}
train_dataset = Dataset.from_dict(train_data)

model = SetFitModel.from_pretrained(
    "BAAI/bge-small-en-v1.5",
    multi_target_strategy="one-vs-rest",  # or "multi-output", "classifier-chain"
)

trainer = Trainer(
    model=model,
    args=TrainingArguments(batch_size=16, num_epochs=4),
    train_dataset=train_dataset,
    column_mapping={"text": "text", "label": "label"},
)
trainer.train()

preds = model.predict(["cancel and refund my bill"])
# => [[1, 1, 1]] — one prediction per input, multi-hot vector
```

**`multi_target_strategy` options:**

| Value | Description |
|-------|-------------|
| `"one-vs-rest"` | Binary classifier per label (default, robust) |
| `"multi-output"` | Multi-output classifier — single model, multiple outputs |
| `"classifier-chain"` | Chains classifiers, capturing label correlation |

For most use cases, `"one-vs-rest"` is the safe default.

---

## 8. Zero-shot Mode

When no labeled data is available, SetFit generates synthetic examples from class names using templates, trains on them, and performs zero-shot inference.

```python
from datasets import load_dataset
from setfit import SetFitModel, Trainer, TrainingArguments, get_templated_dataset

# The class names themselves act as semantic anchors
candidate_labels = ["cancellation", "billing", "shipping", "technical_support"]

# Generates templated synthetic examples like "This sentence is about cancellation"
train_dataset = get_templated_dataset(
    candidate_labels=candidate_labels,
    sample_size=8,          # synthetic examples per class
    text_column="text",
    label_column="label",
)

model = SetFitModel.from_pretrained("BAAI/bge-small-en-v1.5")

trainer = Trainer(
    model=model,
    args=TrainingArguments(batch_size=32, num_epochs=1),
    train_dataset=train_dataset,
)
trainer.train()

preds = model.predict(["my invoice is wrong", "track my package"])
```

Zero-shot SetFit consistently outperforms the Transformers zero-shot pipeline (`facebook/bart-large-mnli`) in both accuracy and speed (67x faster). Good class names matter — descriptive, distinct names work better than single-word ambiguous labels.

**Zero-shot accuracy benchmark (dair-ai/emotion, 6 classes):**

| Method | Accuracy | Latency |
|--------|----------|---------|
| SetFit zero-shot (bge-small) | 59.1% | 0.46ms/sentence |
| Transformers zero-shot (bart-large-mnli) | 37.7% | 31.2ms/sentence |

---

## 9. Hyperparameter Tuning

SetFit supports Optuna-based hyperparameter search. Useful when you want to squeeze out extra accuracy from a fixed labeled set.

```python
from setfit import SetFitModel, Trainer, TrainingArguments
from datasets import Dataset

def model_init(params):
    params = params or {}
    max_iter = params.get("max_iter", 100)
    solver = params.get("solver", "liblinear")
    return SetFitModel.from_pretrained(
        "BAAI/bge-small-en-v1.5",
        head_params={"max_iter": max_iter, "solver": solver},
    )

def hp_space(trial):
    return {
        "learning_rate": trial.suggest_float("learning_rate", 1e-6, 1e-4, log=True),
        "num_epochs": trial.suggest_int("num_epochs", 1, 5),
        "batch_size": trial.suggest_categorical("batch_size", [4, 8, 16, 32]),
        "num_iterations": trial.suggest_categorical("num_iterations", [5, 10, 20]),
        "seed": trial.suggest_int("seed", 1, 40),
        "max_iter": trial.suggest_int("max_iter", 50, 300),
        "solver": trial.suggest_categorical("solver", ["newton-cg", "lbfgs", "liblinear"]),
    }

trainer = Trainer(
    train_dataset=train_dataset,
    eval_dataset=eval_dataset,
    model_init=model_init,
)

# Run 10 Optuna trials — takes a few minutes with small models
best = trainer.hyperparameter_search(hp_space, n_trials=10, direction="maximize")
print(best)

# Apply best params and train final model
trainer.apply_hyperparameters(best.hyperparameters, final_model=True)
trainer.train()
```

**Practical impact:** Hyperparameter search typically yields 5–10% accuracy improvement over defaults. The biggest levers are `learning_rate` and `num_iterations`.

**Key hyperparameters ranked by impact:**

1. `learning_rate` — most influential; search `1e-6` to `1e-4` log-scale
2. `num_iterations` — controls contrastive pair count; higher = more training data, slower; try `[5, 10, 20, 40]`
3. `num_epochs` — overfitting risk increases past 5 with tiny datasets
4. `solver` / `max_iter` — logistic regression head settings; `liblinear` fast for small data, `lbfgs` for larger
5. `batch_size` — lower (4–8) can help with tiny datasets

---

## 10. Knowledge Distillation

Train a large accurate teacher, then distill into a smaller faster student using unlabeled data. Useful when you have unlabeled examples but want fast CPU inference.

```python
from datasets import load_dataset
from setfit import SetFitModel, Trainer, TrainingArguments, DistillationTrainer, sample_dataset

dataset = load_dataset("ag_news")
train_dataset = sample_dataset(dataset["train"], label_column="label", num_samples=16)
eval_dataset = dataset["test"]

# 1. Train teacher (large, accurate)
teacher_model = SetFitModel.from_pretrained("sentence-transformers/paraphrase-mpnet-base-v2")
teacher_trainer = Trainer(
    model=teacher_model,
    args=TrainingArguments(batch_size=16, num_epochs=4),
    train_dataset=train_dataset,
    eval_dataset=eval_dataset,
)
teacher_trainer.train()

# 2. Prepare unlabeled data for distillation
unlabeled = dataset["train"].shuffle(seed=0).select(range(500))
unlabeled = unlabeled.remove_columns("label")

# 3. Initialize student (small, fast)
student_model = SetFitModel.from_pretrained("sentence-transformers/paraphrase-MiniLM-L3-v2")

# 4. Distill teacher knowledge into student
distillation_trainer = DistillationTrainer(
    teacher_model=teacher_model,
    student_model=student_model,
    args=TrainingArguments(batch_size=16, max_steps=500),
    train_dataset=unlabeled,
    eval_dataset=eval_dataset,
)
distillation_trainer.train()

# Student now matches teacher accuracy with much faster inference
metrics = distillation_trainer.evaluate()
```

Distillation can yield up to 123x inference speedup with minimal accuracy loss. Use when you need CPU inference at scale.

---

## 11. Performance Characteristics

### Accuracy vs. training size

| Examples per class | Expected accuracy (2–4 classes, clean labels) |
|-------------------|----------------------------------------------|
| 4–8 | 70–80% — useful baseline, often beats zero-shot |
| 16–25 | 80–88% — strong for most production use cases |
| 50–100 | 88–93% — diminishing returns begin |
| 100+ | 90–95% — consider full fine-tuning instead |

These are rough guides for well-separated classes on short text. Ambiguous or overlapping classes reduce accuracy by 5–15%.

Benchmark (SST-2 sentiment, 8 examples per class):
- SetFit (bge-small): 85.1% accuracy
- SetFit (paraphrase-mpnet-base): 86.9% accuracy
- Full RoBERTa-Large fine-tune (3,000 examples): ~91%

### Training time

| Hardware | Model | Examples | Time |
|----------|-------|----------|------|
| NVIDIA V100 | any bge-small | 8/class | ~30 seconds |
| CPU (32 GB RAM) | bge-small | 16/class | ~2–5 minutes |
| CPU (32 GB RAM) | bge-base | 16/class | ~10–15 minutes |

Training cost with 8 examples on V100: ~$0.025. Equivalent T-Few 3B training: ~$0.70.

### Inference speed

| Setup | Throughput |
|-------|-----------|
| bge-small, CPU | ~2,000–5,000 texts/sec |
| bge-small, GPU | ~10,000–50,000 texts/sec |
| bart-large-mnli (zero-shot baseline) | 30–100 texts/sec |

SetFit inference is 67x faster than transformers zero-shot pipelines and up to 123x faster than teacher models after distillation.

### GPU requirements

Training: Single consumer GPU (e.g., RTX 3060, 8 GB VRAM) is fine for bge-small and bge-base with the labeled set sizes where SetFit is useful. No A100 needed.

Inference: CPU is viable for production with small models (bge-small). bge-small is 33M parameters — comparable to running a small web server.

---

## 12. Comparison with Alternatives

### SetFit vs. zero-shot classification (bart-large-mnli)

| Factor | SetFit (few-shot) | Zero-shot (bart-large-mnli) |
|--------|------------------|---------------------------|
| Labeled data needed | 8–50 per class | None |
| Accuracy (short text) | 85–92% | 60–75% |
| Inference speed | Very fast (ms) | Slow (30ms+/text) |
| Model size | 22–150M | 400M+ |
| Setup effort | 30–60 min labeling | None |
| GPU required | No (CPU viable) | Often yes |

**Recommendation:** If you can label 50–100 examples, SetFit beats zero-shot by 15–25% accuracy and is 67x faster. Zero-shot only wins when labeling is impossible.

### SetFit vs. full fine-tuning (BERT/RoBERTa)

| Factor | SetFit | Full fine-tuning |
|--------|--------|-----------------|
| Training data | 8–100/class | 500–5,000+/class |
| Training time | Minutes | Hours |
| Accuracy at small N | Competitive | Worse (overfits) |
| Accuracy at large N | Slightly worse | Better |
| Infrastructure | Consumer GPU/CPU | Often A100 |
| Expertise needed | Low | Moderate |

**Recommendation:** Use SetFit when N < 200 per class. Switch to full fine-tuning when N > 500 per class and you have the GPU budget.

### SetFit vs. LLM prompt-based classification (GPT-4, Claude)

| Factor | SetFit | LLM prompting |
|--------|--------|--------------|
| Cost at scale | Very low (self-hosted) | $0.01–0.10/1000 texts |
| Latency | 1–5ms | 200–2000ms |
| Accuracy (clear categories) | 85–92% | 85–95% |
| Accuracy (subtle categories) | Worse | Better |
| Privacy | Full (local) | Data leaves org |
| Iteration speed | Fast | Instant (no training) |

**Recommendation:** SetFit for high-volume, latency-sensitive, or privacy-constrained classification. LLM prompting for low-volume, nuanced categories where accuracy matters more than cost.

---

## 13. Pitfalls

**Using the deprecated `SetFitTrainer` instead of `Trainer`**

As of v1.0.0, `SetFitTrainer` was renamed to `Trainer` and hyperparameters moved to `TrainingArguments`. Old patterns like `SetFitTrainer(num_iterations=20)` no longer work.

```python
# Wrong (pre-v1.0)
trainer = SetFitTrainer(model=model, num_iterations=20, ...)

# Correct (v1.0+)
args = TrainingArguments(num_iterations=20)
trainer = Trainer(model=model, args=args, ...)
```

**Forgetting `column_mapping` when dataset columns aren't named "text" and "label"**

Trainer expects columns named `text` and `label`. If your dataset uses different names, provide a mapping:

```python
trainer = Trainer(
    ...,
    column_mapping={"sentence": "text", "category": "label"},
)
```

**Not stratifying splits with imbalanced classes**

If one class has 80% of examples, a random split may leave a class with zero test examples. Always use `stratify=` in `train_test_split` or verify class counts manually.

**Using too many epochs in Phase 1 (catastrophic forgetting)**

The sentence transformer body was pre-trained on massive data. Aggressive fine-tuning (many epochs, high LR) can destroy its general-purpose representations, hurting generalization. Keep `num_epochs` to 4–10 for Phase 1 and `learning_rate` at 1e-5 to 2e-5.

**Setting `num_iterations` too high without removing duplicates**

High `num_iterations` generates many redundant pairs, especially with small datasets. This slows training without accuracy benefit. The default of 20 is a good starting point; search up to 40 before going higher.

**Expecting SetFit to handle overlapping or ambiguous class definitions**

SetFit learns from the embedding geometry of your training examples. If "billing" and "refunds" genuinely overlap in your label scheme, even 1,000 examples won't fix it — the problem is the label taxonomy, not the model. Resolve class ambiguity in your labels before training.

**Ignoring predict_proba for uncertain inputs**

`model.predict()` always returns a label. For production, use `model.predict_proba()` and apply a confidence threshold to route low-confidence predictions to a human or a fallback.

```python
import numpy as np

probs = model.predict_proba(texts)
max_conf = np.max(probs, axis=1)
high_conf_mask = max_conf > 0.8

auto_labeled = [model.labels[i] for i in np.argmax(probs[high_conf_mask], axis=1)]
uncertain = [texts[i] for i, m in enumerate(~high_conf_mask) if m]
```

**Not rebuilding after changes when running inference from a saved model**

`model.save_pretrained()` saves both the body and head. Loading with `from_pretrained` restores both. If you retrain and forget to save, you are loading the old model.

**Using ModernBERT as a transfer-learning feature extractor without fine-tuning**

`answerdotai/ModernBERT-embed-base` underperforms standard BERT variants when used purely as a frozen feature extractor. It excels when fine-tuned end-to-end (which is exactly what SetFit's Phase 1 does). Avoid comparing ModernBERT's pre-fine-tuning quality to BGE's.

---

## Integration Patterns

### Production classification service (FastAPI)

```python
from fastapi import FastAPI
from pydantic import BaseModel
from setfit import SetFitModel
import numpy as np

app = FastAPI()
model = SetFitModel.from_pretrained("./my-setfit-model")
CONFIDENCE_THRESHOLD = 0.75

class ClassifyRequest(BaseModel):
    texts: list[str]

@app.post("/classify")
def classify(req: ClassifyRequest):
    probs = model.predict_proba(req.texts)
    results = []
    for text, prob_row in zip(req.texts, probs):
        max_conf = float(np.max(prob_row))
        label = model.labels[int(np.argmax(prob_row))]
        results.append({
            "text": text,
            "label": label if max_conf >= CONFIDENCE_THRESHOLD else "uncertain",
            "confidence": max_conf,
        })
    return {"results": results}
```

### Active learning loop

```python
# Start with 8 examples per class, iteratively add uncertain examples
from setfit import SetFitModel, Trainer, TrainingArguments
from datasets import Dataset
import numpy as np

def retrain(labeled_texts, labeled_labels, label_names):
    model = SetFitModel.from_pretrained("BAAI/bge-small-en-v1.5", labels=label_names)
    trainer = Trainer(
        model=model,
        args=TrainingArguments(batch_size=16, num_epochs=4),
        train_dataset=Dataset.from_dict({"text": labeled_texts, "label": labeled_labels}),
    )
    trainer.train()
    return model

def find_uncertain(model, unlabeled_texts, n=10):
    probs = model.predict_proba(unlabeled_texts)
    max_conf = np.max(probs, axis=1)
    uncertain_idx = np.argsort(max_conf)[:n]  # lowest confidence
    return uncertain_idx

# Iteration 0: bootstrap with 8/class
model = retrain(initial_texts, initial_labels, label_names)

# Iteration N: add uncertain examples that a human labels
uncertain_idx = find_uncertain(model, unlabeled_pool)
# ... human labels uncertain_pool[uncertain_idx] ...
# ... append to labeled set, retrain ...
```

### Batch inference with pandas

```python
import pandas as pd
from setfit import SetFitModel

model = SetFitModel.from_pretrained("./my-setfit-model")

df = pd.read_csv("to_classify.csv")
preds = model.predict(df["text"].tolist())
probs = model.predict_proba(df["text"].tolist())

df["predicted_label"] = preds
df["confidence"] = probs.max(axis=1)
df.to_csv("classified.csv", index=False)
```
