# Contrastive Learning for NLP — Reference

Theoretical reference with practical implications for text classification tasks. Covers the core mechanics, key architectures, and why contrastive fine-tuning works with small labeled datasets.

---

## 1. Core Concept

Contrastive learning is a form of metric learning. The objective is not to predict a label directly, but to learn an embedding space where similar examples are close and dissimilar examples are far apart. Training signal comes entirely from the structure of pairs or groups — no labels required in the unsupervised case.

The mechanism: given an anchor, pull its positive (semantically related) embedding toward it, and push all negatives away. The loss function quantifies how well the current embeddings satisfy this constraint.

### Siamese Networks

The standard architecture for contrastive learning is a siamese network: two (or more) copies of the same encoder sharing weights, processing different inputs simultaneously. Weights are identical because the goal is a single consistent embedding space, not separate representations for each input position.

At inference time only one copy of the encoder is used — the siamese structure is a training convenience to process pairs efficiently. SBERT (Reimers & Gurevych, 2019) brought this architecture to BERT-based models for sentence-level semantic similarity.

### Loss Functions

**Contrastive loss (Hadsell et al., 2006):** Operates on pairs. Minimizes distance for positive pairs, maximizes distance (up to a margin) for negative pairs. Forces intra-class distances toward zero, which can suppress meaningful within-class variation. Useful as a baseline and in self-supervised contexts; tends to underperform triplet and ranking losses on retrieval.

**Triplet loss (Schroff et al., 2015; FaceNet):** Operates on (anchor, positive, negative) triples. The objective is that `dist(anchor, positive) + margin < dist(anchor, negative)`. Allows intra-class spread — the cluster can be larger as long as negatives stay outside the margin. Produces stronger but less frequent gradient updates than contrastive loss. Used in the original SBERT as one of its training objectives.

**Multiple Negatives Ranking Loss (MNRL / InfoNCE / NT-Xent):** The dominant modern approach. Given a batch of (anchor, positive) pairs, every other positive in the batch serves as a negative for the current anchor. This is called in-batch negatives. The loss is cross-entropy over the similarity scores: maximize similarity for the true positive, minimize it for the `N-1` in-batch negatives. Also known as InfoNCE (Oord et al., 2018), NT-Xent (SimCLR), or simply cross-entropy with in-batch negatives.

Key properties of MNRL:
- Requires only (anchor, positive) pairs — no explicit negatives needed
- Scales with batch size: larger batches yield more negatives per anchor and generally better embeddings
- Temperature parameter τ controls sharpness; lower temperature concentrates probability mass and effectively emphasizes hard negatives
- Assumes that any two non-matching examples in a batch are true negatives — this fails when two anchors are semantically identical (false negative problem)

For most NLP embedding tasks, MNRL is the preferred loss. Triplet loss is useful when you have explicit negative supervision (e.g., NLI contradiction pairs). Contrastive loss remains common in unsupervised/self-supervised pre-training.

---

## 2. Sentence Embeddings via Contrastive Pre-training

Vanilla BERT embeddings perform poorly for sentence similarity. The reason is anisotropy: pre-trained BERT embeddings occupy a narrow cone in the high-dimensional space rather than being spread uniformly. Word frequency biases the space — high-frequency tokens cluster near the origin, low-frequency tokens scatter sparsely. Cosine similarity becomes nearly meaningless when all vectors point in roughly the same direction.

Contrastive pre-training directly addresses this by imposing two geometric properties on the learned space:

- **Alignment:** positive pairs (same meaning) should be close
- **Uniformity:** embeddings should be spread across the hypersphere

Wang & Isola (2020) showed these two properties together explain most of the effectiveness of contrastive losses. SimCSE's theoretical analysis builds on this framework directly.

### SBERT (Reimers & Gurevych, EMNLP 2019)

**Paper:** arXiv:1908.10084

SBERT was the first major application of siamese networks to BERT for semantic sentence similarity. The core insight: standard BERT requires both sentences to be fed simultaneously as a pair, making similarity search over 10,000 sentences require ~50 million inference passes (~65 hours). SBERT decouples this by encoding each sentence independently, reducing the same task to ~5 seconds.

Architecture: mean-pool BERT token embeddings to a fixed-size sentence vector, train via siamese fine-tuning on NLI and STS datasets, using classification and regression objectives. Mean pooling over CLS token performed best empirically.

SBERT established the training recipe; later work (SimCSE, sentence-transformers) replaced the fine-tuning objective with MNRL and hard negatives, achieving substantial gains.

### SimCSE (Gao, Yao & Chen, EMNLP 2021)

**Paper:** arXiv:2104.08821

Two modes, one framework:

