# scikit-learn Text Classification Reference

Short text classification (20–200 chars) into 2–4 categories with ~100–200 labeled examples. Covers TF-IDF pipelines, classifier selection, feature engineering, cross-validation, calibration, embeddings, and persistence.

Docs: <https://scikit-learn.org/stable/>

---

## 1. Installation

```bash
pip install scikit-learn joblib sentence-transformers
```

```python
# Core imports used throughout this doc
from sklearn.pipeline import Pipeline, FeatureUnion
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.linear_model import LogisticRegression, LogisticRegressionCV, SGDClassifier
from sklearn.svm import SVC, LinearSVC
from sklearn.ensemble import RandomForestClassifier
from sklearn.calibration import CalibratedClassifierCV
from sklearn.model_selection import (
    StratifiedKFold, RepeatedStratifiedKFold,
    cross_val_score, GridSearchCV, RandomizedSearchCV
)
from sklearn.metrics import classification_report
import joblib
import numpy as np
```

---

## 2. Pipeline Architecture

The canonical pattern: vectorizer → classifier, all in a `Pipeline` object.

```python
from sklearn.pipeline import Pipeline
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.linear_model import LogisticRegression

pipeline = Pipeline([
    ("tfidf", TfidfVectorizer(
        ngram_range=(1, 2),       # unigrams + bigrams
        sublinear_tf=True,        # log(1 + tf) — compresses high-frequency terms
        max_features=5000,        # cap vocabulary for small datasets
        min_df=2,                 # ignore terms appearing in fewer than 2 docs
        strip_accents="unicode",
        analyzer="word",
    )),
    ("clf", LogisticRegression(
        C=1.0,
        solver="liblinear",       # preferred for small datasets
        max_iter=1000,
        class_weight="balanced",  # handles class imbalance
    )),
])

pipeline.fit(X_train, y_train)
y_pred = pipeline.predict(X_test)
print(classification_report(y_test, y_pred))
```

### Why Pipeline matters

- `fit_transform` on train, `transform` on test — prevents data leakage
- Saved as a single object; load it and call `predict(raw_text)` directly
- Grid search tunes vectorizer and classifier jointly with `__` separator

```python
# Access nested parameters with double underscore
pipeline.set_params(tfidf__ngram_range=(1, 3), clf__C=0.5)
```

---

## 3. TfidfVectorizer Parameters

### Key parameters for short text

| Parameter | Recommended | Notes |
|-----------|-------------|-------|
| `analyzer` | `"word"` or `"char_wb"` | `char_wb` handles typos, morphology |
| `ngram_range` | `(1, 2)` for word; `(2, 5)` for char | Bigrams capture phrases |
| `sublinear_tf` | `True` | `log(1 + tf)` — reduces dominance of repeated terms |
| `max_features` | `3000–10000` | Limit vocab size; critical for small datasets |
| `min_df` | `2` or `0.01` | Discard rare/noisy terms |
| `max_df` | `0.95` | Discard near-universal stop words |
| `strip_accents` | `"unicode"` | Normalize accented chars |
| `binary` | `True` for short text | Presence-only; very short texts have noisy tf values |

### Short text specifics

Very short text (20–100 chars) has noisy TF values — a word appearing once in 30 chars is not meaningfully different from twice. Consider `binary=True`:

```python
TfidfVectorizer(
    binary=True,          # presence/absence only, not frequency
    ngram_range=(1, 2),
    max_features=3000,
    min_df=2,
)
```

### `sublinear_tf` effect

```
raw tf=1  → log(2) ≈ 0.69
raw tf=5  → log(6) ≈ 1.79   (not 5x, ~2.6x)
raw tf=10 → log(11) ≈ 2.40  (not 10x, ~3.5x)
```

Compresses the range without zeroing counts. Almost always beneficial.

---

## 4. Feature Engineering

### Character n-grams

Robust to misspellings, morphological variation, and very short text where word tokenization is sparse.

