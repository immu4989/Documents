# Wolf Pack Identity Trust Scoring Model

**Project:** Zyberpol Iron Dome — Wolf Pack Agent
**Purpose:** Real-time identity trust scoring for zero-trust security enforcement
**Version:** 3.0.0

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Dataset Generation](#dataset-generation)
4. [Feature Engineering](#feature-engineering)
5. [Classification Models](#classification-models)
6. [Regression Models](#regression-models)
7. [Model Comparison & Selection](#model-comparison--selection)
8. [Error Analysis](#error-analysis)
9. [Deployment — AWS SageMaker](#deployment--aws-sagemaker)
10. [Deployment — AWS EKS](#deployment--aws-eks)
11. [API Reference](#api-reference)
12. [Project Structure](#project-structure)
13. [Getting Started](#getting-started)
14. [Production Considerations](#production-considerations)

---

## Overview

Wolf Pack is the ML scoring engine for the Zyberpol Iron Dome security agent. It evaluates users and entities across 22 security features spanning five domains and assigns a **trust score (0–100)** mapped to four actionable tiers:

| Tier       | Score Range | Action          |
|------------|-------------|-----------------|
| **GREEN**  | >= 75       | Allow           |
| **YELLOW** | 50 – 74     | Caution / MFA   |
| **ORANGE** | 25 – 49     | Elevated risk   |
| **RED**    | < 25        | Block / Alert   |

The project includes **4 classification models** (tier prediction) and **3 regression models** (continuous score prediction), with production deployment to both AWS SageMaker and AWS EKS.

---

## Architecture

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
```

---

## Dataset Generation

**Script:** `build_realworld_wolfpack_dataset.py`
**Output:** `wolfpack_realworld_dataset.csv` — 100,000 rows, 26 columns

The synthetic dataset is calibrated against five open-source security datasets:

| Domain           | Source Datasets                                      |
|------------------|------------------------------------------------------|
| HR               | CERT Insider Threat v6.2                             |
| Device           | UNSW-NB15 + CIC-IDS-2017                            |
| Behavioral       | LANL Comprehensive Cyber-Security Events             |
| Threat Intel     | MaxMind GeoLite2 + AbuseIPDB + AlienVault OTX       |
| Event Context    | Splunk BOTS v3                                       |

### Trust Score Computation

The trust score (0–100) is the sum of five domain sub-scores (each 0–20 points):

- **HR (20 pts):** Active status (5) + tenure (3) + low sensitivity (5) + recent background check (5)
- **Device (20 pts):** Compliance (7) + managed device (4) + patched (5) + recent scan (4)
- **Behavioral (20 pts):** Business hours (3) + weekday (1.5) + low geo-risk (4) + known device (3) + travel plausibility (3.5) + low velocity (5)
- **Threat Intel (20 pts):** Good IP reputation (8) + no anonymizer (4) + not malicious (4) + no IOC matches (4)
- **Event Context (20 pts):** Few failed logins (8) + no MFA fatigue (6) + no privilege escalation (6)

### Dataset Split

| Split | Rows   | Purpose                                  |
|-------|--------|------------------------------------------|
| Train | 70,000 | Model training                           |
| Val   | 10,000 | Hyperparameter tuning / early stopping   |
| Test  | 20,000 | Final evaluation                         |

---

## Feature Engineering

22 features across 5 security domains, all normalized to [0, 1]:

| #   | Feature                | Domain       | Description                              |
|-----|------------------------|--------------|------------------------------------------|
| f01 | hr_status              | HR           | Active employment status                 |
| f02 | tenure                 | HR           | Normalized tenure (0–4000 days)          |
| f03 | role_sensitivity       | HR           | Role sensitivity / privilege level       |
| f04 | background_check       | HR           | Recency of background check              |
| f05 | device_compliance      | Device       | Device compliance score                  |
| f06 | device_managed         | Device       | Corporate-managed device (binary)        |
| f07 | compliance_score       | Device       | Patch / update compliance                |
| f08 | scan_recency           | Device       | Last security scan recency               |
| f09 | time_of_day            | Behavioral   | Off-hours indicator (1=deep night)       |
| f10 | weekend                | Behavioral   | Weekend indicator                        |
| f11 | geo_risk               | Behavioral   | Geographic risk (high: NG, RU, KP, IR)   |
| f12 | known_device           | Behavioral   | Previously seen device                   |
| f13 | travel_plausibility    | Behavioral   | Travel speed plausibility                |
| f14 | velocity_minute        | Behavioral   | Login velocity (per minute)              |
| f15 | velocity_hour          | Behavioral   | Login velocity (per hour)                |
| f16 | ip_reputation          | Threat Intel | IP reputation score (0=bad, 1=good)      |
| f17 | anonymizer             | Threat Intel | Tor / VPN anonymizer usage               |
| f18 | known_malicious        | Threat Intel | Known malicious IP flag                  |
| f19 | ioc_matches            | Threat Intel | Indicator of compromise matches          |
| f20 | failed_logins          | Event Ctx    | Failed login ratio                       |
| f21 | mfa_fatigue            | Event Ctx    | MFA fatigue bombing indicator            |
| f22 | priv_escalation        | Event Ctx    | Privilege escalation attempt             |

### Top Feature Importances (from GradientBoosting)

1. `f20_failed_logins` — 22.3%
2. `f05_device_compliance` — 21.2%
3. `f16_ip_reputation` — 16.7%
4. `f11_geo_risk` — 12.7%
5. `f04_background_check` — 7.8%

---

## Classification Models

Four models predict the trust tier (GREEN / YELLOW / ORANGE / RED) as a 4-class classification task.

### 1. GradientBoosting Classifier (sklearn)

**Script:** `train_wolfpack_ml_model.py`
**Artifact:** `wolfpack_ml_model.pkl`

| Metric         | Value   |
|----------------|---------|
| Test Accuracy  | 96.24%  |
| Macro F1       | 0.9623  |
| Val Accuracy   | 95.68%  |

Hyperparameters: `n_estimators=200, max_depth=4, learning_rate=0.1`

**Confusion Matrix (Test Set):**
```
          GREEN  YELLOW  ORANGE   RED
GREEN     4947      0        0    53
YELLOW       0   4756       61   183
ORANGE       0    154     4846     0
RED        150    152        0  4698
```

### 2. PyTorch FT-Transformer Lite

**Script:** `train_wolfpack_dl_model.py`
**Artifact:** `wolfpack_dl_model.pt`

| Metric         | Value    |
|----------------|----------|
| Test Accuracy  | 96.41%   |
| Macro F1       | 0.9640   |
| Model Params   | 107,844  |
| Epochs         | 27 (early stopped from 60) |

Architecture: `d_model=64, 8 heads, 3 layers, ff_hidden=128, dropout=0.1`

```
22 features → Per-feature linear tokenizer (22 tokens, d=64)
           → Prepend [CLS] token (23 tokens total)
           → 3x Transformer encoder (8 heads, pre-norm)
           → [CLS] → LayerNorm → MLP head → 4-class softmax
```

**Confusion Matrix (Test Set):**
```
          GREEN  YELLOW  ORANGE   RED
GREEN     4953      0        0    47
YELLOW       0   4737       62   201
ORANGE       0    134     4866     0
RED        149    126        0  4725
```

### 3. Keras/TensorFlow FT-Transformer Lite

**Script:** `train_wolfpack_keras_model.py`
**Artifact:** `wolfpack_keras_model.keras`

| Metric         | Value    |
|----------------|----------|
| Test Accuracy  | 96.48%   |
| Macro F1       | 0.9648   |
| Model Params   | 107,844  |
| Epochs         | 27       |
| Framework      | TensorFlow 2.20.0 |

Same architecture as PyTorch variant. Uses `AdamW` with `CosineDecay` schedule, `weight_decay=1e-4`.

**Confusion Matrix (Test Set):**
```
          GREEN  YELLOW  ORANGE   RED
GREEN     4924      0        0    76
YELLOW       0   4770       79   151
ORANGE       0    104     4896     0
RED        129    165        0  4706
```

### 4. Official RTDL FT-Transformer

**Script:** `train_wolfpack_rtdl_model.py`
**Artifact:** `wolfpack_rtdl_model.pt`

| Metric         | Value    |
|----------------|----------|
| Test Accuracy  | 96.45%   |
| Macro F1       | 0.9645   |
| Model Params   | 900,868  |
| Epochs         | 35 (early stopped from 60) |

Uses `rtdl.FTTransformer.make_default()` with paper-recommended hyperparameters (Gorishniy et al., NeurIPS 2021). `n_blocks=3, lr=0.0001, batch_size=256`.

**Confusion Matrix (Test Set):**
```
          GREEN  YELLOW  ORANGE   RED
GREEN     4927      0        0    73
YELLOW       0   4784       63   153
ORANGE       0    122     4878     0
RED        129    170        0  4701
```

---

## Regression Models

Three models predict the continuous trust score (0–100), which is then mapped to tier thresholds.

### 1. Stacked Ensemble Regressor (Production Model)

**Script:** `train_wolfpack_regression_ensemble.py`
**Artifact:** `wolfpack_regression_ensemble_model.pkl`

**Architecture:** 5-fold out-of-fold stacking with Ridge meta-learner

| Layer   | Model     | Test RMSE | Test R²  | Tier Acc |
|---------|-----------|-----------|----------|----------|
| Base #1 | XGBoost   | 1.1476    | 0.9950   | 96.88%   |
| Base #2 | LightGBM  | 1.1636    | 0.9949   | 96.75%   |
| Base #3 | CatBoost  | 1.1028    | 0.9954   | 96.94%   |
| **Meta**| **Ridge** | **1.1011**| **0.9954**| **96.95%**|

**Ridge meta-learner weights:** XGBoost=10.2%, LightGBM=7.8%, CatBoost=82.2%

Hyperparameters (all base learners): `n_estimators=500, max_depth=6, learning_rate=0.05, subsample=0.8, colsample_bytree=0.8, early_stopping=20`

### 2. FT-Transformer Regressor

**Script:** `train_wolfpack_regression_transformer.py`
**Artifact:** `wolfpack_regression_transformer_model.pt`

| Metric         | Value    |
|----------------|----------|
| Test RMSE      | 1.0821   |
| Test MAE       | 0.8629   |
| Test R²        | 0.9956   |
| Tier Accuracy  | 97.07%   |
| Loss Function  | Huber (delta=5.0) |
| Model Params   | 900,289  |
| Epochs         | 23 (early stopped from 60) |

Uses `rtdl.FTTransformer.make_default(d_out=1)` with target standardization (mean=68.58, std=16.21).

### 3. Quantile Regression (Uncertainty-Aware)

**Script:** `train_wolfpack_regression_quantile.py`
**Artifact:** `wolfpack_regression_quantile_model.pkl`

| Metric                  | Value   |
|-------------------------|---------|
| Median RMSE             | 1.2285  |
| Median MAE              | 0.9727  |
| Median R²               | 0.9943  |
| Tier Accuracy           | 96.68%  |
| 80% PI Coverage         | 75.9%   |
| Avg Interval Width      | 2.97 pts|

Three LightGBM regressors predict P10 / P50 / P90 quantiles, providing a confidence band around each score. Useful for flagging boundary cases where the prediction interval spans a tier threshold.

**Pinball Losses:** P10=0.2252, P50=0.4863, P90=0.2219

---

## Model Comparison & Selection

### Classification Results

| Model               | Accuracy | Macro F1 | Params  | Framework   |
|---------------------|----------|----------|---------|-------------|
| GradientBoosting    | 96.24%   | 0.9623   | —       | sklearn     |
| PyTorch Transformer | 96.41%   | 0.9640   | 107.8K  | PyTorch     |
| **Keras Transformer** | **96.48%** | **0.9648** | 107.8K | TensorFlow |
| RTDL Transformer    | 96.45%   | 0.9645   | 900.9K  | rtdl        |

### Regression Results

| Model               | RMSE   | R²     | Tier Acc | Notes              |
|---------------------|--------|--------|----------|--------------------|
| Stacked Ensemble    | 1.1011 | 0.9954 | 96.95%   | Production model   |
| **FT-Transformer**  | **1.0821** | **0.9956** | **97.07%** | Best overall |
| Quantile Regression | 1.2285 | 0.9943 | 96.68%   | Confidence intervals |

### Why the Stacked Ensemble Was Chosen for Production

The **Stacked Ensemble** (`wolfpack_regression_ensemble_model.pkl`) was selected for deployment despite the FT-Transformer having slightly better metrics:

1. **No GPU required** — Uses sklearn-compatible models (XGBoost, LightGBM, CatBoost, Ridge), runs on CPU-only instances
2. **Simple deployment** — Single pickle file, no PyTorch/CUDA runtime dependency
3. **Near-best performance** — R²=0.9954 vs transformer's 0.9956 (negligible difference)
4. **Interpretability** — Base learner weights are inspectable; SHAP values available for individual predictions
5. **Low latency** — Tree ensembles predict in ~1ms vs transformer's ~5ms per batch

The FT-Transformer is better suited for GPU-backed batch scoring pipelines.

---

## Error Analysis

### Where Models Struggle

All models share the same error pattern — **boundary cases near tier thresholds** (scores near 25, 50, 75):

- **YELLOW ↔ RED:** 126–201 misclassifications per 5K samples — the most common error. Users near score=50 with high threat intel signals.
- **YELLOW ↔ ORANGE:** 61–79 per 5K — Adjacent tiers with borderline scores.
- **GREEN ↔ RED:** 47–76 per 5K — Rare but severe (extreme tier mismatch).

### Regression Residuals

- Mean absolute error: ~0.88 points on a 0–100 scale
- 97% of predictions land in the correct tier
- Remaining 3% are within 2–3 points of the tier boundary

### Mitigation

The **quantile regression model** addresses boundary uncertainty by providing P10–P90 prediction intervals. When the interval spans a tier boundary (e.g., P10=73, P90=77), downstream systems can flag the case for human review rather than making an automatic trust decision.

---

## Deployment — AWS SageMaker

**Directory:** `sagemaker_deploy/`
**Endpoint:** `wolfpack-trust-score`
**Region:** `us-east-1`
**Instance:** `ml.m5.xlarge`

### Architecture

```
Client → SageMaker Endpoint → sklearn Container (1.4-2, Python 3.10)
                                  ├── nginx (port 8080)
                                  ├── gunicorn
                                  └── inference.py
                                       ├── model_fn()   → load pickle
                                       ├── input_fn()   → parse JSON
                                       ├── predict_fn() → run ensemble
                                       └── output_fn()  → return JSON
```

### Deployment Procedure

**Step 1: Setup AWS prerequisites**
```bash
python sagemaker_deploy/setup_aws.py
```
Interactive setup that:
- Configures AWS credentials
- Creates S3 bucket (`wolfpack-sagemaker-{account_id}-{region}`)
- Creates IAM role (`WolfPackSageMakerRole`) with SageMaker + S3 permissions
- Saves config to `sagemaker_deploy/deploy_config.json`

**Step 2: Package the model**
```bash
python sagemaker_deploy/package_model.py
```
- Bundles model pickle + inference code + manylinux wheels into `model.tar.gz`
- Pre-bakes XGBoost, LightGBM, CatBoost wheels for the Linux container
- Output: `sagemaker_deploy/model.tar.gz` (~309 MB)

**Step 3: Deploy to SageMaker**
```bash
python sagemaker_deploy/deploy_to_sagemaker.py
```
- Uploads `model.tar.gz` to S3
- Creates SageMaker model + endpoint configuration + endpoint
- Uses `SKLearnModel` with `framework_version="1.4-2"`, `py_version="py3"`
- Sets `container_startup_health_check_timeout=900` (15 min)
- Auto-cleans stale endpoints from failed deployments
- Runs smoke test on deployment completion

**Step 4: Verify**
```bash
python sagemaker_deploy/test_endpoint.py
```

### Key Files

| File                    | Purpose                                        |
|-------------------------|-------------------------------------------------|
| `setup_aws.py`          | Interactive AWS setup (credentials, S3, IAM)   |
| `package_model.py`      | Build model.tar.gz with pre-baked wheels       |
| `deploy_to_sagemaker.py`| Deploy model + create endpoint                 |
| `inference.py`          | SageMaker inference handlers                   |
| `test_endpoint.py`      | Endpoint integration tests                     |
| `invoke_from_other_model.py` | Client library for downstream services    |
| `deploy_config.json`    | Auto-generated AWS config                      |
| `requirements.txt`      | Container dependency pins                      |

### Lessons Learned

- **sklearn container version matters:** `1.2-1` uses Python 3.8, which is incompatible with `lightgbm>=4.6.0`. Use `1.4-2` (Python 3.10+).
- **Pre-bake wheels:** The sklearn container installs from `code/lib/` at startup. Without pre-baked wheels, pip install can timeout during health check.
- **Platform tags differ:** XGBoost/LightGBM need `manylinux_2_28`, CatBoost needs `manylinux2014`.
- **Health check timeout:** Default is too short for large dependency installs. Set to 900s.

---

## Deployment — AWS EKS

**Directory:** `eks_deploy/`
**Cluster:** `zyberpol-east`
**Namespace:** `default`
**ECR Image:** `707075084132.dkr.ecr.us-east-1.amazonaws.com/zyberpol/wolfpack-trust-score:latest`

### Architecture

```
Client → NLB (internal) → K8s Service (port 80)
              → Pod #1 (Flask on port 8080)
              → Pod #2 (Flask on port 8080)  ← 2 replicas for HA
```

### Deployment Procedure

**Single command:**
```bash
python eks_deploy/deploy_to_eks.py
```

This script automates the full pipeline:
1. Creates ECR repository `zyberpol/wolfpack-trust-score` (if needed)
2. Uploads build context to S3
3. Builds & pushes Docker image via AWS CodeBuild (no local Docker needed)
4. Installs `kubectl` (if needed)
5. Configures `kubectl` for EKS cluster
6. Applies Kubernetes manifests (Deployment + Service)
7. Waits for rollout and prints endpoint URL

### Kubernetes Resources

**Deployment** (`k8s/deployment.yaml`):
- 2 replicas for high availability
- Container: Python 3.10-slim + Flask + XGBoost/LightGBM/CatBoost
- Resources: 500m–2000m CPU, 1–4 GiB memory
- Readiness probe: `/health` (30s init, 10s period, 3 failures)
- Liveness probe: `/health` (60s init, 30s period, 3 failures)

**Service** (`k8s/service.yaml`):
- Type: LoadBalancer (internal NLB via AWS annotation)
- Port: 80 → 8080

### Key Files

| File              | Purpose                                  |
|-------------------|------------------------------------------|
| `app.py`          | Flask application with /predict, /health |
| `Dockerfile`      | Python 3.10-slim + model + dependencies  |
| `deploy_to_eks.py`| Full automated deployment script         |
| `buildspec.yml`   | AWS CodeBuild spec for Docker image      |
| `requirements.txt`| Python dependencies                      |
| `k8s/deployment.yaml` | Kubernetes Deployment manifest      |
| `k8s/service.yaml`    | Kubernetes Service manifest (NLB)   |

---

## API Reference

Both SageMaker and EKS endpoints accept the same request/response format.

### Request

```json
POST /predict       (EKS)
POST /invocations   (SageMaker)

{
  "instances": [
    [0.9, 0.3, 0.4, 0.2, 0.85, 1.0, 0.9, 0.1, 0.0, 0.0,
     0.1, 1.0, 0.9, 0.1, 0.2, 0.9, 0.0, 0.0, 0.0, 0.05,
     0.0, 0.0]
  ]
}
```

Each instance is a 22-element array matching the feature order (f01–f22).

### Response

```json
{
  "predictions": [
    {
      "trust_score": 82.35,
      "tier": "GREEN",
      "base_scores": {
        "XGBoost": 81.2,
        "LightGBM": 82.5,
        "CatBoost": 83.1
      }
    }
  ]
}
```

### EKS Additional Endpoints

| Endpoint     | Method | Description            |
|--------------|--------|------------------------|
| `/health`    | GET    | Health check           |
| `/predict`   | POST   | Run inference          |
| `/`          | GET    | Service info & version |

### Integration from Downstream Services

```python
from sagemaker_deploy.invoke_from_other_model import WolfPackTrustScorer

scorer = WolfPackTrustScorer(endpoint_name="wolfpack-trust-score")

# Single prediction
result = scorer.predict(features=[0.9, 0.3, 0.4, ...])  # 22 features
print(result["trust_score"], result["tier"])

# Batch prediction
results = scorer.predict_batch(instances=[[...], [...], [...]])
```

---

## Project Structure

```
darwin/
├── README.md                                  ← Darwin Engine docs
├── WOLFPACK_README.md                         ← This file
│
│   ── Dataset ──
├── build_realworld_wolfpack_dataset.py        ← Dataset generation (100K rows)
├── wolfpack_realworld_dataset.csv             ← Generated dataset
├── wolfpack_training_dataset.csv              ← Training subset
│
│   ── Original Scoring Engine ──
├── wolf_pack_scoring_model.py                 ← Scoring formula + ROMA agent
│
│   ── Classification Models ──
├── train_wolfpack_ml_model.py                 ← GradientBoosting classifier
├── train_wolfpack_dl_model.py                 ← PyTorch FT-Transformer classifier
├── train_wolfpack_keras_model.py              ← Keras FT-Transformer classifier
├── train_wolfpack_rtdl_model.py               ← RTDL FT-Transformer classifier
│
│   ── Regression Models ──
├── train_wolfpack_regression_ensemble.py      ← Stacked Ensemble (XGB+LGBM+CB+Ridge)
├── train_wolfpack_regression_transformer.py   ← FT-Transformer regressor
├── train_wolfpack_regression_quantile.py      ← LightGBM quantile (P10/P50/P90)
│
│   ── Model Artifacts ──
├── wolfpack_ml_model.pkl                      ← GradientBoosting model
├── wolfpack_dl_model.pt                       ← PyTorch transformer model
├── wolfpack_keras_model.keras                 ← Keras transformer model
├── wolfpack_rtdl_model.pt                     ← RTDL transformer model
├── wolfpack_regression_ensemble_model.pkl     ← Stacked ensemble (DEPLOYED)
├── wolfpack_regression_transformer_model.pt   ← Transformer regressor
├── wolfpack_regression_quantile_model.pkl     ← Quantile regression
│
│   ── Metrics & Reports ──
├── wolfpack_ml_metrics.json
├── wolfpack_dl_metrics.json
├── wolfpack_keras_metrics.json
├── wolfpack_rtdl_metrics.json
├── wolfpack_regression_ensemble_metrics.json
├── wolfpack_regression_transformer_metrics.json
├── wolfpack_regression_quantile_metrics.json
├── wolfpack_*_report.txt                      ← Human-readable reports
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
│   ├── setup_aws.py                           ← AWS prerequisites setup
│   ├── package_model.py                       ← Build model.tar.gz
│   ├── deploy_to_sagemaker.py                 ← Deploy endpoint
│   ├── inference.py                           ← SageMaker inference handlers
│   ├── test_endpoint.py                       ← Endpoint tests
│   ├── invoke_from_other_model.py             ← Client library
│   ├── requirements.txt                       ← Container dependencies
│   ├── deploy_config.json                     ← AWS config (auto-generated)
│   └── model.tar.gz                           ← Packaged model (~309 MB)
│
│   ── EKS Deployment ──
├── eks_deploy/
│   ├── app.py                                 ← Flask inference server
│   ├── Dockerfile                             ← Container image definition
│   ├── deploy_to_eks.py                       ← Full deployment automation
│   ├── buildspec.yml                          ← CodeBuild spec
│   ├── requirements.txt                       ← Python dependencies
│   └── k8s/
│       ├── deployment.yaml                    ← K8s Deployment (2 replicas)
│       └── service.yaml                       ← K8s Service (internal NLB)
│
│   ── Word Documents ──
├── WOLFPACK_DATASETS_AND_FEATURES.docx
├── WOLFPACK_FILE_INVENTORY.docx
├── WOLFPACK_MODEL_GUIDE.docx
├── WOLFPACK_SCORING_SPEC.docx
├── ZYBERPOL_ARCHITECTURE.docx
├── ZYBERPOL_ARCHITECTURE.md
└── ZYBERPOL_ARCHITECTURE.pdf
```

---

## Getting Started

### Prerequisites

```bash
# Core ML
pip install numpy pandas scikit-learn xgboost lightgbm catboost

# Transformer models
pip install torch rtdl          # PyTorch + RTDL
pip install tensorflow          # Keras model

# AWS deployment
pip install boto3 'sagemaker>=2.200,<3'
```

### 1. Generate the Dataset

```bash
python build_realworld_wolfpack_dataset.py
# Output: wolfpack_realworld_dataset.csv (100K rows, 26 columns)
```

### 2. Train Classification Models

```bash
python train_wolfpack_ml_model.py       # GradientBoosting
python train_wolfpack_dl_model.py       # PyTorch Transformer
python train_wolfpack_keras_model.py    # Keras Transformer
python train_wolfpack_rtdl_model.py     # RTDL Transformer
```

### 3. Train Regression Models

```bash
python train_wolfpack_regression_ensemble.py      # Stacked Ensemble
python train_wolfpack_regression_transformer.py   # FT-Transformer
python train_wolfpack_regression_quantile.py      # Quantile Regression
```

### 4. Deploy to SageMaker

```bash
python sagemaker_deploy/setup_aws.py              # One-time setup
python sagemaker_deploy/package_model.py           # Build model.tar.gz
python sagemaker_deploy/deploy_to_sagemaker.py     # Deploy endpoint
python sagemaker_deploy/test_endpoint.py           # Verify
```

### 5. Deploy to EKS

```bash
python eks_deploy/deploy_to_eks.py                 # Full automated deploy
```

---

## Production Considerations

| Area                  | Recommendation                                                                 |
|-----------------------|--------------------------------------------------------------------------------|
| **Model Monitoring**  | Track prediction distribution (especially RED tier ratio) for data drift. Alert if RED rate exceeds 2x baseline. |
| **Threshold Tuning**  | Tier boundaries (25/50/75) should be recalibrated quarterly based on security outcomes. |
| **Feature Freshness** | Top features (`f20_failed_logins`, `f05_device_compliance`, `f16_ip_reputation`) are time-sensitive. Stale inputs degrade accuracy. |
| **Boundary Cases**    | For predictions within 3 points of a tier boundary, use quantile model's confidence interval to flag for human review. |
| **Scaling**           | EKS: horizontal pod autoscaling. SageMaker: auto-scaling policies on endpoint. |
| **Latency**           | EKS: ~40ms p50. SageMaker: ~50–90ms p50 (includes network overhead). |
| **Explainability**    | Use SHAP on the stacked ensemble for per-prediction explanations to the security team. |
| **Retraining**        | Retrain monthly on fresh data. Monitor for concept drift via R² on holdout. |

---

**Author:** Wolf Pack ML Engineering Team
**AWS Account:** 707075084132
**Region:** us-east-1
**SageMaker Endpoint:** `wolfpack-trust-score`
**EKS Cluster:** `zyberpol-east`