**Unsupervised SimCSE:** Feed the same sentence twice through BERT with different dropout masks. The two resulting embeddings are the positive pair; all other sentences in the batch are negatives. Training objective is MNRL. The key insight is that dropout acts as minimal data augmentation — it introduces just enough perturbation to prevent representational collapse without distorting semantics. Removing dropout entirely collapses all embeddings to a single point.

Result: 76.3% average Spearman correlation on STS benchmarks using BERT-base, a 4.2% improvement over prior state-of-the-art.

**Supervised SimCSE:** Uses NLI datasets. Entailment pairs are positives; contradiction pairs are hard negatives (explicitly included in the loss, not just as in-batch negatives). Neutral pairs are discarded.

Result: 81.6% average Spearman correlation — 2.2% improvement over prior best. Validates that NLI supervision provides stronger alignment signal than dropout augmentation alone.

**Why it works:** SimCSE's contrastive objective regularizes the embedding space to be more uniform (spreads the embeddings across the hypersphere) while maintaining or improving alignment for same-meaning pairs. This directly fixes the anisotropy problem inherited from pre-training.

### SimCLR (Chen et al., ICML 2020) — Visual Precursor

**Paper:** arXiv:2002.05709

SimCLR is a computer vision paper but is the direct precursor to SimCSE's approach. Given an image, apply two random augmentations (crop, color jitter, blur) to get two views. Encode both, project through a learned MLP head, compute NT-Xent loss. The projection head is discarded after training — only the encoder is used downstream. This decoupling of the representation encoder from the contrastive projection head is a key structural insight adopted by SimCSE.

SimCLR showed that larger batch sizes, stronger augmentations, and the projection head all contribute substantially to representation quality. A linear classifier on top of SimCLR representations achieved 76.5% top-1 on ImageNet with no labels during training.

---

## 3. Few-Shot Fine-tuning: SetFit

**Paper:** arXiv:2209.11055 (Tunstall et al., NeurIPS 2022 workshop)
**Code:** github.com/huggingface/setfit

SetFit (Sentence Transformer Fine-tuning) is a two-stage framework designed for labeled-data-scarce classification.

### The Core Insight

Standard fine-tuning of large language models for classification requires thousands of labeled examples to update the embedding space meaningfully. SetFit separates the problem:

1. **Stage 1 — Contrastive fine-tuning:** Starting from a pretrained Sentence Transformer (e.g., `all-mpnet-base-v2`), generate training pairs by sampling within and across classes. Fine-tune using a siamese contrastive objective (cosine similarity + cross-entropy or MNRL). This reshapes the embedding space so that same-class sentences cluster tightly and different-class sentences are pushed apart. Only the embedding model is updated here.

2. **Stage 2 — Head training:** Freeze the fine-tuned encoder. Encode all labeled examples to get embeddings. Train a simple classification head (logistic regression or a small MLP) on these embeddings. The head requires very few gradient steps — it is solving a now-easy classification problem in a well-structured embedding space.

### Why So Few Examples Suffice

The critical observation: contrastive fine-tuning multiplies the effective training signal from labeled examples through pair generation. With 8 examples per class across C classes, naive training provides 8C data points. Pair generation produces O(k²) within-class positive pairs and O(k² × (C-1)) cross-class negative pairs per label group. The contrastive signal grows quadratically with examples, not linearly.

Empirically: 8 labeled examples per class on Customer Reviews sentiment is competitive with fine-tuning RoBERTa-large on 3,000 labeled examples. On RAFT (few-shot classification benchmark), SetFit with 355M parameters outperforms GPT-3 (175B parameters) and matches performance within 1-2% of models 30x larger (T-Few at 11B).

### What Is Actually Learned

Stage 1 is not learning to classify — it is learning a distance metric. The encoder learns that "this sentence should be near other sentences from its class and far from sentences of different classes." This is a different and stronger inductive bias than learning to predict a label token. The geometry of the embedding space encodes the decision boundary; the head merely reads it off.

---

## 4. Pair Generation and Hard Negative Mining

The quality of training pairs is often more important than the quantity of data. A contrastive model can only learn boundaries it is shown.

### Pair Generation Strategies

**From labeled data (SetFit style):**
- Positive pairs: randomly sample 2 examples from the same class label
- Negative pairs: randomly sample 1 example from each of two different classes
- With C classes and k examples per class, generates O(k² × C) positives and O(k² × C²) negatives
- Balance positives and negatives to avoid the model defaulting to "everything is different"

**From unlabeled data (SimCSE unsupervised style):**
- Positive pairs: same sentence with different dropout masks
- Negatives: all other sentences in the batch
- No labels needed; quality depends on how well dropout augmentation reflects semantic invariance

**From NLI datasets:**
- Positives: entailment pairs (the premise entails the hypothesis)
- Hard negatives: contradiction pairs (the premise contradicts the hypothesis)
- Neutral pairs are discarded — they are neither clearly positive nor clearly negative

