---
title: "Product Tech Brief: Sentence Embedding Evaluation Framework for Cold-Start Search"
status: "complete"
created: "2026-04-18"
updated: "2026-04-18"
inputs: ["docs/poduct-tech-brief-draft.txt"]
---

# Product Tech Brief: Sentence Embedding Evaluation Framework for Cold-Start Search

## Overview

This document describes the technical architecture, data pipeline, model stack, and evaluation methodology for the Sentence Embedding Evaluation Framework — a course work project that benchmarks sentence embedding models for Learning-to-Rank (LTR) search using human-annotated datasets.

## Problem Statement (Technical Framing)

Cold-start search — building a ranking model without pre-existing click data — requires an offline evaluation strategy. The core technical challenge is: given a corpus, a set of queries, and human relevance judgements, which sentence embedding models best capture query-document relevance at the field level, and how should their signals be combined for ranking?

This project answers that question empirically using the Elasticsearch LTR plugin as the evaluation harness and two human-annotated datasets as ground truth.

## System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Data Ingestion                          │
│  Home Depot Dataset  ──►  Elasticsearch Index               │
│  (title, description,     - BM25 fields                     │
│   brand, category,        - Dense vectors per model         │
│   colour, custom fields)  - Schema: common + array overflow  │
└─────────────────────┬───────────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────────┐
│                   Dataset Splitting                          │
│  Split by query (not document) → train / test / eval        │
│  Research optimal split ratios and alternative strategies    │
└─────────────────────┬───────────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────────┐
│               Similarity Feature Computation                 │
│  For each (query, document field, embedding model):          │
│  - Cosine similarity via Elasticsearch LTR plugin            │
│  - BM25 variants as baseline features                        │
│  Output: feature matrix for LTR training                     │
└─────────────────────┬───────────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────────┐
│                    Evaluation Layer                           │
│  Per-model, per-field metrics:                               │
│  - NDCG@k                                                    │
│  - Pair accuracy                                             │
│  - State-of-the-art LTR metrics (to be researched)          │
└─────────────────────┬───────────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────────┐
│                  Signal Fusion                               │
│  - Reciprocal Rank Fusion (RRF)                              │
│  - Weighted Reciprocal Rank Fusion                           │
│  - Weights derived exclusively from training-set eval scores │
└─────────────────────┬───────────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────────┐
│                    LTR Training                              │
│  Models: XGBoost, Rank SVM                                   │
│  Features: all similarity signals (embedding + BM25)         │
│  Analysis: loss-based + prediction-value-change importance   │
└─────────────────────┬───────────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────────┐
│             Embedding Fine-Tuning                            │
│  - Fine-tune ≥1 sentence embedding model                     │
│  - Hard negative analysis                                    │
│  - Add fine-tuned model as new LTR feature; retrain          │
└─────────────────────┬───────────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────────┐
│              LLM Dataset Augmentation                        │
│  - Train LLM on Home Depot assessor instructions             │
│  - Generate labels for expanded document set                 │
│  - Measure consistency with human labels                     │
│  - Fine-tune vectors on: (a) LLM-only, (b) combined datasets │
│  - Retrain LTR on: (a) LLM-only, (b) combined datasets       │
│  - Double document count; repeat (a) and (b)                 │
└─────────────────────┬───────────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────────┐
│              Pipeline Replication (Dataset 2)                │
│  - Identify new LTR dataset                                  │
│  - Rerun full pipeline                                       │
│  - Cross-dataset comparison                                  │
└─────────────────────────────────────────────────────────────┘
```

## Data

### Dataset 1: Home Depot

- **Type:** E-commerce query-product relevance dataset with human assessor labels
- **Document fields:** title, description, brand name, product category, colour, product-specific fields
- **Schema strategy:** Map common fields explicitly (title, description, brand, category, colour); store remaining product-specific fields as `Array<String>` to simplify schema
- **Assessor instructions:** Available — used both as labeling guidance reference and as LLM fine-tuning input

### Dataset 2: TBD

- A second publicly available LTR dataset with human-annotated query-document pairs
- Selection criterion: sufficient query and document volume for meaningful train/test/eval split

### Dataset Splitting

- **Unit of split:** query (not document) — prevents query-level data leakage
- **Subsets:** train, test, evaluation
- **Rationale to research:** optimal split ratios, stratified vs. random split strategies, impact on metric stability

## Embedding Models

- All models available in the `sentence-transformers` package
- Additional models available on HuggingFace that provide sentence-level embeddings
- Stored as dense vectors in Elasticsearch at index time
- Evaluated per field independently (not concatenated document representation)

### Fine-Tuning

- At least one model selected based on off-the-shelf evaluation results
- Fine-tuned on the Home Depot training split
- Hard negative mining: analyze the effect of including hard negatives on fine-tuning quality
- Fine-tuned model added as a new feature to the LTR feature set; full LTR training repeated

## Evaluation Metrics

| Metric | Description |
|---|---|
| NDCG@k | Normalized Discounted Cumulative Gain — primary LTR quality metric |
| Pair accuracy | Fraction of correctly ordered query-document pairs |
| Additional LTR metrics | State-of-the-art metrics to be identified via literature review |

Metrics computed:
- Per model, per document field (embedding evaluation layer), aka pre feature
- Per LTR model (ranking quality layer)
- Per train/validation/test dataset (generalizability layer)

## Ranking & Fusion

### Baseline
Multiple BM25 query variants (e.g., multi-match, cross-fields, best-fields) to establish a strong traditional retrieval baseline.

### Signal Fusion
- **Reciprocal Rank Fusion (RRF):** Combines ranked lists without requiring score normalization
- **Weighted Reciprocal Rank Fusion:** Assigns weights to each signal; weights derived solely from training-set evaluation scores (no leakage into test/eval sets)

## LTR Models

| Model | Notes |
|---|---|
| XGBoost (regression/ranking) | Gradient-boosted trees; supports LambdaRank objective |
| Rank SVM | Pairwise ranking; strong baseline for LTR |

**Feature importance analysis:**
- Loss-based importance (XGBoost native)
- Prediction-value-change importance (permutation-based)
- Goal: understand which embedding models and fields drive ranking quality

## LLM Dataset Augmentation

**Motivation:** Assessors are expensive; LLMs can apply consistent labeling criteria at scale if the criteria are well-specified. The Home Depot dataset provides explicit assessor instructions — an unusually clean input for LLM fine-tuning.

**Protocol:**
1. Fine-tune an LLM on the Home Depot assessor instructions
2. Apply LLM to label additional documents
3. Measure consistency: agreement rate between LLM labels and human labels on held-out annotated examples
4. Fine-tune the vectors, and retrain latest LTR model on two extra configurations:
   - LLM-generated labels only
   - Combined human + LLM labels
5. Double the number of documents; repeat both retraining configurations
6. Compare LTR quality across all configurations

## Pipeline Replication

The full pipeline (indexing → evaluation → fusion → LTR → fine-tuning) is replicated on a second dataset. Goal: validate that findings generalize beyond the Home Depot domain and expose any pipeline assumptions that were dataset-specific.

## Technology Stack

| Component | Technology |
|---|---|
| Search engine | Elasticsearch |
| LTR plugin | Elasticsearch LTR plugin (feature logging + model scoring) |
| Embedding inference | sentence-transformers, HuggingFace Transformers |
| LTR training | XGBoost, scikit-learn (Implement Rank SVM) |
| Fine-tuning | HuggingFace Trainer / sentence-transformers training API |
| LLM augmentation | LLM of choice (GPT-5.4 class or open-weight equivalent) |
| Evaluation | Custom metrics + established LTR evaluation libraries |

## Deliverables

1. Elasticsearch schema and indexing pipeline
2. Per-model, per-field evaluation report (NDCG, pair accuracy)
3. Trained and evaluated LTR models (XGBoost, Rank SVM) with feature importance analysis
4. Fine-tuned embedding model with hard negative analysis
5. LLM augmentation consistency report + Additonal Fine-Tuning + LTR retrain comparison
6. Cross-dataset evaluation results, conclusion regarding which embeddings worked the best
7. Metric-driven guidance: which models to use, how to combine signals, what fine-tuning and LLM augmentation buy — for teams starting search from zero
