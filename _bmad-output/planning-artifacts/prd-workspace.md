---
title: "PRD: Sentence Embedding Evaluation Framework for Cold-Start Search"
status: "complete"
created: "2026-04-18"
updated: "2026-04-18"
inputs: ["docs/poduct-tech-brief-draft.txt"]
---

# PRD: Sentence Embedding Evaluation Framework for Cold-Start Search

## Purpose

This document defines the requirements for a course work project that builds and validates a sentence embedding evaluation pipeline for Learning-to-Rank (LTR) search. The goal is to produce metric-driven guidance for teams building search without pre-existing labeled data.

## Goals

1. Evaluate sentence embedding models based on user-provided labels across document fields and datasets
2. Build, train, and evaluate LTR models on the resulting similarity features
3. Fine-tune sentence embeddings, and evaluate the role of negative samplping strategies
4. Measure the impact of embedding fine-tuning and LLM-based data generation on LTR quality
5. Validate the pipeline on a second dataset and produce actionable guidance regarding which embeddings are the best, and the role of fine-tuning.

## Non-Goals

- Production deployment or real-time serving
- Multi-language or cross-lingual evaluation
- Building a user-facing search interface
- Custom Elasticsearch plugin development
- Candidate-generation algorithm for search
- Performance optimizations


---

## Epic 1: Data Preparation and Indexing

**Goal:** Ingest the Home Depot dataset into Elasticsearch with a clean, reusable schema and precomputed sentence embeddings for all candidate models.

### Story 1.1 — Elasticsearch Schema Design

**As a** researcher, **I want** a well-structured Elasticsearch schema for the Home Depot dataset **so that** all subsequent evaluation queries operate on a consistent document representation.

**Acceptance Criteria:**
- Schema explicitly maps: `title`, `description`, `brand_name`, `product_category`, `colour`
- Additional product-specific fields stored as `Array<String>` under a single `extra_fields` fields
- Optional additiona fields
- Schema is documented and version-controlled
- Index can be rebuilt deterministically from the raw dataset

### Story 1.2 — Sentence Embedding Indexing

**As a** researcher, **I want** sentence embeddings for all candidate models stored in the index **so that** cosine similarity can be computed via the Elasticsearch LTR plugin at query time.

**Acceptance Criteria:**
- All models available in `sentence-transformers` are indexed
- Additional HuggingFace models providing sentence-level embeddings are indexed
- Embeddings are stored per-field (not as a single concatenated document vector)
- Embedding dimension and model name are recorded in index metadata
- Indexing pipeline is reproducible and logged

### Story 1.3 — Dataset Splitting

**As a** researcher, **I want** the dataset split into train, test, and evaluation subsets **so that** all metric reporting is free of leakage and reliable.

**Acceptance Criteria:**
- Split is performed at the query level (not document level)
- At least two split strategies are researched and the chosen strategy is justified
- Optimal train/test/eval ratio is documented with rationale
- Split is deterministic (fixed random seed)
- Class balance (relevance grade distribution) is reported per split

---

## Epic 2: Baseline Evaluation

**Goal:** Establish a strong BM25 baseline and measure embedding model performance per field before any fusion or LTR training.

### Story 2.1 — BM25 Baseline

**As a** researcher, **I want** multiple BM25 query variants evaluated **so that** I have a robust traditional retrieval baseline to compare against embeddings.

**Acceptance Criteria:**
- At least 3 BM25 query strategies evaluated (e.g., `multi_match` best_fields, cross_fields, most_fields)
- NDCG@k and pair accuracy reported for each strategy on the test set
- Best-performing BM25 variant identified as the primary baseline

### Story 2.2 — Per-Model, Per-Field Embedding Evaluation

**As a** researcher, **I want** cosine similarity scores computed between query embeddings and each document field for every model **so that** I can rank models by field-level relevance quality.

**Acceptance Criteria:**
- Cosine similarity computed for every (model, field) combination using the Elasticsearch LTR plugin
- NDCG@k and pair accuracy reported for every (model, field) combination on the test set
- Results presented in a comparison matrix

