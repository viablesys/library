# Text Classification Theory

Foundational concepts for designing, building, and evaluating text classifiers. This is a theoretical reference, not a code tutorial.

## 1. Taxonomy Design

A taxonomy is learnable when classes are mutually exclusive, semantically distinct, and aligned with surface-level language signals. The taxonomy design phase determines ceiling performance.

**Mutual exclusivity and granularity.** Classes should not overlap semantically. If a document could legitimately belong to multiple classes, move to multi-label classification. Fine-grained taxonomies (100+ classes) increase confusion and data sparsity; coarse taxonomies (3-5 classes) lack expressiveness. Start with 5-15 classes and refine downward.

**Surface language vs. pragmatic intent.** Failure occurs when a label refers to intent or context rather than textual features. For example, "user feedback is negative because of customer acquisition team failure" is not learnable from text alone—the label conflates sentiment with organizational facts. Separate labels based on what the text *contains*, not why it was created.

**Why taxonomies fail.** Overlapping categories (e.g., "complaint" and "angry feedback"), vague definitions ("miscellaneous"), or labels defined procedurally ("high-priority issue") rather than linguistically. Modern LLM-based approaches can refine taxonomies by detecting actual model confusion patterns and re-labeling categories to align with model inductive biases. If models persistently confuse two classes, the taxonomy may need merging or splitting.