```python
TfidfVectorizer(
    analyzer="char_wb",   # char n-grams within word boundaries (adds spaces)
    ngram_range=(2, 5),   # 2-grams through 5-grams
    max_features=5000,
    sublinear_tf=True,
)
# "words" → " w", "wo", "or", "rd", "ds", "s "
# "wprds" → " w", "wp", "pr", "rd", "ds", "s "
# Shared: " w", "rd", "ds", "s " — captures partial similarity
```

`analyzer="char"` vs `analyzer="char_wb"`:
- `char`: n-grams across all characters including spaces between words
- `char_wb`: n-grams within word boundaries (pads words with spaces) — less noisy, recommended

### Combining word + character n-grams with FeatureUnion

```python
from sklearn.pipeline import Pipeline, FeatureUnion
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.svm import LinearSVC

combined = Pipeline([
    ("features", FeatureUnion([
        ("word", TfidfVectorizer(
            analyzer="word",
            ngram_range=(1, 2),
            sublinear_tf=True,
            max_features=5000,
            min_df=2,
        )),
        ("char", TfidfVectorizer(
            analyzer="char_wb",
            ngram_range=(2, 5),
            sublinear_tf=True,
            max_features=5000,
            min_df=3,
        )),
    ])),
    ("clf", LinearSVC(C=1.0, max_iter=2000)),
])
```

### Feature combination comparison

| Approach | Vocab size | Handles typos | Short text | Training time |
|----------|-----------|---------------|------------|---------------|
| Word unigrams | Small | No | OK | Fast |
| Word 1-2 grams | Medium | No | Good | Fast |
| Char 2-5 grams | Large | Yes | Good | Moderate |
| Word + char | Large | Partial | Best | Moderate |

### Binary features for boolean signals

```python
# When categories differ by keyword presence, not frequency
TfidfVectorizer(binary=True, ngram_range=(1, 1), min_df=2)
```

---

## 5. Classifier Selection for Small Datasets

### Comparison: 100–200 labeled examples

| Classifier | Small data fit | Probability output | Speed | Notes |
|-----------|---------------|-------------------|-------|-------|
| `LogisticRegression` | Good | Native, well-calibrated | Fast | Start here; `liblinear` for small |
| `LinearSVC` | Excellent | Requires calibration | Fast | Often beats LR on high-dim text |
| `SVC(kernel='rbf')` | Excellent | Requires calibration | Moderate | Can dominate; O(n²) training |
| `SGDClassifier` | Mediocre | Unreliable on small data | Fast | Better for large-scale / online |
| `RandomForestClassifier` | Fair | Good | Slow | Struggles with sparse features |
| `MultinomialNB` | Good | OK | Very fast | Strong baseline; requires non-negative |

### Recommendation for <200 examples

1. **Start**: `LogisticRegression(solver='liblinear', C=1.0, class_weight='balanced')`
2. **Try**: `LinearSVC` — benchmarks show it frequently beats LR on text
3. **Try**: `SVC(kernel='rbf', probability=True)` for non-linearly separable categories
4. **Avoid**: `SGDClassifier` (unstable), `RandomForestClassifier` (poor with sparse)

### LogisticRegression — solver choice

```python
# Small datasets (<10k examples): liblinear
LogisticRegression(solver="liblinear", C=1.0, max_iter=1000)

# Multi-class with elasticnet regularization: saga
LogisticRegression(solver="saga", penalty="elasticnet", l1_ratio=0.5, max_iter=5000)

# Binary classification: liblinear handles natively
# Multi-class: all solvers except liblinear optimize multinomial loss
```

### SVC for small datasets

```python
from sklearn.svm import SVC

# Small data: SVC with RBF — finds optimal margin, handles non-linear boundaries
svc = SVC(
    C=1.0,
    kernel="rbf",
    probability=False,    # probability=True is slow; wrap with CalibratedClassifierCV instead
    class_weight="balanced",
)

# LinearSVC is faster and often equally good for text (already in high-dim space)
from sklearn.svm import LinearSVC
lsvc = LinearSVC(C=1.0, max_iter=2000, class_weight="balanced")
```

Warning: `SVC(probability=True)` fits an internal cross-validation to calibrate probabilities, which can be unreliable on very small datasets. Use `CalibratedClassifierCV` instead (see section 7).

