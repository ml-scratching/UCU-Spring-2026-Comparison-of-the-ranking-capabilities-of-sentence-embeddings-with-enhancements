---
title: "Product Brief: Sentence Embedding Evaluation Framework for Cold-Start Search"
status: "complete"
created: "2026-04-18"
updated: "2026-04-18"
inputs: ["docs/poduct-tech-brief-draft.txt"]
---

# Product Brief: Sentence Embedding Evaluation Framework for Cold-Start Search

## Executive Summary

Every search product starts with the same unsolvable paradox: you need user click data to train a ranking model, but you need a good ranking model to generate meaningful clicks. Hiring human assessors sidesteps this — but it is slow, expensive, and requires constant surveillance. This project cuts through that deadlock by repurposing existing, human-annotated Learning-to-Rank (LTR) datasets as a proxy for ground truth, systematically evaluating sentence embedding models across the full ranking stack, and producing metric-driven guidance that teams can act on from day one.

The result is a reusable benchmarking pipeline — built on Elasticsearch, sentence-transformers, and LTR tooling — that tells engineers *which* embedding models perform best, *how* to combine signals, and *what* improvements fine-tuning and LLM-augmented labeling actually buy. Rather than making arbitrary model choices at project launch, teams get an empirical foundation.

## The Problem

Search quality directly drives revenue and retention, yet every new product launches without a labeled dataset. Engineers face a choice between two imperfect paths:

- **Wait for clicks** — circular: you need a working ranker to generate useful signal, but you need signal to build the ranker
- **Hire assessors** — costly, slow to stand up, and brittle unless control questions are continuously refreshed to detect assessor drift

In practice, teams default to BM25 with ad hoc tuning, or pick an embedding model based on benchmarks that don't reflect their domain. There is no rigorous, reproducible process for making these decisions before user data exists.

## The Solution

This project builds a full evaluation pipeline anchored to two human-annotated LTR datasets. The pipeline:

1. Indexes documents into Elasticsearch with sentence embeddings from multiple models (sentence-transformers, HuggingFace)
2. Evaluates each model on every document field using NDCG and pair accuracy — producing a ranked comparison of embedding quality per field
3. Establishes a BM25 baseline and applies Reciprocal Rank Fusion (RRF) and Weighted RRF to combine signals
4. Trains LTR models (XGBoost, Rank SVM) on the combined similarity features
5. Fine-tunes the best candidate embedding model, analyzes the impact of hard negatives, and re-evaluates
6. Expands the labeled dataset using an LLM applied using original assessor instructions, measuring consistency with human labels
7. Replays the full pipeline on a second dataset to validate generalizability, potentially requires to come up with your own instructions for LLM-based labeling.

The output is not just a trained model — it is a set of concrete, metric-backed recommendations for building search without pre-existing labeled data.

## What Makes This Different

- **Empirical over opinionated** — every recommendation is backed by NDCG and LTR metrics, not vendor benchmarks or intuition
- **Full-stack coverage** — from BM25 baseline through fusion strategies, LTR training, embedding fine-tuning, and LLM data augmentation
- **LLM-augmented labeling** — uses the original assessor instructions to extend the dataset systematically, with consistency measurement against human labels
- **Validated on two+ datasets** — findings are not an artifact of one domain; the pipeline is replicated to test generalizability
- **Actionable guidance** — the final deliverable is a practical decision framework, not just experimental results

## Who This Serves

**Primary audience:** ML engineers and search engineers starting a new search product with no existing click data. They need a principled starting point for model selection and ranking strategy.

**Secondary audience:** Academic researchers studying embedding evaluation methodology, LTR benchmarking, and the use of LLMs for dataset augmentation.

## Success Criteria

| Criterion | Measure |
|---|---|
| Embedding comparison complete | NDCG and pair accuracy reported for all models × all fields |
| Baseline established | BM25 variants evaluated; RRF and Weighted RRF fusion applied |
| LTR models trained and evaluated | XGBoost and Rank SVM both trained, evaluated, and analyzed for feature importance based on previously extracted and evaluated features |
| Fine-tuning impact measured | Fine-tuned model evaluated against off-the-shelf; hard negative analysis complete |
| The fine-tuning impact on LTR measured | Fine-tuned model added to the featureset, and the LTR is retrained and evaluated |
| LLM augmentation validated | Consistency between LLM-generated and human labels measured; LTR retrained on augmented data; fine-tuning repeated; |
| Pipeline replicated | Full pipeline run on second dataset; cross-dataset comparison documented |
| Guidance delivered | Metric-driven recommendations published covering model selection and strategy |

## Scope

**In scope:**
- Home Depot dataset (multi-field: title, description, brand, category, colour)
- One additional LTR dataset (to be identified)
- Elasticsearch + LTR plugin as the evaluation platform
- Sentence-transformers models and HuggingFace-available embedding models
- XGBoost and Rank SVM as LTR algorithms for signals fusion
- At least one fine-tuned embedding model
- LLM-based dataset expansion with consistency evaluation
- Evaluation of doubling the LLM-based dataset

**Out of scope:**
- Production deployment or real-time inference optimization
- Multi-language or cross-lingual embedding evaluation
- Custom LTR plugin development
- User-facing search interface

## Vision

If this pipeline proves generalizable, it becomes a standard reference for cold-start search — a go-to empirical foundation that teams can adapt to their domain, rather than each team independently discovering the same lessons. In a broader research context, the methodology for measuring LLM-augmented label consistency could inform how the field approaches annotation at scale.