### Hard Negative Mining

Easy negatives provide little gradient signal — if an anchor and negative are already far apart in the embedding space, the loss is already near zero and the update is tiny. Hard negatives are examples that the current model incorrectly places close to the anchor. They carry the largest gradient updates and drive the most learning.

Two principles for selecting useful negatives:
1. Only sample true negatives — avoid false negatives (examples that are semantically similar to the anchor but labeled differently)
2. Among true negatives, prefer those the current model scores as most similar to the anchor

Common strategies:
- **BM25 hard negatives:** retrieve top lexically similar non-matching documents; these share surface features but differ semantically
- **Model-mined hard negatives:** run a first-stage retrieval model, take top-k non-matching results as hard negatives for a second-stage model
- **In-batch negatives (MNRL default):** within a batch, all non-matching positives are negatives — with large batches and diverse sampling this naturally includes moderately hard cases

Warning: mining negatives that are too hard early in training causes divergence. The model has not yet learned the embedding space well enough to use them productively. A curriculum strategy (easy → hard) is more stable than hard negatives from the start.

### Pair Quality Effects

Poor pair quality manifests as:
- **Anisotropic collapse:** all embeddings converge to a narrow region; within-class variance collapses
- **Fuzzy class boundaries:** the model learns a noisy boundary that fails to generalize to new examples
- **False negative contamination:** hard negatives that are actually semantically similar drag the positive anchor away from its correct neighborhood

The SimCSE paper's use of NLI contradiction pairs as hard negatives (vs. random in-batch negatives) explains much of the supervised > unsupervised performance gap: contradiction pairs are maximally informative boundary examples.

---

## 5. Embedding Space Geometry

Contrastive training directly sculpts the geometry of the embedding space. Understanding what "good geometry" looks like is essential for diagnosing training problems.

### Alignment and Uniformity

Wang & Isola (2020) decomposed the quality of a representation space into two independent properties:

- **Alignment:** Positive pairs (semantically similar) should have nearly identical embeddings. Measured as average L2 distance between normalized embeddings of positive pairs.
- **Uniformity:** The marginal distribution of embeddings should be approximately uniform on the unit hypersphere. Measured as average pairwise Gaussian kernel value across all embeddings.

Good contrastive training minimizes both: tight alignment for positives, wide spread for the overall distribution. Anisotropic embeddings (like raw BERT) have poor uniformity — embeddings cluster in a narrow cone rather than spreading across the sphere.

SimCSE explicitly tracks alignment and uniformity metrics across training and shows that: unsupervised SimCSE improves uniformity substantially while maintaining alignment; supervised SimCSE improves both, with alignment gains explaining the performance advantage.

### Cluster Compactness and Inter-Cluster Distance

For classification, the geometry that matters is:
- **Intra-class compactness:** examples within the same class should form tight, compact clusters
- **Inter-class separation:** clusters of different classes should be well-separated with a clear margin

Triplet loss explicitly enforces a margin between intra- and inter-class distances, making it geometrically interpretable. MNRL achieves similar effects implicitly via the softmax over in-batch negatives — temperature τ controls how sharp the separation must be.

Pathological states:
- **Mode collapse:** all embeddings map to a single point; loss is near zero but no information is encoded. Prevented by the uniformity term in contrastive objectives, or by ensuring negatives in the batch are diverse.
- **Class overlap:** two classes that share surface features but differ in pragmatic intent occupy overlapping regions; the classifier cannot find a separating hyperplane. Solved by using better positives (same intent, diverse surface form) and hard negatives (opposite intent, similar surface form).

### Visualizing with UMAP / t-SNE

Both UMAP (McInnes et al., 2018) and t-SNE are dimensionality reduction methods grounded in attraction-repulsion dynamics — conceptually similar to contrastive learning itself. They are useful for diagnosing embedding geometry:

- Well-separated, compact clusters indicate the encoder has learned a discriminative space
- Overlapping clusters indicate insufficient separation — poor pair quality or insufficient fine-tuning
- Uniform spread across the plane indicates good uniformity (healthy); all-one-blob indicates collapse

Practical notes:
- t-SNE preserves local neighborhood structure well; distorts global distances
- UMAP preserves both local and some global structure; generally preferred for interpreting cluster relationships
- UMAP + HDBSCAN is a standard clustering pipeline: UMAP reduces to ~5-10 dimensions (not 2), HDBSCAN finds density-based clusters, then 2D UMAP visualization for display
- Compare UMAP plots before and after contrastive fine-tuning to measure what the training achieved

---

## 6. Application to Plan vs. Build Classification

This section discusses why contrastive learning is specifically well-suited to classifying utterances where surface language overlaps but pragmatic intent differs.