---

## 6. Cross-Validation for Small Datasets

### StratifiedKFold — always use for classification

```python
from sklearn.model_selection import StratifiedKFold, cross_val_score

cv = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)

scores = cross_val_score(
    pipeline,
    X, y,
    cv=cv,
    scoring="f1_macro",   # macro for imbalanced multi-class
)
print(f"F1 macro: {scores.mean():.3f} ± {scores.std():.3f}")
```

### RepeatedStratifiedKFold — reduces variance on small datasets

With ~100 examples and 5-fold CV, each fold has only ~20 test samples. Variance is high. Repeat the CV multiple times:

```python
from sklearn.model_selection import RepeatedStratifiedKFold

cv = RepeatedStratifiedKFold(n_splits=5, n_repeats=10, random_state=42)
# 50 total evaluation runs — far more stable estimate
scores = cross_val_score(pipeline, X, y, cv=cv, scoring="f1_macro")
```

### Scoring metrics

```python
# For balanced classes
scoring = "accuracy"

# For imbalanced or multi-class
scoring = "f1_macro"      # unweighted mean of per-class F1
scoring = "f1_weighted"   # class-frequency-weighted F1

# Get multiple metrics at once
from sklearn.model_selection import cross_validate
results = cross_validate(pipeline, X, y, cv=cv,
                         scoring=["f1_macro", "accuracy"],
                         return_train_score=True)
```

### Pitfalls

| Pitfall | Problem | Fix |
|---------|---------|-----|
| `cross_val_score` default `shuffle=False` | Biased splits if labels are sorted | Always pass explicit `StratifiedKFold(shuffle=True)` |
| Tune hyperparameters inside same CV loop | Score inflation from leakage | Use nested CV or `LogisticRegressionCV` |
| 3-way split on <200 examples | Too few training samples | Use CV; reserve test set only for final evaluation |
| Single train-test split | High variance | Use CV for all intermediate decisions |

---

## 7. Hyperparameter Tuning

### LogisticRegressionCV — built-in C search, no leakage

```python
from sklearn.linear_model import LogisticRegressionCV

pipeline = Pipeline([
    ("tfidf", TfidfVectorizer(ngram_range=(1, 2), sublinear_tf=True, max_features=5000)),
    ("clf", LogisticRegressionCV(
        Cs=10,                      # 10 values from 1e-4 to 1e4 (log scale)
        cv=5,                       # internal StratifiedKFold
        scoring="f1_macro",
        solver="liblinear",
        max_iter=1000,
        class_weight="balanced",
    )),
])
pipeline.fit(X_train, y_train)
print(f"Best C: {pipeline.named_steps['clf'].C_}")
```

### GridSearchCV for joint vectorizer + classifier tuning

```python
from sklearn.model_selection import GridSearchCV

param_grid = {
    "tfidf__ngram_range": [(1, 1), (1, 2), (1, 3)],
    "tfidf__max_features": [2000, 5000],
    "tfidf__sublinear_tf": [True, False],
    "clf__C": [0.01, 0.1, 1.0, 10.0],
}

cv = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)
search = GridSearchCV(
    pipeline,
    param_grid,
    cv=cv,
    scoring="f1_macro",
    n_jobs=-1,
    refit=True,          # refit best params on full training set
)
search.fit(X_train, y_train)
print(search.best_params_)
best_pipeline = search.best_estimator_
```

### RandomizedSearchCV for larger search spaces

```python
from sklearn.model_selection import RandomizedSearchCV
from scipy.stats import loguniform

param_dist = {
    "tfidf__ngram_range": [(1, 1), (1, 2), (1, 3)],
    "tfidf__max_features": [1000, 3000, 5000, 10000],
    "clf__C": loguniform(1e-3, 1e2),
}

search = RandomizedSearchCV(
    pipeline,
    param_dist,
    n_iter=30,
    cv=cv,
    scoring="f1_macro",
    random_state=42,
    n_jobs=-1,
)
search.fit(X_train, y_train)
```

### Nested CV — unbiased performance estimate when tuning