---

## Epic 3: Signal Fusion

**Goal:** Combine embedding and BM25 similarity scores into fused ranking signals and evaluate fusion strategies.

### Story 3.1 — Reciprocal Rank Fusion

**As a** researcher, **I want** Reciprocal Rank Fusion applied to the similarity signals **so that** I can measure whether combining signals outperforms individual signals.

**Acceptance Criteria:**
- RRF applied to all similarity signals (per-model, per-field, BM25 variants)
- NDCG@k and pair accuracy reported for RRF output on the test set
- Comparison with best individual signal documented

### Story 3.2 — Weighted Reciprocal Rank Fusion

**As a** researcher, **I want** weights assigned to each signal based on training-set evaluation scores **so that** higher-quality signals contribute more to the fused ranking.

**Acceptance Criteria:**
- Weights derived exclusively from evaluation scores on the training split (no test/eval leakage)
- Weighted RRF evaluated on the test set
- Weight derivation methodology documented
- Metric-based comparison between unweighted RRF and weighted RRF reported

---

## Epic 4: LTR Model Training and Evaluation

**Goal:** Train LTR models on the full similarity feature matrix and analyze what drives ranking quality.

### Story 4.1 — Feature Matrix Construction

**As a** researcher, **I want** a structured feature matrix combining all similarity signals **so that** LTR models have a rich set of inputs.

**Acceptance Criteria:**
- Feature matrix includes: all (model, field) cosine similarities, BM25 variant scores, RRF scores
- Feature names are documented
- Matrix covers all query-document pairs in the training split

### Story 4.2 — XGBoost LTR Model

**As a** researcher, **I want** an XGBoost LTR model trained and evaluated **so that** I have a gradient-boosted baseline for learned ranking.

**Acceptance Criteria:**
- XGBoost trained with a ranking objective (e.g., LambdaRank or pairwise)
- Evaluated on the evaluation split using NDCG@k and pair accuracy
- Loss-based feature importance computed and reported
- Prediction-value-change feature importance computed and reported
- Top contributing features identified and analyzed

### Story 4.3 — Rank SVM LTR Model

**As a** researcher, **I want** a Rank SVM model trained and evaluated **so that** results can be compared against the XGBoost model.

**Acceptance Criteria:**
- Rank SVM trained on the same feature matrix as XGBoost
- Evaluated on the evaluation split using the same metrics
- Performance comparison with XGBoost documented
- Feature weights analyzed and reported

---

## Epic 5: Embedding Fine-Tuning

**Goal:** Fine-tune a sentence embedding model on the training data and measure the impact on LTR quality.

### Story 5.1 — Model Selection for Fine-Tuning

**As a** researcher, **I want** the best candidate model selected for fine-tuning **so that** fine-tuning effort is applied where it is most likely to yield improvement.

**Acceptance Criteria:**
- Selection justified by off-the-shelf evaluation results from Epic 2
- Rationale documented

### Story 5.2 — Fine-Tuning with Hard Negative Analysis

**As a** researcher, **I want** the selected model fine-tuned and the impact of hard negatives analyzed **so that** I understand how training data quality affects embedding quality.

**Acceptance Criteria:**
- Model fine-tuned on the Home Depot training split
- At least two training configurations: with and without hard negatives
- NDCG and pair accuracy compared across configurations
- Hard negative mining strategy documented

### Story 5.3 — LTR Integration of Fine-Tuned Model

**As a** researcher, **I want** the fine-tuned model added as a feature to the LTR model and the full training repeated **so that** I can measure the uplift it provides in the full ranking stack.

**Acceptance Criteria:**
- Fine-tuned model embeddings indexed for all documents
- Feature matrix updated with fine-tuned model similarity scores
- XGBoost and Rank SVM retrained
- Evaluation results compared against pre-fine-tuning LTR baseline

---

## Epic 6: LLM Dataset Augmentation

**Goal:** Use an LLM trained on assessor instructions to expand the labeled dataset, validate consistency, and measure the effect on LTR quality.