**Validation step.** Before labeling a large dataset, label 50 documents and measure inter-annotator agreement (Cohen's kappa >0.7). If agreement is poor, the taxonomy is not ready.

## 2. Small Dataset Strategies

With 50–200 examples, the bottleneck is label scarcity, not feature learning. Maximize signal from limited data.

**Few-shot learning.** BERT fine-tuned on 50 examples reaches near-max performance; PET and other pattern-matching methods need 200+. SetFit (contrastive fine-tuning on Sentence Transformers) achieves 0.9 accuracy with 55 examples—comparable to BERT fine-tuned on 3,000. Contrastive learning creates positive/negative pairs from each example, multiplying the training signal without adding labels.

**Data augmentation.** Backtranslation (translate text to another language and back), paraphrasing, and synonym replacement add variability without changing intent. Augmentation is safe for small datasets but risks label noise—validate augmented examples manually on ~10% of new data.

**Active learning.** Train on an initial pool (20 examples per class), use model uncertainty to select the next batch (examples where the model is least confident), and label those. This requires fewer total labels than random sampling to reach the same performance. Uncertainty sampling (entropy, margin) is simple; query-by-committee (disagreement across ensemble members) is more robust.

**Teacher-student bootstrapping.** Fine-tune a larger model on your labeled data, use it to predict unlabeled data, then train a smaller student model on both labeled and high-confidence predictions. Effective when the teacher generalizes well and you have substantial unlabeled data.

## 3. Feature Representation

Text must be converted to numeric vectors. The choice affects interpretability, performance, and compute cost.

**Bag-of-words vs. embeddings.** Bag-of-words (TF-IDF, count vectors) treats text as an unordered collection of terms, assigning weights based on term frequency and document rarity. This preserves interpretability—you see exactly which words signal a class. Word embeddings (dense vectors) encode semantic relationships and capture context, but are black boxes. On datasets where class boundaries are clear and separable (e.g., spam vs. not spam), TF-IDF with an SVM often outperforms embeddings (0.987 vs. 0.95 accuracy). Embeddings excel when semantics matter (e.g., distinguishing sarcasm from sincere praise).

**When TF-IDF wins.** Small datasets (<500 examples), high sparsity preferred (few documents per class), linear separability in feature space, and the need to debug which words drive predictions. TF-IDF has no hyperparameters beyond vocabulary size; embeddings require model choice and hyperparameter tuning.

**Hybrid approaches.** Concatenate TF-IDF vectors with pretrained embeddings (e.g., BERT embeddings) to get both interpretability and semantic signal. This is common in production systems.

**Embeddings for small data.** Use pretrained embeddings (Word2Vec, FastText, BERT) rather than training from scratch. A frozen BERT encoder + simple classifier fine-tuned on 50 examples outperforms training embeddings end-to-end from random initialization.

## 4. Evaluation

Small dataset evaluation is fragile. Standard train/test splits overfit; proper methodology prevents illusions of performance.

**Stratified k-fold cross-validation.** Divide data into k folds (typically 5 or 10) such that each fold preserves class proportions. Train on k-1 folds, test on 1 fold, and average metrics across runs. This uses all data for both training and testing and detects overfitting better than a single split. With 50-200 examples, k-fold is essential because a single test set is too small to be representative.

**Per-class metrics.** Overall accuracy misleads when classes are imbalanced. Report precision, recall, and F1 per class. Precision = TP / (TP + FP): of the samples predicted as this class, how many were correct? Recall = TP / (TP + FN): of the true samples of this class, how many did the model find? F1 = 2 × (precision × recall) / (precision + recall) balances both.

**Confusion matrix analysis.** A confusion matrix shows actual vs. predicted labels. Patterns reveal systematic failures: if class A is always confused with class B, either the classes overlap or one class is rarer and harder to learn. Use confusion matrix to drive error analysis (see section 6).

**Avoiding test set overfitting.** Do not use test set results to make hyperparameter choices; use cross-validation or a held-out validation fold. If you tune on the test set, reported performance is optimistic.

## 5. Class Imbalance

When one class dominates (e.g., 90% negative, 10% positive), standard algorithms bias toward the majority class and ignore the minority.

**Oversampling and SMOTE.** Oversampling duplicates minority examples; SMOTE (Synthetic Minority Oversampling) generates synthetic examples by interpolating between neighbors in feature space. For text, SMOTE is risky because it operates in high-dimensional space where KNN is unreliable. Synthetic text often violates linguistic structure.

**Class weights and threshold tuning.** Instead of resampling, assign higher loss weight to minority examples during training. Most optimizers (SGD, Adam) support class weights directly. Alternatively, tune the decision threshold: if the model outputs P(minority), classify as minority only if P > 0.5 + ε. Moving the threshold shifts precision/recall trade-off.

**Weighted loss in practice.** Assign weight = (total samples) / (samples in class). For a 90/10 split: weight_majority = 1, weight_minority = 9. This automatically upweights minority errors during backpropagation.

**Why accuracy lies.** With 90/10 imbalance, a trivial classifier that always predicts majority achieves 90% accuracy. Use balanced accuracy = (recall_minority + recall_majority) / 2 or F1 for minority class as the primary metric.

## 6. Error Analysis

Understanding failures drives both model and taxonomy improvements.

**Confusion-driven re-labeling.** After training, inspect the confusion matrix. If class A has many false positives from class B, either the examples are mislabeled, the classes semantically overlap, or the model is underfitting. Manual review of 20-30 confusing examples reveals whether the problem is data or taxonomy.

**Failure taxonomy.** Categorize errors: are they annotation errors, examples that straddle two classes, or classes the model genuinely cannot distinguish? Build a structured error taxonomy (e.g., "annotation error," "ambiguous," "data drift," "model capacity") to track what's fixable.

**When to revise taxonomy vs. improve model.** If confusion is systematic across classes, the taxonomy is ambiguous—merge or re-define classes. If confusion is isolated to one pair of classes, the model may need more capacity or data augmentation for that pair. If errors are random, the dataset is too small; label more examples.

**External tool inspection.** For interpretable classifiers (e.g., linear SVM, logistic regression with TF-IDF), inspect feature weights to see which words the model relies on. Highly surprising weights (e.g., a stop word or artifact) suggest data labeling issues.

## 7. Transfer Learning

Pretrained models encode linguistic knowledge from large corpora. Leverage this to achieve high performance with few labels.

**Feature extraction vs. fine-tuning.** Use a pretrained encoder (e.g., BERT) to convert text to embeddings, then train a simple classifier (logistic regression, SVM, or small MLP) on top. This is fast and avoids overfitting; the pretrained model is frozen. Alternatively, fine-tune the encoder end-to-end on your labeled data, updating all weights. Fine-tuning captures task-specific patterns but risks overfitting with small data—use dropout, early stopping, and regularization.

**SetFit: contrastive fine-tuning.** SetFit fine-tunes a Sentence Transformer model using contrastive pairs created from labeled examples. Each example is paired with in-class positives and out-class negatives. Contrastive learning multiplies signal: from 8 examples per class, you generate hundreds of unique pairs. SetFit achieves competitive accuracy with 8-16 examples per class in 30 seconds on a V100. The two-stage process (embedding fine-tuning + classifier training) is crucial: stage 1 learns task-relevant embeddings; stage 2 trains a linear classifier on those embeddings.

**Pretrained model selection.** For few-shot classification, use lightweight models (DistilBERT, MiniLM) or domain-specific pretrained models (SciBERT for scientific text). Large models (BERT-large, GPT-3) overfit on small data. SetFit defaults to `sentence-transformers/all-mpnet-base-v2`, which balances size and performance.

**When transfer learning fails.** If your domain has unique language (e.g., medical jargon, code comments, specialized industry terms) and few labeled examples, a general pretrained model may not capture domain patterns. In this case, bootstrap with 100+ examples of domain-specific data, or use domain-adaptive pretraining (e.g., ClinicalBERT for healthcare).

---

## Roadmap

Start with taxonomy design (section 1). Validate on 50 examples (section 2). Choose representation based on interpretability needs (section 3). Evaluate with stratified k-fold and per-class metrics (section 4). If classes are imbalanced, apply weights or SMOTE (section 5). Run error analysis (section 6). For small datasets, default to SetFit or transfer learning (section 7).

## References

- [TELEClass: Taxonomy Enrichment and LLM-Enhanced Hierarchical Text Classification](https://arxiv.org/abs/2403.00165)
- [RAFT: A Real-World Few-Shot Text Classification Benchmark](https://datasets-benchmarks-proceedings.neurips.cc/paper/2021/file/ca46c1b9512a7a8315fa3c5a946e8265-Paper-round2.pdf)
- [Few-Shot Text Classification with SetFit](https://huggingface.co/docs/setfit/en/conceptual_guides/setfit)
- [Why TF-IDF Still Beats Embeddings](https://medium.com/@cmtwskb/why-tf-idf-still-beats-embeddings-ad85c123e1b2)
- [A Comprehensive Evaluation of Oversampling Techniques for Text Classification](https://www.nature.com/articles/s41598-025-05791-7)
- [ErrorMap: Model-Oriented Error Analysis for NLP Systems](https://arxiv.org/html/2601.15812)
- [Stratified K-Fold Cross-Validation for Small Datasets](https://towardsdatascience.com/how-to-plot-a-confusion-matrix-from-a-k-fold-cross-validation-b607317e9874/)