```python
# Outer CV: estimate generalization; inner CV: tune hyperparameters
outer_cv = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)
inner_cv = StratifiedKFold(n_splits=3, shuffle=True, random_state=42)

inner_search = GridSearchCV(pipeline, param_grid, cv=inner_cv, scoring="f1_macro")
nested_scores = cross_val_score(inner_search, X, y, cv=outer_cv, scoring="f1_macro")
print(f"Nested CV F1: {nested_scores.mean():.3f} ± {nested_scores.std():.3f}")
```

---

## 8. Probability Calibration

Many classifiers produce uncalibrated scores. `LinearSVC` has no `predict_proba`. `SVC(probability=True)` fits an internal calibration that can be unreliable on small data. Use `CalibratedClassifierCV` explicitly.

### Which classifiers need calibration

| Classifier | Raw probabilities | Needs calibration |
|-----------|------------------|-------------------|
| `LogisticRegression` | Well-calibrated | Often not |
| `SVC(probability=False)` | None | Yes (add wrapper) |
| `LinearSVC` | None | Yes (add wrapper) |
| `SGDClassifier(loss='log_loss')` | Moderate | Sometimes |
| `RandomForestClassifier` | Over-confident | Sometimes |
| `GaussianNB` | Under-confident | Often |

### CalibratedClassifierCV

```python
from sklearn.calibration import CalibratedClassifierCV
from sklearn.svm import LinearSVC

base_clf = LinearSVC(C=1.0, max_iter=2000)
calibrated = CalibratedClassifierCV(
    base_clf,
    method="sigmoid",   # "sigmoid" (Platt scaling) or "isotonic"
    cv=5,               # internal CV to avoid leakage
)

pipeline = Pipeline([
    ("tfidf", TfidfVectorizer(ngram_range=(1, 2), sublinear_tf=True)),
    ("clf", calibrated),
])
pipeline.fit(X_train, y_train)
probs = pipeline.predict_proba(X_test)  # now available
```

### Method choice

| Method | When to use | Notes |
|--------|-------------|-------|
| `"sigmoid"` (Platt) | Small calibration sets (<1000); imbalanced | Parametric; fits intercept and slope |
| `"isotonic"` | Large calibration sets (>1000) | Non-parametric; overfits on small data |
| `"temperature"` | Multi-class; scikit-learn >=1.8 | Softmax temperature; natural multi-class |

For <1000 calibration samples, always use `"sigmoid"`. `"isotonic"` overfits and returns hard 0/1 probabilities on small data.

### Reading the probabilities

```python
probs = pipeline.predict_proba(["this is a test sentence"])
# shape: (1, n_classes)
classes = pipeline.classes_
confidence = probs.max(axis=1)  # highest probability for predicted class
uncertain = confidence < 0.6    # flag low-confidence predictions for review
```

### Calibration diagnostics

```python
from sklearn.calibration import CalibrationDisplay
import matplotlib.pyplot as plt

fig, ax = plt.subplots()
CalibrationDisplay.from_estimator(pipeline, X_test, y_test, ax=ax, n_bins=5)
plt.show()
# Ideal: diagonal line. Overconfident: curve below diagonal.
```

---

## 9. Embedding-Based Classification

When categories are semantically similar (hard to separate by keywords) or training data is very small, pre-computed sentence embeddings often outperform TF-IDF.

### Pattern: encode once, classify with sklearn

```python
from sentence_transformers import SentenceTransformer
from sklearn.linear_model import LogisticRegression
from sklearn.preprocessing import normalize
import numpy as np

# Encode all examples once — embeddings are reusable features
model = SentenceTransformer("all-MiniLM-L6-v2")  # 384-dim, fast, ~23MB
X_embeddings = model.encode(texts, show_progress_bar=True)
X_embeddings = normalize(X_embeddings)            # L2-normalize; often improves stability

# Classify with any sklearn estimator
clf = LogisticRegression(
    C=1.0,
    solver="liblinear",
    class_weight="balanced",
    max_iter=1000,
)
clf.fit(X_embeddings[train_idx], y[train_idx])
y_pred = clf.predict(X_embeddings[test_idx])
```