### The Problem

Consider two utterances:
- "We plan to build a data pipeline next quarter"
- "We are building a data pipeline right now"

Lexically, these sentences share most content words: build, data, pipeline. A bag-of-words classifier or a model relying on surface n-grams will struggle to separate them. Even a fine-tuned BERT model trained with cross-entropy on labels may not consistently separate these if the labeled data is small — the model may latch onto lexical features that happen to correlate with the label in training but fail on new surface variations.

The distinction is pragmatic, not lexical. "Plan to build" signals intent about a future action; "building" signals an ongoing, present action. This is an aspect of verbal aspect, modality, and tense — features that operate above the word level and are distributed across sentence structure.

### Why Contrastive Learning Captures This

Contrastive training on intent-labeled pairs directly teaches the model to cluster by pragmatic intent, not by surface similarity. The key is pair construction:

**Positive pairs (same intent, different surface form):**
- "We plan to build..." / "We are thinking about building..."
- "Currently building a pipeline" / "We have been working on the pipeline"

These pairs force the encoder to find features that both have in common — despite different wording — and represent them similarly. The shared feature is pragmatic intent, not lexical content.

**Hard negatives (different intent, similar surface form):**
- "Plan to build a data pipeline" (planning) vs. "Built a data pipeline" (completed)
- "Designing the architecture" (planning) vs. "Implementing the architecture" (executing)

These pairs force the encoder to push apart embeddings that share many surface features but differ in the crucial pragmatic dimension. These are the most informative training examples for learning the plan/build boundary.

### The Geometry That Results

After contrastive fine-tuning:
- "Plan" utterances form a compact cluster at one region of the embedding space
- "Build" utterances form a compact cluster at another region
- The decision boundary between them captures modality and tense markers, not content words

The classification head (logistic regression in SetFit) then only needs to find the hyperplane separating these two clusters — a trivial task given a well-structured embedding space.

This explains why contrastive fine-tuning works with small labeled datasets for this kind of task: 8-16 examples per class, used to generate many contrastive pairs, is often sufficient to reshape the embedding space around the pragmatic distinction that matters.

### Practical Recommendations

- Generate pairs that vary surface form while preserving intent (paraphrases, different syntactic constructions)
- Use plan/build labels to construct hard negatives explicitly — do not rely on random in-batch negatives alone
- At inference time, neighbor retrieval in the embedding space doubles as a confidence signal: if the nearest neighbors are a mix of plan and build examples, the model is uncertain
- UMAP visualization of the fine-tuned space should show two distinct clusters; overlap indicates insufficient hard negative training

---

## Key Papers

| Paper | Reference | Key Contribution |
|-------|-----------|-----------------|
| Sentence-BERT | Reimers & Gurevych, EMNLP 2019. arXiv:1908.10084 | Siamese BERT for sentence embeddings; mean pooling; reduced similarity search from 65h to 5s |
| SimCLR | Chen et al., ICML 2020. arXiv:2002.05709 | NT-Xent loss; projection head decoupling; in-batch negatives at scale for vision |
| SimCSE | Gao, Yao & Chen, EMNLP 2021. arXiv:2104.08821 | Dropout as augmentation (unsupervised); NLI contradiction hard negatives (supervised); alignment + uniformity theory |
| SetFit | Tunstall et al., NeurIPS workshop 2022. arXiv:2209.11055 | Two-stage: contrastive encoder fine-tuning + simple head; 8 examples/class competitive with full fine-tuning |
| Alignment & Uniformity | Wang & Isola, ICML 2020. arXiv:2005.10242 | Decomposed contrastive quality into two measurable geometric properties |

---

## Quick Reference: Loss Function Selection

| Situation | Recommended Loss |
|-----------|-----------------|
| Labeled pairs available, large dataset | MNRL (in-batch negatives) |
| Labeled pairs with explicit hard negatives (e.g., NLI contradiction) | MNRL + explicit negatives |
| Three-way supervision (anchor / positive / negative) | Triplet loss |
| Unsupervised, no labels | Contrastive loss with self-augmentation (dropout, SimCSE style) |
| Few-shot classification (8-16 per class) | SetFit: MNRL fine-tuning + logistic regression head |

---

## Quick Reference: Diagnosing Embedding Quality

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| All embeddings similar (low variance) | Representational collapse | Check negatives exist; increase batch diversity |
| Overlapping class clusters in UMAP | No pragmatic separation | Add hard negatives at class boundaries |
| Good training loss, poor generalization | False negatives in batch | Audit pairs; mine true hard negatives |
| One class dominates embedding space | Class imbalance in pairs | Balance positive/negative pair counts per class |
| Poor STS scores after fine-tuning | Catastrophic forgetting of general semantics | Use lower learning rate; start from strong SBERT base |