### Story 6.1 — LLM Fine-Tuning on Assessor Instructions

**As a** researcher, **I want** an LLM fine-tuned on the Home Depot assessor instructions **so that** it can generate relevance labels consistent with human annotation guidelines.

**Acceptance Criteria:**
- LLM fine-tuned using the Home Depot assessor instruction document as training signal
- Fine-tuning approach documented (prompt format, training examples)

### Story 6.2 — LLM Label Generation and Consistency Measurement

**As a** researcher, **I want** LLM-generated labels compared against human labels on held-out annotated examples **so that** I can quantify how reliable LLM labeling is.

**Acceptance Criteria:**
- LLM generates relevance labels for a held-out set of human-labeled query-document pairs
- Agreement rate (and optionally Cohen's kappa) between LLM and human labels reported
- Systematic disagreement patterns identified and discussed

### Story 6.3 — LTR Retraining and Vector-space model fine-tuning on Augmented Dataset

**As a** researcher, **I want** the LTR model retrained under multiple augmentation configurations **so that** I can measure the effect of LLM-augmented data on ranking quality.

**Acceptance Criteria:**
- Configuration A: LTR retrained, vector-space model fine-tune on LLM-generated labels only 
- Configuration B: LTR retrained vector-space model fine-tune on combined human + LLM labels
- Document count doubled; configurations A and B repeated on the larger dataset
- All four configurations evaluated on the evaluation split
- Results compared against all previous models
---

## Epic 7: Pipeline Replication on Second Dataset

**Goal:** Run the complete pipeline on a new dataset to validate generalizability.

### Story 7.1 — Second Dataset Selection and Ingestion

**As a** researcher, **I want** a second LTR dataset identified and ingested **so that** the pipeline can be validated beyond the Home Depot domain.

**Acceptance Criteria:**
- Dataset has human-annotated query-document pairs
- Dataset is sufficiently large for train/test/eval splitting
- Elasticsearch index created using the same pipeline as Dataset 1, but with the required adjustments

### Story 7.2 — Full Pipeline Execution on Second Dataset

**As a** researcher, **I want** the full evaluation pipeline (embedding evaluation → fusion → LTR → fine-tuning) run on the second dataset **so that** results can be compared cross-dataset.

**Acceptance Criteria:**
- All metrics from Epics 2–5 reproduced for the second dataset
- Cross-dataset comparison documented: which findings generalize, which are dataset-specific

---

## Epic 8: Guidance and Conclusions

**Goal:** Synthesize all experimental results into actionable, metric-driven recommendations.

### Story 8.1 — Guidance Document

**As a** reader (search engineer / ML engineer), **I want** a clear summary of what works and what doesn't **so that** I can make informed decisions when starting a search project from scratch.

**Acceptance Criteria:**
- Guidance covers: model selection (best off-the-shelf, best fine-tuned), field importance, fusion strategy, LTR model choice
- Each recommendation is linked to supporting metric evidence
- Trade-off between quality and inference speed addressed explicitly
- Guidance is dataset-domain-aware (noting which findings are domain-specific vs. general)

---

## Metrics Summary

| Metric | Where Used |
|---|---|
| NDCG@k | All evaluation stages |
| Pair accuracy | All evaluation stages |
| LLM-human label agreement rate | Epic 6 only |
| Feature importance scores | LTR analysis (Epic 4, 5) |

## Dependencies and Constraints

- Elasticsearch with LTR plugin must be available for local development
- HuggingFace model access required for embedding inference and fine-tuning
- GPU access required for embedding fine-tuning (sentence-transformers)
- Home Depot dataset must be downloaded and preprocessed prior to Epic 1
- LLM for augmentation must support fine-tuning or in-context labeling at scale

## Open Questions

1. Which second dataset will be used? (Selection affects Epic 7 scope)
2. What LLM will be used for dataset augmentation? (Affects consistency measurement approach)
3. What are the target split ratios? (Research required in Story 1.3)
4. That are the optimal parameters for the gradient boosting.
5. How to sample hard? Is it possible to use the LTR model to improve the vector-space embeddings?