### Wrapping as a sklearn transformer

```python
from sklearn.base import BaseEstimator, TransformerMixin

class SentenceEmbedder(BaseEstimator, TransformerMixin):
    def __init__(self, model_name="all-MiniLM-L6-v2", normalize=True):
        self.model_name = model_name
        self.normalize = normalize

    def fit(self, X, y=None):
        from sentence_transformers import SentenceTransformer
        self._model = SentenceTransformer(self.model_name)
        return self

    def transform(self, X):
        from sklearn.preprocessing import normalize as sk_normalize
        embeddings = self._model.encode(list(X), show_progress_bar=False)
        if self.normalize:
            embeddings = sk_normalize(embeddings)
        return embeddings

# Full sklearn pipeline — works with cross_val_score, GridSearchCV, joblib
embedding_pipeline = Pipeline([
    ("embed", SentenceEmbedder("all-MiniLM-L6-v2")),
    ("clf", LogisticRegression(C=1.0, solver="liblinear", class_weight="balanced")),
])
embedding_pipeline.fit(X_train, y_train)
```

Note: `SentenceEmbedder.fit()` loads the model on first call. Cross-validation will reload it in each fold — cache the embeddings externally if CV is slow.

### Model selection for short text (<256 tokens)

| Model | Dim | Size | Speed | Quality |
|-------|-----|------|-------|---------|
| `all-MiniLM-L6-v2` | 384 | 23MB | Very fast | Good — default choice |
| `all-mpnet-base-v2` | 768 | 110MB | Moderate | Better quality |
| `paraphrase-MiniLM-L6-v2` | 384 | 23MB | Very fast | Good for paraphrase tasks |
| `multi-qa-MiniLM-L6-cos-v1` | 384 | 23MB | Very fast | Good for question-like text |

### Pre-computing and caching embeddings

```python
import numpy as np

# Encode all data once; save to disk
embeddings = model.encode(all_texts, batch_size=64, show_progress_bar=True)
np.save("embeddings_cache.npy", embeddings)

# Load for experiments without re-encoding
embeddings = np.load("embeddings_cache.npy")

# Then CV over classifiers only — fast
cv = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)
for clf in [LogisticRegression(C=1.0), LinearSVC(C=1.0), SVC(C=1.0)]:
    scores = cross_val_score(clf, embeddings, y, cv=cv, scoring="f1_macro")
    print(f"{clf.__class__.__name__}: {scores.mean():.3f} ± {scores.std():.3f}")
```

---

## 10. TF-IDF vs. Embeddings — Decision Guide

| Factor | TF-IDF + Linear | Sentence Embeddings |
|--------|----------------|---------------------|
| Training data | 50+ examples | 20+ examples (transfers knowledge) |
| Category distinction | Keyword-based | Semantic / meaning-based |
| Text length | Works at any length | Best for short-medium text |
| Interpretability | Feature weights visible | Opaque |
| Inference speed | ~0.1ms/text (sparse ops) | ~5–50ms/text (model forward pass) |
| Disk footprint | ~KB (vocab + weights) | 23MB–110MB (model) |
| Handles typos | Only with char n-grams | Yes (subword tokenization) |
| Handles synonyms | Only if both in training | Yes (semantic similarity) |
| Setup complexity | pip install sklearn | pip install sentence-transformers |

### When TF-IDF wins

- Categories are keyword-discriminative ("urgent", "refund", "bug report")
- Training set >100 examples
- Inference must be sub-millisecond
- Interpretability matters (feature importance via `clf.coef_`)

### When embeddings win

- Categories are semantically similar ("frustrated customer" vs. "happy customer")
- Training set <50 examples (transfers pretrained knowledge)
- Text contains synonyms, paraphrases, informal language
- Typos and spelling variation are common

### Hybrid: TF-IDF + embeddings

