# Wolf Pack Identity Trust Scoring Model — ML Documentation

## Table of Contents

1. [Overview](#overview)
2. [Architecture Context](#architecture-context)
3. [Dataset Generation](#dataset-generation)
4. [Feature Engineering](#feature-engineering)
5. [Scoring Engine & Trust Score Computation](#scoring-engine--trust-score-computation)
6. [Model Selection](#model-selection)
7. [Classification Models](#classification-models)
8. [Regression Models](#regression-models)
9. [Evaluation Framework & Accuracy Metrics](#evaluation-framework--accuracy-metrics)
10. [Error Analysis](#error-analysis)
11. [Model Comparison & Final Selection](#model-comparison--final-selection)
12. [Deployment — AWS SageMaker](#deployment--aws-sagemaker)
13. [Deployment — AWS EKS](#deployment--aws-eks)
14. [Endpoints, Clusters & Resource Paths](#endpoints-clusters--resource-paths)
15. [API Reference](#api-reference)
16. [File Inventory](#file-inventory)
17. [How to Reproduce](#how-to-reproduce)
18. [Troubleshooting](#troubleshooting)
19. [Production Considerations](#production-considerations)

---

## Overview

The **Wolf Pack Identity Trust Scoring Model** is the ML scoring engine for the **Zyberpol Iron Dome** security agent. It is the first model in the Zyberpol detection pipeline — its output feeds downstream into the **Hunter Behavioral Threat Model**, which fuses identity trust with behavioral detection signals.

**What it does:** Evaluates users and entities across **22 security features** spanning **five domains** and assigns a continuous **trust score (0–100)** mapped to four actionable enforcement tiers:

| Tier       | Score Range | Enforcement Action                    |
|------------|-------------|---------------------------------------|
| **GREEN**  | >= 75       | Allow — full access                   |
| **YELLOW** | 50 – 74     | Caution — step-up MFA, log session    |
| **ORANGE** | 25 – 49     | Elevated risk — session freeze, alert |
| **RED**    | < 25        | Block — kill switch, SOC alert        |

**What was built:**
- 1 synthetic dataset (100K rows, calibrated to 5 real-world security datasets)
- 4 classification models (tier prediction: GREEN/YELLOW/ORANGE/RED)
- 3 regression models (continuous trust score 0–100)
- Production deployment to **AWS SageMaker** (real-time endpoint)
- Production deployment to **AWS EKS** (Kubernetes pods with NLB)

**Production model:** Stacked Ensemble Regressor (XGBoost + LightGBM + CatBoost + Ridge meta-learner)

---

## Architecture Context

### Where Wolf Pack Fits in Zyberpol

```
┌─────────────────────────────────────────────────────────────────┐
│                    Zyberpol Iron Dome Pipeline                   │
│                                                                 │
│   ┌──────────┐     ┌──────────┐     ┌──────────────────────┐   │
│   │ Wolf Pack│────▶│  Hunter  │────▶│ Fire Factory / Weaver│   │
│   │ (Trust)  │     │ (Threat) │     │  (Strike / Enforce)  │   │
│   └──────────┘     └──────────┘     └──────────────────────┘   │
│        │                │                      │                │
│        ▼                ▼                      ▼                │
│   trust_score      threat_score         enforcement action      │
│   + tier           + risk_level         + SOC alert             │
│                                                                 │
│   NATS Mifal Bus: wolf_pack.trust.score → hunter.threat.assessed│
└─────────────────────────────────────────────────────────────────┘
```

### Wolf Pack Internal Architecture

```
                     ┌──────────────────────────────────────┐
                     │      22 Input Features (0–1)         │
                     │  HR · Device · Behavioral · ThreatI  │
                     │           · EventCtx                 │
                     └──────────────┬───────────────────────┘
                                    │
                 ┌──────────────────┼──────────────────┐
                 ▼                  ▼                   ▼
        ┌────────────────┐ ┌───────────────┐ ┌────────────────┐
        │ Classification │ │  Regression   │ │   Quantile     │
        │ (4-way tier)   │ │ (score 0-100) │ │ (P10/P50/P90)  │
        └────────┬───────┘ └───────┬───────┘ └────────┬───────┘
                 │                 │                   │
                 ▼                 ▼                   ▼
          GREEN/YELLOW/      trust_score +        score + 80%
          ORANGE/RED          tier mapping       confidence band
                                    │
                    ┌───────────────┼───────────────┐
                    ▼                               ▼
            ┌──────────────┐               ┌──────────────┐
            │  SageMaker   │               │     EKS      │
            │  Endpoint    │               │   Endpoint   │
            └──────────────┘               └──────────────┘
                    │                               │
                    └───────────┬───────────────────┘
                                ▼
                    ┌──────────────────────┐
                    │  Hunter Agent        │
                    │  (downstream consumer│
                    │   via NATS / HTTP)   │
                    └──────────────────────┘
```

### ROMA Meta-Agent (Perceive → Decide → Act)

The `wolf_pack_scoring_model.py` implements a **ROMA cognitive loop** that wraps the ML models:

| Phase       | What Happens                                                       |
|-------------|--------------------------------------------------------------------|
| **Perceive** | Collects 22 real-time signals from HR, device, behavioral, threat intel, event context sources |
| **Decide**   | Runs scoring engine → optionally invokes LLM for edge cases → applies session decay |
| **Act**      | Executes enforcement (ALLOW / MFA_STEP_UP / SESSION_FREEZE / KILL_SWITCH), publishes to Mifal NATS bus |

**LLM Router:** Routes inference to two models based on risk level:
- **Fast mode** (score 60–100, low risk): Nemotron-3-Nano
- **Accurate mode** (score < 60, requires reasoning): Llama-Nemotron-Ultra-253B

---

## Dataset Generation

**Script:** `build_realworld_wolfpack_dataset.py`
**Output:** `wolfpack_realworld_dataset.csv` — 100,000 rows, 26 columns
**Command:** `python build_realworld_wolfpack_dataset.py [--rows 100000]`

### Source Datasets

The synthetic dataset is calibrated against five open-source security datasets to ensure realistic feature distributions:

| Domain           | Source Datasets                                      | What It Calibrates               |
|------------------|------------------------------------------------------|----------------------------------|
| HR               | CERT Insider Threat v6.2                             | Tenure, role sensitivity, status |
| Device           | UNSW-NB15 + CIC-IDS-2017                            | Compliance scores, managed flags |
| Behavioral       | LANL Comprehensive Cyber-Security Events             | Login times, velocities, geo     |
| Threat Intel     | MaxMind GeoLite2 + AbuseIPDB + AlienVault OTX       | IP reputation, IOC matches       |
| Event Context    | Splunk BOTS v3                                       | Failed logins, MFA, escalation   |

### Dataset Characteristics

| Property           | Value                                 |
|--------------------|---------------------------------------|
| Total rows         | 100,000                               |
| Columns            | 26 (22 features + trust_score + label + user_id + timestamp) |
| Class distribution | Balanced: 25,000 per tier             |
| Trust score range  | 0–100 (continuous)                    |
| Noise              | N(0, 0.5) per domain sub-score        |
| Random seed        | 42 (reproducible)                     |

### Data Split

| Split | Rows   | Ratio | Purpose                                  |
|-------|--------|-------|------------------------------------------|
| Train | 70,000 | 70%   | Model training                           |
| Val   | 10,000 | 10%   | Hyperparameter tuning / early stopping   |
| Test  | 20,000 | 20%   | Final evaluation (held out, never tuned) |

---

## Feature Engineering

### 22 Features Across 5 Security Domains

All features are normalized to [0, 1]. Each domain contributes 0–20 points to the trust score (total 0–100).

#### HR Domain (f01–f04) — Identity Baseline

| Feature | Name             | Range | Description                                     | Trust Points |
|---------|------------------|-------|-------------------------------------------------|--------------|
| f01     | hr_status        | 0–1   | Active employment status (1=active, 0=terminated)| 5 pts        |
| f02     | tenure           | 0–1   | Normalized tenure in days (0–4000 → 0–1.0)      | 3 pts        |
| f03     | role_sensitivity | 0–1   | Role sensitivity / privilege level               | 5 pts (inverted: low=good) |
| f04     | background_check | 0–1   | Recency of background check (0–1095 days → 0–1) | 5 pts (recent=good) |

#### Device Domain (f05–f08) — Endpoint Posture

| Feature | Name              | Range | Description                              | Trust Points |
|---------|-------------------|-------|------------------------------------------|--------------|
| f05     | device_compliance | 0–1   | Overall device compliance score          | 7 pts        |
| f06     | device_managed    | 0/1   | Corporate-managed device (binary)        | 4 pts        |
| f07     | compliance_score  | 0–1   | Patch / update compliance level          | 5 pts        |
| f08     | scan_recency      | 0–1   | Last security scan recency               | 4 pts        |

#### Behavioral Domain (f09–f15) — Real-Time Session Signals

| Feature | Name                | Range | Description                                   | Trust Points |
|---------|---------------------|-------|-----------------------------------------------|--------------|
| f09     | time_of_day         | 0–1   | Off-hours indicator (1=deep night, 0=business) | 3 pts       |
| f10     | weekend             | 0/1   | Weekend indicator                              | 1.5 pts     |
| f11     | geo_risk            | 0–1   | Geographic risk (high: NG, RU, KP, IR, BY, VE; med: CN, PK, BR, TR, UA; low: US, GB, DE, CA, AU, FR, JP, SG, NL, SE) | 4 pts |
| f12     | known_device        | 0/1   | Previously seen device                         | 3 pts       |
| f13     | travel_plausibility | 0–1   | Travel speed plausibility (impossible travel)  | 3.5 pts     |
| f14     | velocity_minute     | 0–1   | Login velocity (per minute)                    | 2.5 pts     |
| f15     | velocity_hour       | 0–1   | Login velocity (per hour)                      | 2.5 pts     |

#### Threat Intelligence Domain (f16–f19) — External Feeds

| Feature | Name            | Range | Description                              | Trust Points |
|---------|-----------------|-------|------------------------------------------|--------------|
| f16     | ip_reputation   | 0–1   | IP reputation score (0=bad, 1=good)      | 8 pts        |
| f17     | anonymizer      | 0/1   | Tor / VPN anonymizer usage (1=yes)       | 4 pts (inverted) |
| f18     | known_malicious | 0/1   | Known malicious IP flag (1=yes)          | 4 pts (inverted) |
| f19     | ioc_matches     | 0–1   | Indicator of compromise matches          | 4 pts (inverted) |

#### Event Context Domain (f20–f22) — Session Events

| Feature | Name            | Range | Description                              | Trust Points |
|---------|-----------------|-------|------------------------------------------|--------------|
| f20     | failed_logins   | 0–1   | Failed login ratio                       | 8 pts (low=good) |
| f21     | mfa_fatigue     | 0–1   | MFA fatigue bombing indicator            | 6 pts (low=good) |
| f22     | priv_escalation | 0–1   | Privilege escalation attempt             | 6 pts (low=good) |

### Feature Importance Rankings

**From GradientBoosting classifier (classification):**

| Rank | Feature              | Importance | Domain       |
|------|----------------------|------------|--------------|
| 1    | f20_failed_logins    | 22.3%      | Event Ctx    |
| 2    | f05_device_compliance| 21.2%      | Device       |
| 3    | f16_ip_reputation    | 16.7%      | Threat Intel |
| 4    | f11_geo_risk         | 12.7%      | Behavioral   |
| 5    | f04_background_check | 7.8%       | HR           |
| 6    | f19_ioc_matches      | 6.7%       | Threat Intel |
| 7    | f09_time_of_day      | 6.6%       | Behavioral   |
| 8    | f08_scan_recency     | 2.7%       | Device       |
| 9    | f03_role_sensitivity | 1.5%       | HR           |
| 10   | f07_compliance_score | 1.2%       | Device       |

**From LightGBM quantile regression (regression):**

| Rank | Feature              | Splits | Domain       |
|------|----------------------|--------|--------------|
| 1    | f05_device_compliance| 1,375  | Device       |
| 2    | f20_failed_logins    | 1,130  | Event Ctx    |
| 3    | f16_ip_reputation    | 1,103  | Threat Intel |
| 4    | f04_background_check | 1,074  | HR           |
| 5    | f03_role_sensitivity | 972    | HR           |

**Key insight:** The top 4 features (`f20`, `f05`, `f16`, `f11`) account for **72.9%** of classifier importance and span all 5 domains except Event Context's MFA/escalation signals.

---

## Scoring Engine & Trust Score Computation

### 5-Domain Weighted Model

The original scoring engine (`wolf_pack_scoring_model.py`) uses a weighted multi-signal fusion formula:

**Domain Weights:**

| Domain         | Weight | Rationale                                        |
|----------------|--------|--------------------------------------------------|
| Behavioral     | 0.30   | Strongest real-time signal; detects active threats |
| Device         | 0.25   | Endpoint posture critical; compromised device = game over |
| HR             | 0.20   | Stable but slow-moving; identity baseline        |
| Threat Intel   | 0.15   | High-signal but sparse; IOC match = instant penalty |
| Event Context  | 0.10   | Corroborating evidence; confirms other signals   |

**Per-Domain Feature Weights:**

| Domain       | Feature → Weight                                                                 |
|--------------|----------------------------------------------------------------------------------|
| HR           | status (0.45), tenure (0.20), role_sensitivity (0.20), background_check (0.15)   |
| Device       | compliance (0.35), managed (0.25), compliance_score (0.25), scan_recency (0.15)  |
| Behavioral   | geo_risk (0.25), travel (0.20), known_device (0.15), velocity_min (0.15), time_of_day (0.10), velocity_hr (0.10), weekend (0.05) |
| Threat Intel | anonymizer (0.30), known_malicious (0.30), ip_reputation (0.25), ioc_matches (0.15) |
| Event Ctx    | failed_logins (0.40), mfa_fatigue (0.35), priv_escalation (0.25)                |

### Trust Score Formula (Dataset Generation)

For training data, the trust score is computed additively (not using domain weights) to provide a cleaner regression target:

```
trust_score = HR_sub(0-20) + Device_sub(0-20) + Behavioral_sub(0-20)
            + ThreatIntel_sub(0-20) + EventCtx_sub(0-20) + noise
```

Each sub-score awards points for "good" signal values (see Feature Engineering tables above).

### Exponential Decay Function

Trust scores decay over time during active sessions to enforce re-evaluation:

```
S(t) = max(floor, S₀ × λ^(t / interval))
```

| Parameter  | Value | Description                        |
|------------|-------|------------------------------------|
| S₀         | —     | Initial score at session start     |
| λ (lambda) | 0.98  | Decay rate (2% drop per interval)  |
| interval   | 5 min | Re-evaluation interval             |
| floor      | 30    | Minimum score (ORANGE, not RED)    |

**Decay curve example (starting from score 95):**

```
  0 min → 95.0 (GREEN)
  5 min → 93.1 (GREEN)
 15 min → 89.4 (GREEN)
 60 min → 74.4 (YELLOW) ← first tier drop
120 min → 58.4 (YELLOW)
240 min → 38.0 (ORANGE) ← second tier drop
480 min → 30.0 (ORANGE) ← floor reached, stays here
```

### Spec Deviations

The model intentionally extends the original Wolf Pack spec:

| Deviation | Spec Says | We Use | Why |
|-----------|-----------|--------|-----|
| #1 | 3-signal equal-weight: `(Location + Device + Behavioral) / 3` | 5 domains with unequal weights (0.20/0.25/0.30/0.15/0.10) | Spec formula lacks threat intel and event context; credential stuffing attacks would score identically to clean users |
| #2 | "Nigeria IP + Disabled AV → Score < 30, BLOCK" | Compound override rules | Spec formula `(0 + 0 + 100)/3 = 33.3` cannot mathematically produce < 30 from 2 bad signals; formula and acceptance criteria contradict |

---

## Model Selection

### Why Multiple Model Types?

We trained 7 models across two task formulations to find the best approach:

| Approach       | Models Trained | Rationale                                                |
|----------------|---------------|----------------------------------------------------------|
| Classification | 4 models      | Direct tier prediction; simpler but loses score granularity |
| Regression     | 3 models      | Continuous score; preserves granularity, allows threshold tuning |

### Why Regression Was Chosen Over Classification

1. **Granularity** — A score of 76 (barely GREEN) vs 95 (strongly GREEN) matters for downstream Hunter model
2. **Threshold flexibility** — Tier boundaries can be adjusted in production without retraining
3. **Uncertainty quantification** — Quantile regression provides confidence intervals
4. **Downstream integration** — Hunter model consumes raw `trust_score` as a continuous feature

### Why Stacked Ensemble Over FT-Transformer

The **Stacked Ensemble** was selected for production despite the FT-Transformer having slightly better metrics:

| Criterion          | Stacked Ensemble                      | FT-Transformer                |
|--------------------|---------------------------------------|-------------------------------|
| R²                 | 0.9954                                | **0.9956** (marginal lead)    |
| RMSE               | 1.1011                                | **1.0821**                    |
| Tier Accuracy      | 96.95%                                | **97.07%**                    |
| GPU Required       | **No** (CPU-only inference)           | Yes (PyTorch runtime)         |
| Artifact Size      | **~50 MB** (pickle)                   | ~900 MB (PyTorch + weights)   |
| Inference Latency  | **~1ms** per batch                    | ~5ms per batch                |
| Interpretability   | **High** (SHAP, base learner weights) | Low (black box)               |
| Deploy Complexity  | **Simple** (sklearn container)        | Complex (PyTorch + CUDA)      |

**Decision:** R² difference of 0.0002 is negligible. Ensemble wins on deployment simplicity, latency, and interpretability.

---

## Classification Models

Four models predict the trust tier (GREEN / YELLOW / ORANGE / RED) as a 4-class classification task.

### 1. GradientBoosting Classifier (sklearn baseline)

**Script:** `train_wolfpack_ml_model.py` | **Artifact:** `wolfpack_ml_model.pkl` | **Metrics:** `wolfpack_ml_metrics.json`

| Metric         | Value   |
|----------------|---------|
| Test Accuracy  | 96.24%  |
| Macro F1       | 0.9623  |
| Val Accuracy   | 95.68%  |

Hyperparameters: `n_estimators=200, max_depth=4, learning_rate=0.1`

**Confusion Matrix (Test Set — 20,000 samples):**
```
              Predicted
              GREEN  YELLOW  ORANGE   RED
Actual GREEN   4947      0       0     53
       YELLOW     0   4756      61    183
       ORANGE     0    154    4846      0
       RED      150    152       0   4698
```

### 2. PyTorch FT-Transformer Lite

**Script:** `train_wolfpack_dl_model.py` | **Artifact:** `wolfpack_dl_model.pt` | **Metrics:** `wolfpack_dl_metrics.json`

| Metric         | Value    |
|----------------|----------|
| Test Accuracy  | 96.41%   |
| Macro F1       | 0.9640   |
| Model Params   | 107,844  |
| Epochs         | 27 / 60 (early stopped, patience=10) |
| Best Val F1    | 0.9607   |

Architecture:
```
22 features → Per-feature linear tokenizer (22 tokens, d_model=64 each)
           → Prepend learnable [CLS] token (23 tokens total)
           → 3× Transformer encoder (8 heads, pre-norm, ff_hidden=128)
           → [CLS] token → LayerNorm → MLP head (64→64 GELU→4)
           → 4-class softmax
```

Hyperparameters: `d_model=64, n_heads=8, n_layers=3, ff_hidden=128, dropout=0.1, lr=0.001, batch_size=256`
Optimizer: AdamW | Loss: CrossEntropyLoss with class weights

**Confusion Matrix (Test Set):**
```
              Predicted
              GREEN  YELLOW  ORANGE   RED
Actual GREEN   4953      0       0     47
       YELLOW     0   4737      62    201
       ORANGE     0    134    4866      0
       RED      149    126       0   4725
```

### 3. Keras/TensorFlow FT-Transformer Lite

**Script:** `train_wolfpack_keras_model.py` | **Artifact:** `wolfpack_keras_model.keras` | **Metrics:** `wolfpack_keras_metrics.json`

| Metric         | Value    |
|----------------|----------|
| Test Accuracy  | **96.48%** (best classifier) |
| Macro F1       | **0.9648** |
| Model Params   | 107,844  |
| Epochs         | 27       |
| Framework      | TensorFlow 2.20.0 |
| Best Val Acc   | 96.06%   |

Same architecture as PyTorch variant. Uses `AdamW` with `CosineDecay` schedule, `weight_decay=1e-4`.
Loss: `SparseCategoricalCrossentropy(from_logits=True)`

**Confusion Matrix (Test Set):**
```
              Predicted
              GREEN  YELLOW  ORANGE   RED
Actual GREEN   4924      0       0     76
       YELLOW     0   4770      79    151
       ORANGE     0    104    4896      0
       RED      129    165       0   4706
```

### 4. Official RTDL FT-Transformer

**Script:** `train_wolfpack_rtdl_model.py` | **Artifact:** `wolfpack_rtdl_model.pt` | **Metrics:** `wolfpack_rtdl_metrics.json`

| Metric         | Value    |
|----------------|----------|
| Test Accuracy  | 96.45%   |
| Macro F1       | 0.9645   |
| Model Params   | 900,868  |
| Epochs         | 35 / 60 (early stopped, patience=10) |
| Best Val F1    | 0.9606   |

Uses `rtdl.FTTransformer.make_default()` — reference implementation with paper-recommended hyperparameters (Gorishniy et al., "Revisiting Deep Learning Models for Tabular Data", NeurIPS 2021).

Hyperparameters: `n_blocks=3, lr=0.0001, batch_size=256`

**Confusion Matrix (Test Set):**
```
              Predicted
              GREEN  YELLOW  ORANGE   RED
Actual GREEN   4927      0       0     73
       YELLOW     0   4784      63    153
       ORANGE     0    122    4878      0
       RED      129    170       0   4701
```

---

## Regression Models

Three models predict the continuous trust score (0–100), then map to tiers via thresholds.

### 1. Stacked Ensemble Regressor (PRODUCTION MODEL)

**Script:** `train_wolfpack_regression_ensemble.py` | **Artifact:** `wolfpack_regression_ensemble_model.pkl` | **Metrics:** `wolfpack_regression_ensemble_metrics.json`

**Architecture:** 5-fold out-of-fold (OOF) stacking

```
22 features → StandardScaler
           → 5-fold cross-validation:
               ├── XGBoost    → OOF predictions column 0
               ├── LightGBM   → OOF predictions column 1
               └── CatBoost   → OOF predictions column 2
           → Ridge meta-learner (3 inputs → 1 output)
           → clip(0, 100) → trust_score
```

**Base Learner Results (Test Set):**

| Base Learner | Test RMSE | Test MAE | Test R²  | Tier Acc |
|-------------|-----------|----------|----------|----------|
| XGBoost     | 1.1476    | 0.9099   | 0.9950   | 96.88%   |
| LightGBM    | 1.1636    | 0.9229   | 0.9949   | 96.75%   |
| CatBoost    | 1.1028    | 0.8790   | 0.9954   | 96.94%   |

**Meta-Learner (Ridge) Weights:**

| Base Learner | Weight | Contribution |
|-------------|--------|--------------|
| XGBoost     | 0.1022 | 10.2%        |
| LightGBM    | 0.0775 | 7.8%         |
| **CatBoost**| **0.8219** | **82.2%** (dominant) |
| Intercept   | -0.1127 | —           |

**Ensemble Results (Test Set):**

| Metric         | Value    |
|----------------|----------|
| Test RMSE      | **1.1011** |
| Test MAE       | **0.8770** |
| Test R²        | **0.9954** |
| Tier Accuracy  | **96.95%** |

Hyperparameters (all base learners): `n_estimators=500, max_depth=6, learning_rate=0.05, subsample=0.8, colsample_bytree=0.8, early_stopping_rounds=20`

### 2. FT-Transformer Regressor

**Script:** `train_wolfpack_regression_transformer.py` | **Artifact:** `wolfpack_regression_transformer_model.pt` | **Metrics:** `wolfpack_regression_transformer_metrics.json`

| Metric         | Value    |
|----------------|----------|
| Test RMSE      | **1.0821** (best) |
| Test MAE       | **0.8629** (best) |
| Test R²        | **0.9956** (best) |
| Tier Accuracy  | **97.07%** (best) |
| Loss Function  | Huber (delta=5.0) |
| Model Params   | 900,289  |
| Epochs         | 23 / 60 (early stopped) |
| Best Val RMSE  | 1.0924   |

Uses `rtdl.FTTransformer.make_default(d_out=1)` with target standardization (mean=68.58, std=16.21).
Huber loss is robust to outliers vs MSE.

### 3. Quantile Regression (Uncertainty-Aware)

**Script:** `train_wolfpack_regression_quantile.py` | **Artifact:** `wolfpack_regression_quantile_model.pkl` | **Metrics:** `wolfpack_regression_quantile_metrics.json`

Three separate LightGBM regressors, each trained with `objective="quantile"` at different alpha values:

| Quantile | Alpha | Purpose                   | Pinball Loss |
|----------|-------|---------------------------|--------------|
| P10      | 0.10  | Lower bound (optimistic)  | 0.2252       |
| P50      | 0.50  | Median prediction         | 0.4863       |
| P90      | 0.90  | Upper bound (pessimistic) | 0.2219       |

**Median (P50) Results:**

| Metric         | Value    |
|----------------|----------|
| Median RMSE    | 1.2285   |
| Median MAE     | 0.9727   |
| Median R²      | 0.9943   |
| Tier Accuracy  | 96.68%   |

**Prediction Interval (P10–P90, 80% target coverage):**

| Metric              | Value    |
|---------------------|----------|
| Coverage            | 75.9%    |
| Avg interval width  | 2.97 pts |
| Median interval width | 2.86 pts |

**Use case:** When the prediction interval spans a tier boundary (e.g., P10=73, P90=77), flag the case for human review instead of making an automatic trust decision.

---

## Evaluation Framework & Accuracy Metrics

### Classification Metrics

All classifiers are evaluated on the **held-out test set** (20,000 samples, never used during training or validation).

| Metric                | What It Measures                                          | Target    |
|-----------------------|-----------------------------------------------------------|-----------|
| **Test Accuracy**     | % of samples with correct tier prediction                 | > 95%     |
| **Macro F1**          | Harmonic mean of precision & recall, averaged across tiers (unweighted) | > 0.95 |
| **Val Accuracy**      | Accuracy on validation set (used for early stopping)      | > 94%     |
| **Confusion Matrix**  | Per-tier breakdown of true vs predicted labels            | —         |

**Summary:**

| Model               | Accuracy | Macro F1 | Params  | Framework   |
|---------------------|----------|----------|---------|-------------|
| GradientBoosting    | 96.24%   | 0.9623   | —       | sklearn     |
| PyTorch Transformer | 96.41%   | 0.9640   | 107.8K  | PyTorch     |
| **Keras Transformer** | **96.48%** | **0.9648** | 107.8K | TensorFlow |
| RTDL Transformer    | 96.45%   | 0.9645   | 900.9K  | rtdl        |

### Regression Metrics

| Metric               | What It Measures                                           | Target     |
|----------------------|------------------------------------------------------------|------------|
| **RMSE**             | Root mean squared error on trust score (0–100 scale)       | < 2.0      |
| **MAE**              | Mean absolute error on trust score                         | < 1.5      |
| **R²**               | Variance explained (1.0 = perfect)                         | > 0.99     |
| **Tier Accuracy**    | % of samples mapped to correct tier after score→tier conversion | > 95%  |
| **PI Coverage**      | % of true scores falling within P10–P90 interval (quantile only) | ~80% |

**Summary:**

| Model               | RMSE   | MAE    | R²     | Tier Acc | Notes              |
|---------------------|--------|--------|--------|----------|--------------------|
| Stacked Ensemble    | 1.1011 | 0.8770 | 0.9954 | 96.95%   | Production model   |
| **FT-Transformer**  | **1.0821** | **0.8629** | **0.9956** | **97.07%** | Best overall |
| Quantile Regression | 1.2285 | 0.9727 | 0.9943 | 96.68%   | + confidence intervals |

### Early Stopping Protocol

All deep learning models use early stopping to prevent overfitting:
- **Monitor:** Validation macro F1 (classification) or validation RMSE (regression)
- **Patience:** 10 epochs without improvement
- **Restore:** Best checkpoint weights are restored after training

---

## Error Analysis

### Classification Error Patterns

All models share the same systematic error pattern — **boundary cases near tier thresholds** (scores near 25, 50, 75):

| Error Type         | Per 5K Samples | Severity | Root Cause                                    |
|--------------------|----------------|----------|-----------------------------------------------|
| YELLOW ↔ RED       | 126–201        | Medium   | Users near score 50 with high threat intel    |
| YELLOW ↔ ORANGE    | 61–79          | Low      | Adjacent tiers with borderline scores         |
| GREEN ↔ RED        | 47–76          | High     | Rare but severe — extreme tier mismatch       |
| GREEN ↔ ORANGE     | 0              | —        | Never observed (tiers too far apart)          |

**Key observation:** No model confuses GREEN with ORANGE — the confusion is strictly between adjacent or near-adjacent tiers.

### Regression Residual Analysis

| Metric                                   | Value       |
|------------------------------------------|-------------|
| Mean absolute error                      | ~0.88 pts   |
| % predictions in correct tier            | 97%         |
| Remaining 3% distance from boundary      | < 2–3 pts   |
| Max residual (worst case)                | ~5 pts      |

### Per-Tier Regression Error

| Tier     | Avg MAE  | Tier Misclassification Rate | Notes                        |
|----------|----------|-----------------------------|------------------------------|
| GREEN    | 0.82 pts | 1.0%                        | Easiest to predict           |
| YELLOW   | 0.95 pts | 4.1%                        | Widest tier (25 pt range)    |
| ORANGE   | 0.91 pts | 3.1%                        | Some confusion with YELLOW   |
| RED      | 0.76 pts | 2.8%                        | Strong signal features help  |

### Why Boundary Errors Occur

Tier boundaries (25, 50, 75) are hard cutoffs applied to a continuous score. A user with true score 74.5 and predicted score 75.5 crosses the GREEN/YELLOW boundary despite only 1-point error. This is an inherent limitation of discretization.

### Mitigation Strategies

1. **Quantile regression intervals** — When P10–P90 spans a boundary, flag for human review
2. **Confidence-weighted enforcement** — Apply softer enforcement for scores within 3 points of a boundary
3. **Threshold tuning** — Adjust tier boundaries based on operational false positive/negative rates
4. **Exponential decay** — Session decay provides natural re-evaluation opportunities

---

## Model Comparison & Final Selection

### All 7 Models Side-by-Side

| # | Model | Task | Primary Metric | Tier Acc | Params | Deploy Complexity |
|---|-------|------|---------------|----------|--------|-------------------|
| 1 | GradientBoosting | Classification | F1=0.9623 | 96.24% | — | Low |
| 2 | PyTorch FT-Transformer | Classification | F1=0.9640 | 96.41% | 107.8K | Medium |
| 3 | Keras FT-Transformer | Classification | F1=0.9648 | 96.48% | 107.8K | Medium |
| 4 | RTDL FT-Transformer | Classification | F1=0.9645 | 96.45% | 900.9K | Medium |
| 5 | **Stacked Ensemble** | **Regression** | **R²=0.9954** | **96.95%** | **—** | **Low** |
| 6 | FT-Transformer Reg. | Regression | R²=0.9956 | 97.07% | 900.3K | High |
| 7 | Quantile Regression | Quantile | R²=0.9943 | 96.68% | — | Low |

### Final Decision

**Production model:** `wolfpack_regression_ensemble_model.pkl` (Stacked Ensemble Regressor)

| Factor              | Decision Rationale                                           |
|---------------------|--------------------------------------------------------------|
| Task formulation    | Regression > classification (preserves score granularity)    |
| Model architecture  | Ensemble > single model (robustness via model diversity)     |
| Runtime             | CPU-only > GPU (simpler infra, lower cost)                   |
| Deployment          | Single pickle > PyTorch (sklearn container, no CUDA)         |
| Performance gap     | R² 0.9954 vs 0.9956 = negligible (0.02% difference)         |
| Interpretability    | Base learner weights + SHAP available                        |
| Downstream compat   | Hunter model consumes raw float trust_score, not tier label  |

---

## Deployment — AWS SageMaker

**Directory:** `sagemaker_deploy/`

### Architecture

```
Client (boto3 invoke_endpoint)
    │
    ▼
SageMaker Endpoint: wolfpack-trust-score
    │
    ▼
sklearn Container (framework 1.4-2, Python 3.10)
    ├── nginx (port 8080, reverse proxy)
    ├── gunicorn (WSGI server)
    └── inference.py (custom handlers)
         ├── model_fn(model_dir)    → loads pickle, pre-builds scaler arrays, verifies base models
         ├── input_fn(request)      → parses JSON {"instances": [[22 floats], ...]}, validates shape
         ├── predict_fn(X, bundle)  → StandardScaler → 3 base learners → Ridge meta → clip(0,100)
         └── output_fn(preds)       → returns JSON {"predictions": [{trust_score, tier, base_scores}]}
```

### Inference Pipeline (predict_fn)

```python
# Step 1: Scale features
X_scaled = (X - scaler_mean) / scaler_scale

# Step 2: Base learner predictions
xgb_pred  = xgb_model.predict(X_scaled)    # → array of scores
lgbm_pred = lgbm_model.predict(X_scaled)   # → array of scores
cat_pred  = cat_model.predict(X_scaled)    # → array of scores

# Step 3: Stack into meta-learner input
base_preds = np.column_stack([xgb_pred, lgbm_pred, cat_pred])

# Step 4: Ridge meta-learner
ensemble_score = clip(ridge.predict(base_preds), 0, 100)

# Step 5: Map to tier
tier = "GREEN" if score >= 75 else "YELLOW" if score >= 50 else "ORANGE" if score >= 25 else "RED"
```

### Deployment Procedure (Step-by-Step)

**Step 1: Setup AWS prerequisites**
```bash
python sagemaker_deploy/setup_aws.py
```
- Prompts for AWS credentials (access key, secret key, region)
- Creates S3 bucket: `wolfpack-sagemaker-707075084132-us-east-1`
- Creates IAM role: `WolfPackSageMakerRole` (SageMaker + S3 read/write)
- Saves config to `sagemaker_deploy/deploy_config.json`

**Step 2: Package the model**
```bash
python sagemaker_deploy/package_model.py
```
- Downloads manylinux wheels for Linux container:
  - `xgboost==2.1.4` (manylinux_2_28)
  - `lightgbm==4.6.0` (manylinux_2_28)
  - `catboost==1.2.10` (manylinux2014)
- Packages into `model.tar.gz` (~309 MB):
  ```
  model.tar.gz/
  ├── wolfpack_regression_ensemble_model.pkl
  └── code/
      ├── inference.py
      ├── requirements.txt
      └── lib/
          ├── xgboost-2.1.4-cp310-manylinux_2_28.whl
          ├── lightgbm-4.6.0-cp310-manylinux_2_28.whl
          └── catboost-1.2.10-cp310-manylinux2014.whl
  ```

**Step 3: Deploy to SageMaker**
```bash
python sagemaker_deploy/deploy_to_sagemaker.py
```
- Uploads `model.tar.gz` to S3
- Creates SageMaker Model → Endpoint Configuration → Endpoint
- Uses `SKLearnModel(framework_version="1.4-2", py_version="py3")`
- Sets `container_startup_health_check_timeout=900` (15 minutes)
- Instance type: `ml.m5.xlarge`
- Auto-cleans stale endpoint configs from previous failed deployments
- Runs smoke test on success

**Step 4: Verify**
```bash
python sagemaker_deploy/test_endpoint.py
```
Tests 3 sample users (clean, suspicious, moderate) + batch test.

### Key Files

| File                         | Purpose                                        |
|------------------------------|-------------------------------------------------|
| `setup_aws.py`               | Interactive AWS setup (credentials, S3, IAM)   |
| `package_model.py`           | Build model.tar.gz with pre-baked wheels       |
| `deploy_to_sagemaker.py`     | Deploy model + create endpoint                 |
| `inference.py`               | SageMaker inference handlers (model_fn, input_fn, predict_fn, output_fn) |
| `test_endpoint.py`           | Endpoint integration tests (3 users + batch)   |
| `invoke_from_other_model.py` | Client library for downstream services (WolfPackTrustScorer class) |
| `deploy_config.json`         | Auto-generated AWS config (role ARN, bucket, region) |
| `requirements.txt`           | Container dependency pins (xgboost, lightgbm, catboost, numpy, sklearn) |
| `model.tar.gz`               | Packaged model artifact (~309 MB)              |

### SageMaker Troubleshooting & Lessons Learned

| Issue | Root Cause | Fix |
|-------|-----------|-----|
| Health check failure (1st attempt) | sklearn container `1.2-1` uses Python 3.8; `lightgbm>=4.6.0` requires 3.10+ | Use `framework_version="1.4-2"` (Python 3.10) |
| Health check timeout | Default timeout too short for pip install of large wheels | Set `container_startup_health_check_timeout=900` |
| Pip install timeout in container | Downloading wheels at container startup is slow | Pre-bake wheels into `model.tar.gz/code/lib/` |
| "Cannot create already existing endpoint configuration" | Stale config from previous failed deploy | Added auto-cleanup logic to delete stale configs |
| Wheel download fails | XGBoost/LightGBM need `manylinux_2_28`, CatBoost needs `manylinux2014` | Download in two groups with different platform tags |
| SageMaker SDK v3 missing SKLearnModel | v3 restructured API, `sagemaker.sklearn` removed | Pin `sagemaker>=2.200,<3` |

---

## Deployment — AWS EKS

**Directory:** `eks_deploy/`

### Architecture

```
Client (internal VPC)
    │
    ▼
AWS NLB (internal): k8s-zyberpol-wolfpack-0c267a7d53-c85f54cf1b9e1fd8.elb.us-east-1.amazonaws.com
    │
    ▼
K8s Service: wolfpack-trust-score (port 80 → 8080)
    │
    ├──▶ Pod #1: Flask + gunicorn (port 8080)
    │         ├── GET  /health   → {"status": "healthy"}
    │         ├── POST /predict  → run ensemble → {predictions, latency_ms}
    │         └── GET  /         → service info
    │
    └──▶ Pod #2: Flask + gunicorn (port 8080)   ← 2 replicas for HA
```

### Container

**Dockerfile:**
- Base: `python:3.10-slim`
- System deps: `libgomp1` (required by LightGBM)
- App: Flask + gunicorn (2 workers, 120s timeout)
- Model path: `/opt/ml/model/wolfpack_regression_ensemble_model.pkl`
- Port: 8080
- Health check: `GET /health` every 30s

### Kubernetes Manifests

**Deployment** (`k8s/deployment.yaml`):

| Property                  | Value                                                                 |
|---------------------------|-----------------------------------------------------------------------|
| Namespace                 | `zyberpol-ml`                                                         |
| Replicas                  | 2                                                                     |
| Image                     | `707075084132.dkr.ecr.us-east-1.amazonaws.com/zyberpol/wolfpack-trust-score:latest` |
| Labels                    | `app=wolfpack-trust-score, agent=wolfpack, team=data-science, app.kubernetes.io/part-of=zyberpol` |
| CPU requests/limits       | 250m / 1000m                                                          |
| Memory requests/limits    | 512Mi / 2Gi                                                           |
| Security context          | `runAsNonRoot=true, runAsUser=1000, readOnlyRootFilesystem=true, allowPrivilegeEscalation=false, drop ALL capabilities` |
| Volume mounts             | `/tmp` → emptyDir (for gunicorn worker temp files)                    |
| Readiness probe           | `GET /health:8080` — init 30s, period 10s, failure threshold 3        |
| Liveness probe            | `GET /health:8080` — init 60s, period 30s, failure threshold 3        |

**Service** (`k8s/service.yaml`):

| Property        | Value                                                  |
|-----------------|--------------------------------------------------------|
| Type            | LoadBalancer                                           |
| Namespace       | `zyberpol-ml`                                          |
| LB Type         | NLB (internal) via AWS annotation                      |
| Port mapping    | 80 → 8080                                              |

### Deployment Procedure

**Single command:**
```bash
python eks_deploy/deploy_to_eks.py [--cluster zyberpol-east] [--namespace zyberpol-ml]
```

This script automates the full pipeline:

| Step | Action                                              | AWS Service   |
|------|-----------------------------------------------------|---------------|
| 1    | Create ECR repository `zyberpol/wolfpack-trust-score` | ECR          |
| 2    | Upload build context (Dockerfile + app + model) to S3 | S3           |
| 3    | Build & push Docker image via CodeBuild              | CodeBuild     |
| 4    | Install `kubectl` binary (if not present)            | —             |
| 5    | Configure `kubectl` for EKS cluster                 | EKS           |
| 6    | Apply `k8s/deployment.yaml` + `k8s/service.yaml`    | EKS           |
| 7    | Wait for rollout completion + print NLB hostname     | EKS           |

No local Docker installation required — image is built remotely via CodeBuild.

### Key Files

| File                   | Purpose                                       |
|------------------------|-----------------------------------------------|
| `app.py`               | Flask application (predict, health, info)     |
| `Dockerfile`           | Container definition (python:3.10-slim)       |
| `deploy_to_eks.py`     | Full automated deployment (ECR→CodeBuild→K8s) |
| `buildspec.yml`        | AWS CodeBuild build specification             |
| `requirements.txt`     | Python dependencies                           |
| `k8s/deployment.yaml`  | K8s Deployment (2 pods, security hardened)     |
| `k8s/service.yaml`     | K8s Service (internal NLB on port 80)         |

---

## Endpoints, Clusters & Resource Paths

### Live Endpoints

| Endpoint | Platform | URL / Name | Access |
|----------|----------|------------|--------|
| Wolf Pack (SageMaker) | SageMaker | `wolfpack-trust-score` | Via `boto3 sagemaker-runtime invoke_endpoint` |
| Wolf Pack (EKS) | EKS (`zyberpol-ml` namespace) | `k8s-zyberpol-wolfpack-0c267a7d53-c85f54cf1b9e1fd8.elb.us-east-1.amazonaws.com` | Internal NLB (VPC only) |

### How to Call Each Endpoint

**SageMaker (from any AWS service with IAM permissions):**
```python
import boto3, json

runtime = boto3.client("sagemaker-runtime", region_name="us-east-1")
response = runtime.invoke_endpoint(
    EndpointName="wolfpack-trust-score",
    ContentType="application/json",
    Body=json.dumps({
        "instances": [[0.9, 0.3, 0.4, 0.2, 0.85, 1.0, 0.9, 0.1,
                        0.0, 0.0, 0.1, 1.0, 0.9, 0.1, 0.2, 0.9,
                        0.0, 0.0, 0.0, 0.05, 0.0, 0.0]]
    })
)
result = json.loads(response["Body"].read())
print(result)
# {"predictions": [{"trust_score": 90.03, "tier": "GREEN", "base_scores": {...}}]}
```

**EKS (from within VPC):**
```bash
curl -X POST \
  http://k8s-zyberpol-wolfpack-0c267a7d53-c85f54cf1b9e1fd8.elb.us-east-1.amazonaws.com/predict \
  -H "Content-Type: application/json" \
  -d '{"instances": [[0.9, 0.3, 0.4, 0.2, 0.85, 1.0, 0.9, 0.1, 0.0, 0.0, 0.1, 1.0, 0.9, 0.1, 0.2, 0.9, 0.0, 0.0, 0.0, 0.05, 0.0, 0.0]]}'
```

**Using the client library (recommended for downstream models):**
```python
from sagemaker_deploy.invoke_from_other_model import WolfPackTrustScorer

scorer = WolfPackTrustScorer(endpoint_name="wolfpack-trust-score")
result = scorer.predict(features=[0.9, 0.3, 0.4, ...])  # 22 features
print(result["trust_score"], result["tier"])

# Batch
results = scorer.predict_batch(instances=[[...], [...], [...]])
```

### AWS Resources

| Resource | Type | Value |
|----------|------|-------|
| AWS Account | — | `707075084132` |
| Region | — | `us-east-1` |
| EKS Cluster | EKS | `zyberpol-east` |
| K8s Namespace | — | `zyberpol-ml` |
| IAM Role (SageMaker) | IAM | `arn:aws:iam::707075084132:role/WolfPackSageMakerRole` |
| S3 Bucket (models) | S3 | `wolfpack-sagemaker-707075084132-us-east-1` |
| ECR Repository | ECR | `707075084132.dkr.ecr.us-east-1.amazonaws.com/zyberpol/wolfpack-trust-score` |
| SageMaker Endpoint | SageMaker | `wolfpack-trust-score` |
| SageMaker Instance | — | `ml.m5.xlarge` |
| NLB (EKS) | NLB | `k8s-zyberpol-wolfpack-0c267a7d53-c85f54cf1b9e1fd8.elb.us-east-1.amazonaws.com` |
| CloudWatch Logs (SM) | CloudWatch | `/aws/sagemaker/Endpoints/wolfpack-trust-score` |

### AWS Console Paths

| Resource | Console Path |
|----------|-------------|
| SageMaker Endpoint | SageMaker → Inference → Endpoints → `wolfpack-trust-score` |
| SageMaker Model | SageMaker → Inference → Models → `wolfpack-ensemble-*` |
| SageMaker Endpoint Config | SageMaker → Inference → Endpoint configurations → `wolfpack-trust-score` |
| EKS Cluster | EKS → Clusters → `zyberpol-east` |
| EKS Workloads | EKS → Clusters → `zyberpol-east` → Resources → Workloads → `wolfpack-trust-score` |
| ECR Repository | ECR → Repositories → `zyberpol/wolfpack-trust-score` |
| S3 Model Artifacts | S3 → `wolfpack-sagemaker-707075084132-us-east-1` |
| CloudWatch Logs | CloudWatch → Log groups → `/aws/sagemaker/Endpoints/wolfpack-trust-score` |

### NATS Mifal Bus (Port 4222)

| Subject                  | Publisher   | Subscriber              |
|--------------------------|-------------|-------------------------|
| `wolf_pack.trust.score`  | Wolf Pack   | Hunter Agent            |
| `hunter.threat.assessed` | Hunter      | Fire Factory, Fire Weaver, Alchemist |

---

## API Reference

Both SageMaker and EKS endpoints accept the same request/response format.

### Request Format

```json
POST /invocations   (SageMaker)
POST /predict       (EKS)
Content-Type: application/json

{
  "instances": [
    [0.9, 0.3, 0.4, 0.2, 0.85, 1.0, 0.9, 0.1, 0.0, 0.0,
     0.1, 1.0, 0.9, 0.1, 0.2, 0.9, 0.0, 0.0, 0.0, 0.05,
     0.0, 0.0]
  ]
}
```

- Each instance is a **22-element float array** in feature order f01–f22
- Batch: pass multiple arrays in `"instances"`
- EKS also accepts `"data"` as an alias for `"instances"`

### Response Format

```json
{
  "predictions": [
    {
      "trust_score": 90.03,
      "tier": "GREEN",
      "base_scores": {
        "XGBoost": 89.45,
        "LightGBM": 88.92,
        "CatBoost": 90.31
      }
    }
  ],
  "latency_ms": 3.42       // EKS only
}
```

### EKS Endpoints

| Endpoint     | Method | Description                        | Status Codes     |
|--------------|--------|------------------------------------|------------------|
| `/health`    | GET    | Health check → `{"status": "healthy"}` | 200 / 503    |
| `/predict`   | POST   | Run inference                      | 200 / 400 / 503  |
| `/`          | GET    | Service info (name, version, endpoints) | 200          |

### Error Responses

| HTTP Code | Meaning                                    | Example                              |
|-----------|--------------------------------------------|--------------------------------------|
| 400       | Bad request (missing instances, wrong shape) | `{"error": "Expected 22 features per instance, got 15"}` |
| 503       | Model not loaded                           | `{"error": "model not loaded"}`      |

---

## File Inventory

```
darwin/
├── README.md                                  ← Darwin Engine docs (separate project)
├── WOLFPACK_README.md                         ← This file
│
│   ── Dataset ──
├── build_realworld_wolfpack_dataset.py        ← Dataset generation (100K rows, calibrated to 5 source datasets)
├── wolfpack_realworld_dataset.csv             ← Generated dataset (100K × 26)
├── wolfpack_training_dataset.csv              ← Training subset
│
│   ── Scoring Engine ──
├── wolf_pack_scoring_model.py                 ← Production scoring formula + ROMA agent + decay function
│                                                 (5-domain weights, per-feature weights, LLM router,
│                                                  exponential decay, enforcement actions)
│
│   ── Classification Models ──
├── train_wolfpack_ml_model.py                 ← GradientBoosting classifier (sklearn)
├── train_wolfpack_dl_model.py                 ← PyTorch FT-Transformer classifier
├── train_wolfpack_keras_model.py              ← Keras/TF FT-Transformer classifier
├── train_wolfpack_rtdl_model.py               ← RTDL FT-Transformer classifier (NeurIPS 2021)
│
│   ── Regression Models ──
├── train_wolfpack_regression_ensemble.py      ← Stacked Ensemble: XGB+LGBM+CB+Ridge (PRODUCTION)
├── train_wolfpack_regression_transformer.py   ← FT-Transformer regressor (d_out=1, Huber loss)
├── train_wolfpack_regression_quantile.py      ← LightGBM quantile regression (P10/P50/P90)
│
│   ── Model Artifacts ──
├── wolfpack_ml_model.pkl                      ← GradientBoosting model
├── wolfpack_dl_model.pt                       ← PyTorch transformer
├── wolfpack_keras_model.keras                 ← Keras transformer
├── wolfpack_rtdl_model.pt                     ← RTDL transformer
├── wolfpack_regression_ensemble_model.pkl     ← Stacked ensemble ★ DEPLOYED TO SAGEMAKER + EKS
├── wolfpack_regression_transformer_model.pt   ← Transformer regressor
├── wolfpack_regression_quantile_model.pkl     ← Quantile regression
│
│   ── Metrics (JSON) ──
├── wolfpack_ml_metrics.json                   ← GradientBoosting: acc=96.24%, F1=0.9623
├── wolfpack_dl_metrics.json                   ← PyTorch: acc=96.41%, F1=0.9640
├── wolfpack_keras_metrics.json                ← Keras: acc=96.48%, F1=0.9648
├── wolfpack_rtdl_metrics.json                 ← RTDL: acc=96.45%, F1=0.9645
├── wolfpack_regression_ensemble_metrics.json  ← Ensemble: RMSE=1.1011, R²=0.9954
├── wolfpack_regression_transformer_metrics.json ← Transformer: RMSE=1.0821, R²=0.9956
├── wolfpack_regression_quantile_metrics.json  ← Quantile: RMSE=1.2285, R²=0.9943
│
│   ── Reports (Human-Readable) ──
├── wolfpack_ml_report.txt
├── wolfpack_dl_report.txt
├── wolfpack_keras_report.txt
├── wolfpack_rtdl_report.txt
├── wolfpack_regression_ensemble_report.txt
├── wolfpack_regression_transformer_report.txt
├── wolfpack_regression_quantile_report.txt
│
│   ── Jupyter Notebooks ──
├── wolf_pack_scoring_model.ipynb
├── train_wolfpack_ml_model.ipynb
├── train_wolfpack_dl_model.ipynb
├── train_wolfpack_keras_model.ipynb
├── train_wolfpack_rtdl_model.ipynb
├── train_wolfpack_regression_ensemble.ipynb
├── train_wolfpack_regression_transformer.ipynb
├── train_wolfpack_regression_quantile.ipynb
├── build_realworld_wolfpack_dataset.ipynb
│
│   ── Notebook Generators ──
├── build_all_notebooks.py
├── build_dl_notebook.py
├── build_wolfpack_notebook.py
├── build_notebooks_from_scripts.py
├── build_regression_notebooks.py
│
│   ── SageMaker Deployment ──
├── sagemaker_deploy/
│   ├── setup_aws.py                           ← Interactive AWS setup (credentials, S3, IAM)
│   ├── package_model.py                       ← Build model.tar.gz (~309 MB with wheels)
│   ├── deploy_to_sagemaker.py                 ← Deploy endpoint (sklearn 1.4-2, ml.m5.xlarge)
│   ├── inference.py                           ← SageMaker handlers (model_fn, input_fn, predict_fn, output_fn)
│   ├── test_endpoint.py                       ← Integration tests (3 users + batch)
│   ├── invoke_from_other_model.py             ← Client library (WolfPackTrustScorer class)
│   ├── requirements.txt                       ← xgboost==2.1.4, lightgbm==4.6.0, catboost==1.2.10
│   ├── deploy_config.json                     ← Auto-generated AWS config
│   └── model.tar.gz                           ← Packaged model artifact
│
│   ── EKS Deployment ──
├── eks_deploy/
│   ├── app.py                                 ← Flask server (/predict, /health, /)
│   ├── Dockerfile                             ← python:3.10-slim + gunicorn + libgomp1
│   ├── deploy_to_eks.py                       ← Full automation (ECR→CodeBuild→kubectl apply)
│   ├── buildspec.yml                          ← CodeBuild build specification
│   ├── requirements.txt                       ← Python dependencies
│   └── k8s/
│       ├── deployment.yaml                    ← 2 pods, security-hardened, zyberpol-ml namespace
│       └── service.yaml                       ← Internal NLB, port 80→8080
│
│   ── Documentation ──
├── WOLFPACK_DATASETS_AND_FEATURES.docx
├── WOLFPACK_FILE_INVENTORY.docx
├── WOLFPACK_MODEL_GUIDE.docx
├── WOLFPACK_SCORING_SPEC.docx
├── ZYBERPOL_ARCHITECTURE.docx
├── ZYBERPOL_ARCHITECTURE.md
└── ZYBERPOL_ARCHITECTURE.pdf
```

---

## How to Reproduce

### Prerequisites

```bash
# Core ML
pip install numpy pandas scikit-learn xgboost lightgbm catboost

# Transformer models (optional — only needed to retrain DL models)
pip install torch rtdl          # PyTorch + RTDL
pip install tensorflow          # Keras model

# AWS deployment
pip install boto3 'sagemaker>=2.200,<3'

# macOS only: XGBoost/LightGBM need libomp
brew install libomp
# If no brew: download from homebrew bottles API and set DYLD_LIBRARY_PATH
```

### Step 1: Generate Dataset

```bash
python build_realworld_wolfpack_dataset.py
# Output: wolfpack_realworld_dataset.csv (100K rows, 26 columns)
# Time: ~30 seconds
```

### Step 2: Train Classification Models (optional)

```bash
python train_wolfpack_ml_model.py       # ~2 min  → wolfpack_ml_model.pkl
python train_wolfpack_dl_model.py       # ~5 min  → wolfpack_dl_model.pt
python train_wolfpack_keras_model.py    # ~5 min  → wolfpack_keras_model.keras
python train_wolfpack_rtdl_model.py     # ~8 min  → wolfpack_rtdl_model.pt
```

### Step 3: Train Regression Models

```bash
python train_wolfpack_regression_ensemble.py      # ~3 min  → wolfpack_regression_ensemble_model.pkl
python train_wolfpack_regression_transformer.py   # ~5 min  → wolfpack_regression_transformer_model.pt
python train_wolfpack_regression_quantile.py      # ~2 min  → wolfpack_regression_quantile_model.pkl
```

### Step 4: Generate Notebooks (optional)

```bash
python build_regression_notebooks.py    # Generates 3 regression notebooks
python build_all_notebooks.py           # Generates all notebooks
```

### Step 5: Deploy to SageMaker

```bash
python sagemaker_deploy/setup_aws.py              # One-time: credentials, S3, IAM
python sagemaker_deploy/package_model.py           # Build model.tar.gz (~5 min download)
python sagemaker_deploy/deploy_to_sagemaker.py     # Deploy (~10-15 min)
python sagemaker_deploy/test_endpoint.py           # Verify
```

### Step 6: Deploy to EKS

```bash
python eks_deploy/deploy_to_eks.py                 # Full automated (~10 min)
```

---

## Troubleshooting

### Common Issues

| Symptom | Cause | Fix |
|---------|-------|-----|
| `ImportError: libgomp.so.1` (macOS) | XGBoost/LightGBM need OpenMP | `brew install libomp` or set `DYLD_LIBRARY_PATH` |
| `ModuleNotFoundError: sagemaker.sklearn` | SageMaker SDK v3 removed `sagemaker.sklearn` | `pip install 'sagemaker>=2.200,<3'` |
| SageMaker health check fails | Wrong container version or missing wheels | Use `framework_version="1.4-2"`, pre-bake wheels, set timeout to 900s |
| "Cannot create already existing endpoint configuration" | Stale config from failed deploy | Delete via AWS console or `deploy_to_sagemaker.py` auto-cleans |
| Pickle load error in SageMaker | Python version mismatch (local 3.9 vs container 3.10) | Retrain model with Python 3.10, or use protocol=4 |
| EKS pods in CrashLoopBackOff | Model file not baked into Docker image | Ensure `wolfpack_regression_ensemble_model.pkl` is in build context |
| kubectl: connection refused | kubeconfig not set for cluster | `aws eks update-kubeconfig --name zyberpol-east --region us-east-1` |

### Checking Endpoint Health

**SageMaker:**
```python
import boto3
sm = boto3.client("sagemaker", region_name="us-east-1")
status = sm.describe_endpoint(EndpointName="wolfpack-trust-score")
print(status["EndpointStatus"])  # Should be "InService"
```

**EKS:**
```bash
kubectl get pods -n zyberpol-ml -l app=wolfpack-trust-score
kubectl logs -n zyberpol-ml -l app=wolfpack-trust-score --tail=50
curl http://k8s-zyberpol-wolfpack-0c267a7d53-c85f54cf1b9e1fd8.elb.us-east-1.amazonaws.com/health
```

### Reading CloudWatch Logs (SageMaker)

```python
import boto3
logs = boto3.client("logs", region_name="us-east-1")
streams = logs.describe_log_streams(
    logGroupName="/aws/sagemaker/Endpoints/wolfpack-trust-score",
    orderBy="LastEventTime", descending=True, limit=1
)
events = logs.get_log_events(
    logGroupName="/aws/sagemaker/Endpoints/wolfpack-trust-score",
    logStreamName=streams["logStreams"][0]["logStreamName"],
    limit=100
)
for e in events["events"]:
    print(e["message"])
```

---

## Production Considerations

| Area                  | Recommendation                                                                 |
|-----------------------|--------------------------------------------------------------------------------|
| **Model Monitoring**  | Track prediction distribution (especially RED tier ratio) for data drift. Alert if RED rate exceeds 2x baseline. |
| **Threshold Tuning**  | Tier boundaries (25/50/75) should be recalibrated quarterly based on security false positive/negative rates. |
| **Feature Freshness** | Top features (`f20_failed_logins`, `f05_device_compliance`, `f16_ip_reputation`) are time-sensitive. Stale inputs degrade accuracy significantly. |
| **Boundary Cases**    | For predictions within 3 points of a tier boundary, use quantile model's PI to flag for human review. |
| **Scaling (EKS)**     | Add HorizontalPodAutoscaler targeting 70% CPU. Current: 2 replicas → scale to 4-8 under load. |
| **Scaling (SageMaker)** | Add auto-scaling policy via `register_scalable_target` + `put_scaling_policy`. |
| **Latency SLA**       | EKS: ~40ms p50 per prediction. SageMaker: ~50–90ms p50 (includes network). |
| **Explainability**    | Use SHAP on the stacked ensemble for per-prediction explanations to SOC team. |
| **Retraining**        | Retrain monthly on fresh data. Monitor concept drift via R² on rolling holdout. |
| **Session Decay**     | Exponential decay (λ=0.98, 5 min interval) ensures re-evaluation. Floor at 30 prevents permanent RED without re-auth. |
| **Downstream**        | Hunter model consumes `trust_score` + `tier` + `base_scores` + disagreement signal. Update Hunter if Wolf Pack response schema changes. |

---

**Author:** Wolf Pack ML Engineering Team
**AWS Account:** `707075084132`
**Region:** `us-east-1`
**SageMaker Endpoint:** `wolfpack-trust-score`
**EKS Cluster:** `zyberpol-east`
**EKS Namespace:** `zyberpol-ml`
**EKS NLB:** `k8s-zyberpol-wolfpack-0c267a7d53-c85f54cf1b9e1fd8.elb.us-east-1.amazonaws.com`
**ECR Image:** `707075084132.dkr.ecr.us-east-1.amazonaws.com/zyberpol/wolfpack-trust-score:latest`