```python
from sklearn.pipeline import FeatureUnion
from sklearn.preprocessing import StandardScaler

# Concatenate TF-IDF sparse features with dense embeddings
# Requires pre-computing embeddings as a fixed array
from sklearn.base import BaseEstimator, TransformerMixin

class PrecomputedEmbeddings(BaseEstimator, TransformerMixin):
    """Pass-through for pre-computed embedding arrays."""
    def fit(self, X, y=None): return self
    def transform(self, X): return X  # X is already embeddings array

# Usually not worth the complexity — test both separately first
```

---

## 11. Model Persistence

### Save and load the full pipeline

```python
import joblib

# Save — bundles vectorizer vocabulary + classifier weights
joblib.dump(pipeline, "classifier.joblib")

# Load — call predict() on raw text immediately
pipeline = joblib.load("classifier.joblib")
label = pipeline.predict(["new text to classify"])[0]
proba = pipeline.predict_proba(["new text to classify"])[0]
```

### Compression for large models

```python
# compress=3 is a good balance; 9 is maximum (slow)
joblib.dump(pipeline, "classifier.joblib.gz", compress=3)
```

### Versioning — critical

```python
import sklearn
import sys

# Save metadata alongside model
meta = {
    "sklearn_version": sklearn.__version__,
    "python_version": sys.version,
    "classes": list(pipeline.classes_),
    "training_date": "2026-02-18",
    "n_train": len(X_train),
}
joblib.dump({"pipeline": pipeline, "meta": meta}, "classifier_versioned.joblib")

# On load, verify version
artifact = joblib.load("classifier_versioned.joblib")
loaded = artifact["pipeline"]
meta = artifact["meta"]
assert meta["sklearn_version"] == sklearn.__version__, \
    f"Version mismatch: saved={meta['sklearn_version']}, current={sklearn.__version__}"
```

scikit-learn provides no guarantee of cross-version compatibility for pickled models. Always record the version. If sklearn is upgraded, retrain and re-save.

### Fast startup pattern

```python
# Module-level load — pay I/O cost once per process
_pipeline = None

def get_pipeline():
    global _pipeline
    if _pipeline is None:
        _pipeline = joblib.load("classifier.joblib")
    return _pipeline

def classify(text: str) -> tuple[str, float]:
    pipe = get_pipeline()
    label = pipe.predict([text])[0]
    proba = pipe.predict_proba([text])[0].max()
    return label, proba
```

---

## 12. Complete Working Example

End-to-end: 150 labeled examples, TF-IDF + LR, cross-validated, calibrated, saved.

```python
import numpy as np
from sklearn.pipeline import Pipeline
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.linear_model import LogisticRegressionCV
from sklearn.calibration import CalibratedClassifierCV
from sklearn.model_selection import (
    RepeatedStratifiedKFold, cross_val_score, train_test_split
)
from sklearn.metrics import classification_report
import joblib

# Example data: classify support tickets into 3 categories
texts = [...]    # list of 150 short strings
labels = [...]   # list of 150 string labels: "billing", "bug", "feature"

# Hold out 20% for final evaluation only
X_train, X_test, y_train, y_test = train_test_split(
    texts, labels, test_size=0.2, stratify=labels, random_state=42
)

# Build pipeline with built-in C selection
pipeline = Pipeline([
    ("tfidf", TfidfVectorizer(
        ngram_range=(1, 2),
        sublinear_tf=True,
        max_features=5000,
        min_df=2,
        strip_accents="unicode",
    )),
    ("clf", CalibratedClassifierCV(
        LogisticRegressionCV(
            Cs=10,
            cv=3,
            solver="liblinear",
            max_iter=1000,
            class_weight="balanced",
        ),
        method="sigmoid",
        cv=3,
    )),
])

# Estimate generalization before final training
cv = RepeatedStratifiedKFold(n_splits=5, n_repeats=5, random_state=42)
scores = cross_val_score(pipeline, X_train, y_train, cv=cv, scoring="f1_macro")
print(f"CV F1 macro: {scores.mean():.3f} ± {scores.std():.3f}")

# Train on full training set
pipeline.fit(X_train, y_train)

# Final evaluation
y_pred = pipeline.predict(X_test)
print(classification_report(y_test, y_pred))

# Inspect confidence
probs = pipeline.predict_proba(X_test)
confidence = probs.max(axis=1)
low_conf = [(texts[i], confidence[i]) for i in np.where(confidence < 0.6)[0]]

# Persist
joblib.dump(pipeline, "support_classifier.joblib")
```

---

## 13. Common Pitfalls

### Fitting vectorizer on all data (data leakage)

```python
# WRONG — vectorizer sees test vocabulary
tfidf = TfidfVectorizer()
X_all = tfidf.fit_transform(all_texts)
X_train, X_test = train_test_split(X_all, ...)

# CORRECT — use Pipeline; fit_transform on train only, transform on test
pipeline = Pipeline([("tfidf", TfidfVectorizer()), ("clf", ...)])
pipeline.fit(X_train, y_train)
pipeline.predict(X_test)
```

### Not shuffling StratifiedKFold

```python
# WRONG — default shuffle=False; sorted data causes biased folds
cv = StratifiedKFold(n_splits=5)

# CORRECT
cv = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)
```

### Too many features relative to training size

```python
# With 100 training examples, a 50k-feature vocabulary overfits badly
TfidfVectorizer(max_features=50000)  # bad for 100 examples

# Rule of thumb: max_features ~ 10–50x training examples
TfidfVectorizer(max_features=3000, min_df=2)  # for 100 examples
```

### Using isotonic calibration with small data

```python
# WRONG — isotonic overfits with < 1000 calibration samples
CalibratedClassifierCV(clf, method="isotonic")

# CORRECT for small datasets
CalibratedClassifierCV(clf, method="sigmoid")
```

### SVC probability=True on small datasets

`SVC(probability=True)` fits a 5-fold internal CV to calibrate probabilities. With only 100 examples, 20 samples per fold makes calibration unreliable. Use `CalibratedClassifierCV` explicitly with control over cv splits.

### Ignoring class imbalance

```python
# If classes are 80/20 split, weighted F1 hides poor minority-class performance
# WRONG metric
scores = cross_val_score(pipeline, X, y, scoring="accuracy")

# CORRECT: use macro F1 + class_weight
pipeline = Pipeline([
    ("tfidf", TfidfVectorizer()),
    ("clf", LogisticRegression(class_weight="balanced")),
])
scores = cross_val_score(pipeline, X, y, scoring="f1_macro", cv=cv)
```

### SGDClassifier on small datasets

`SGDClassifier` is stochastic — it samples mini-batches and updates weights incrementally. With only 100 examples, variance between runs is high and convergence is unstable. Prefer `LogisticRegression` or `LinearSVC` which use deterministic solvers.

---

## 14. Quick Reference

### Decision tree for approach selection

```
Short text, 100–200 examples, 2–4 classes:

1. Are categories keyword-discriminative?
   YES → TF-IDF + LogisticRegression or LinearSVC
   NO  → Sentence Embeddings + LogisticRegression

2. Need probability scores?
   YES → LogisticRegression (native) or wrap with CalibratedClassifierCV
   NO  → LinearSVC is often the best performer

3. Very few examples (<50)?
   → Sentence Embeddings (pretrained knowledge transfers)
   → Or augment data before TF-IDF approach

4. Handling typos/informal text?
   → Add char_wb n-grams to TF-IDF
   → Or use sentence embeddings (handles subwords)

5. Need interpretability (which words matter)?
   → TF-IDF + linear classifier; inspect clf.coef_
```

### Parameter cheatsheet

```python
# Conservative starting point for ~100 examples
TfidfVectorizer(ngram_range=(1, 2), sublinear_tf=True, max_features=3000, min_df=2)
LogisticRegression(C=1.0, solver="liblinear", class_weight="balanced")
StratifiedKFold(n_splits=5, shuffle=True, random_state=42)
CalibratedClassifierCV(clf, method="sigmoid", cv=5)

# Inspect feature importance (linear classifiers only)
feature_names = pipeline.named_steps["tfidf"].get_feature_names_out()
coef = pipeline.named_steps["clf"].coef_   # shape: (n_classes, n_features)
top_features = np.argsort(coef[0])[-10:]   # top 10 for class 0
print(feature_names[top_features])
```
