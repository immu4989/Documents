# Hunter Behavioral Threat Model — ML Documentation

## Table of Contents

1. [Overview](#overview)
2. [Architecture Context](#architecture-context)
3. [Dataset Generation](#dataset-generation)
4. [Feature Engineering](#feature-engineering)
5. [Model Selection](#model-selection)
6. [Training Pipeline](#training-pipeline)
7. [Evaluation Framework & Accuracy Metrics](#evaluation-framework--accuracy-metrics)
8. [Error Analysis](#error-analysis)
9. [Wolf Pack → Hunter Integration](#wolf-pack--hunter-integration)
10. [Deployment — AWS SageMaker](#deployment--aws-sagemaker)
11. [Deployment — AWS EKS](#deployment--aws-eks)
12. [Endpoints, Clusters & Resource Paths](#endpoints-clusters--resource-paths)
13. [File Inventory](#file-inventory)
14. [How to Reproduce](#how-to-reproduce)
15. [Troubleshooting](#troubleshooting)

---

## Overview

The **Hunter Behavioral Threat Model** is a stacked ensemble regressor that predicts `threat_confidence` (0.0–1.0) for security events by fusing two signal sources:

- **Hunter Agent behavioral detection features** (15 features): EDR process anomalies, DNS query patterns, VPC flow signals, and binary detection flags
- **Wolf Pack Agent identity trust output** (6 features): trust score, tier, base learner scores, and model disagreement (uncertainty)

The model outputs a continuous threat score and a discrete risk level (`CRITICAL / HIGH / MEDIUM / LOW`), which downstream agents (Fire Factory, Fire Weaver, Alchemist) consume to generate strike orders, enforcement actions, and SOC alerts.

**Key Insight**: The identity-behavior interaction is the core signal. A suspicious action by an untrusted identity is far more dangerous than the same action by a trusted user. The model learns this non-linear relationship from combined Wolf Pack + Hunter features.

---

## Architecture Context

This model operates within the **Zyberpol** multi-agent security platform:

```
┌─────────────┐    NATS Mifal Bus     ┌──────────────┐    NATS Mifal Bus     ┌─────────────┐
│  Wolf Pack   │ ──────────────────>  │    Hunter     │ ──────────────────>  │ Fire Factory │
│  Agent       │  wolf_pack.trust.    │    Agent      │  hunter.threat.      │ Fire Weaver  │
│              │  score               │              │  assessed            │ Alchemist    │
│  Identity    │                      │  Behavioral   │                      │              │
│  Trust Score │                      │  Threat Model │                      │ Enforcement  │
└─────────────┘                      └──────────────┘                      └─────────────┘
  22 features                          21 features                           StrikeOrders
  → trust_score (0-100)               (15 Hunter + 6 WP)                    → block/isolate
  → tier (GREEN/YELLOW/ORANGE/RED)    → threat_confidence (0.0-1.0)         → quarantine
                                       → risk_level                          → alert SOC
```

**Agent-to-Agent Communication**: NATS Mifal Bus on port 4222
- Wolf Pack publishes to `wolf_pack.trust.score`
- Hunter subscribes, adjusts sensitivity, runs ML, publishes to `hunter.threat.assessed`

---

## Dataset Generation

**Script**: `build_hunter_behavioral_dataset.py`
**Output**: `hunter_behavioral_dataset.csv` (100,000 rows x 25 columns)

### Data Sources (Simulation Mode — Calibrated to Real Statistics)

| Signal Source | Real-World Datasets Used for Calibration |
|---|---|
| EDR Process Trees | CERT Insider Threat v6.2 + Splunk BOTS v3 |
| DNS Queries | CIC-IDS-2017 DNS tunneling + DGA datasets |
| VPC Flow Logs | UNSW-NB15 + LANL network events |
| Wolf Pack Output | Stacked Ensemble (XGBoost+LightGBM+CatBoost+Ridge) deployed on EKS/SageMaker |

### Class Balance

The dataset is class-balanced with 25% per risk level to ensure the model learns all four decision boundaries equally:

| Risk Level | Rows | Percentage |
|---|---|---|
| LOW | 25,000 | 25% |
| MEDIUM | 25,000 | 25% |
| HIGH | 25,000 | 25% |
| CRITICAL | 25,000 | 25% |

### Label Generation

The ground-truth `threat_confidence` is computed as a weighted combination of four signal components:

```
threat_conf = 0.30 * behavioral_strength
            + 0.25 * interaction_term      (behavioral × identity_distrust)
            + 0.20 * severity_score        (attack pattern indicators)
            + 0.15 * identity_distrust     (1 - trust_score)
            + 0.10 * uncertainty_signal    (base model disagreement)
            + N(0, 0.02)                   (noise for realism)
```

The `interaction_term` (behavioral_strength × identity_distrust) captures the key insight: suspicious behavior from untrusted users is far more dangerous.

### How to Generate

```bash
python build_hunter_behavioral_dataset.py --rows 100000 --out hunter_behavioral_dataset.csv
```

---

## Feature Engineering

### Feature Schema (21 Total: 15 Hunter + 6 Wolf Pack)

#### Hunter Detection Features (h01–h15)

| Feature | Name | Type | Range | Description |
|---|---|---|---|---|
| h01 | finding_count | Continuous | 0.0–1.0 | Number of detection findings (normalized by /20) |
| h02 | max_confidence | Continuous | 0.0–1.0 | Highest confidence score across findings |
| h03 | mean_confidence | Continuous | 0.0–1.0 | Average confidence across findings |
| h04 | vpc_flow_ratio | Continuous | 0.0–1.0 | Fraction of findings from VPC flow signals |
| h05 | edr_process_ratio | Continuous | 0.0–1.0 | Fraction of findings from EDR process signals |
| h06 | dns_query_ratio | Continuous | 0.0–1.0 | Fraction of findings from DNS query signals |
| h07 | off_hours_flag | Binary | 0/1 | Off-hours execution detected |
| h08 | beacon_flag | Binary | 0/1 | C2 beaconing pattern detected |
| h09 | lateral_movement_flag | Binary | 0/1 | Lateral movement detected |
| h10 | suspicious_dns_flag | Binary | 0/1 | Suspicious DNS (C2/DGA) detected |
| h11 | process_anomaly_flag | Binary | 0/1 | Process tree anomaly detected |
| h12 | data_exfil_flag | Binary | 0/1 | Data exfiltration pattern detected |
| h13 | lolbin_flag | Binary | 0/1 | Living-off-the-land binary detected |
| h14 | credential_access_flag | Binary | 0/1 | Credential access attempt detected |
| h15 | behavioral_boost | Continuous | 0.0–1.0 | Behavioral fusion anomaly boost score |

#### Wolf Pack Output Features (wp01–wp06)

| Feature | Name | Type | Range | Description |
|---|---|---|---|---|
| wp01 | trust_score | Continuous | 0.0–1.0 | Fused identity trust score (0–100 normalized) |
| wp02 | tier_numeric | Categorical | 0.0/0.33/0.66/1.0 | Traffic-light tier (GREEN=1.0, YELLOW=0.66, ORANGE=0.33, RED=0.0) |
| wp03 | xgb_score | Continuous | 0.0–1.0 | XGBoost base learner score (normalized) |
| wp04 | lgbm_score | Continuous | 0.0–1.0 | LightGBM base learner score (normalized) |
| wp05 | catboost_score | Continuous | 0.0–1.0 | CatBoost base learner score (normalized) |
| wp06 | base_score_variance | Continuous | 0.0–1.0 | Variance between base model scores (uncertainty signal) |

#### Output

| Field | Type | Description |
|---|---|---|
| threat_confidence | Continuous (0.0–1.0) | Predicted threat probability |
| risk_level | Categorical | Derived: CRITICAL (>=0.85), HIGH (>=0.65), MEDIUM (>=0.40), LOW (<0.40) |

---

## Model Selection

### Why Stacked Ensemble (XGBoost + LightGBM + CatBoost + Ridge)?

We evaluated multiple model architectures before selecting the stacked ensemble:

| Model | Pros | Cons | Verdict |
|---|---|---|---|
| **Stacked Ensemble (selected)** | Proven on Wolf Pack (R²=0.9954), handles mixed feature types (binary flags + continuous), complementary tree-building strategies reduce correlated errors, fast inference (<5ms) | Slightly more complex to deploy than a single model | **Selected** — best accuracy, same infra as Wolf Pack |
| Single XGBoost | Simple, strong baseline | Misses complementary error patterns from LightGBM/CatBoost | Good but not optimal |
| Deep Learning (MLP/Transformer) | Can learn complex non-linear interactions | Overkill for 21 tabular features, slower inference, harder to explain | Rejected — diminishing returns on tabular data |
| Random Forest | Simple, interpretable | Lower accuracy on boosting-friendly data, no early stopping | Rejected |
| Linear Regression | Fast, interpretable | Cannot capture non-linear feature interactions | Rejected |
| Single CatBoost | Handles categoricals natively | No ensemble diversity benefit | Good but not optimal |

### Architecture Details

```
Level-0 (Base Learners — 5-fold Out-of-Fold Predictions):
    ┌──────────────────────────────────────────────────┐
    │  XGBoost    → gradient boosted trees             │
    │  LightGBM   → histogram-based gradient boosting  │
    │  CatBoost   → ordered boosting, symmetric trees  │
    └──────────────────────────────────────────────────┘
                        │  OOF predictions
                        ▼
Level-1 (Meta-Learner):
    ┌──────────────────────────────────────────────────┐
    │  Ridge Regression → blends 3 base predictions    │
    │                     into final threat_confidence  │
    └──────────────────────────────────────────────────┘
```

### Hyperparameters

**XGBoost**:
- `n_estimators=500`, `max_depth=6`, `learning_rate=0.05`
- `subsample=0.8`, `colsample_bytree=0.8`, `reg_alpha=0.1`, `reg_lambda=1.0`
- `early_stopping_rounds=20`

**LightGBM**:
- `n_estimators=500`, `max_depth=6`, `learning_rate=0.05`
- `subsample=0.8`, `colsample_bytree=0.8`, `reg_alpha=0.1`, `reg_lambda=1.0`
- `early_stopping=20`

**CatBoost**:
- `iterations=500`, `depth=6`, `learning_rate=0.05`
- `l2_leaf_reg=3.0`, `early_stopping_rounds=20`

**Ridge Meta-Learner**: `alpha=1.0`

---

## Training Pipeline

**Script**: `train_hunter_behavioral_model.py`

### Data Splits

| Split | Percentage | Rows | Purpose |
|---|---|---|---|
| Train | 70% | 70,000 | Base learner training (within K-fold) |
| Validation | 10% | 10,000 | Early stopping for base learners |
| Test | 20% | 20,000 | Final evaluation (never seen during training) |

### Training Process

1. **StandardScaler** fitted on training data, applied to all splits
2. **5-fold OOF predictions** for each base learner on the train+val set (80K rows)
3. **Per-fold early stopping** using validation RMSE to prevent overfitting
4. **Ridge meta-learner** trained on OOF predictions from all 3 base models
5. **Final base models** retrained on full train+val set for the production bundle
6. **Model bundle** saved as pickle: `{base_models, meta_learner, scaler_mean, scaler_scale, feature_columns, ...}`

### How to Train

```bash
# Ensure libomp is available (macOS with PyTorch installed):
export DYLD_LIBRARY_PATH="/Users/imranahamed/Library/Python/3.9/lib/python/site-packages/torch/lib:$DYLD_LIBRARY_PATH"

python train_hunter_behavioral_model.py \
    --dataset hunter_behavioral_dataset.csv \
    --out-model hunter_behavioral_model.pkl \
    --out-metrics hunter_behavioral_metrics.json \
    --out-report hunter_behavioral_report.txt \
    --n-folds 5 \
    --seed 42
```

### Outputs

| File | Description |
|---|---|
| `hunter_behavioral_model.pkl` | Serialized model bundle (~4.5 MB) |
| `hunter_behavioral_metrics.json` | Machine-readable metrics and feature importance |
| `hunter_behavioral_report.txt` | Human-readable performance report |

---

## Evaluation Framework & Accuracy Metrics

### Test Set Results (20,000 samples, never seen during training)

| Metric | Value | Description |
|---|---|---|
| **RMSE** | 0.0470 | Root Mean Square Error — average prediction error magnitude |
| **MAE** | 0.0365 | Mean Absolute Error — average absolute deviation |
| **R²** | 0.9798 | Coefficient of Determination — 97.98% variance explained |
| **Risk Accuracy** | 99.31% | Fraction of samples where predicted risk level matches true risk level |

### Per-Base-Learner Test Metrics

| Base Learner | RMSE | MAE | R² | Risk Accuracy |
|---|---|---|---|---|
| XGBoost | 0.0470 | 0.0364 | 0.9798 | 99.22% |
| LightGBM | 0.0472 | 0.0365 | 0.9796 | 99.23% |
| CatBoost | 0.0477 | 0.0370 | 0.9791 | 99.35% |
| **Stacked Ensemble** | **0.0470** | **0.0365** | **0.9798** | **99.31%** |

### Meta-Learner Weights (Ridge)

| Base Learner | Weight | Interpretation |
|---|---|---|
| XGBoost | 0.4731 | Highest weight — most reliable base predictions |
| CatBoost | 0.2847 | Second — strong on categorical-like features (flags) |
| LightGBM | 0.2430 | Third — complementary histogram-based approach |
| Intercept | -0.0004 | Near-zero — base predictions are well calibrated |

### Top 10 Feature Importances (XGBoost)

| Rank | Feature | Importance | Interpretation |
|---|---|---|---|
| 1 | h01_finding_count | 0.7564 | **Dominant signal** — number of detection findings |
| 2 | h02_max_confidence | 0.1267 | Highest detection confidence drives risk level |
| 3 | h15_behavioral_boost | 0.0321 | Behavioral fusion anomaly score |
| 4 | wp02_tier_numeric | 0.0238 | **Wolf Pack tier** — most important identity feature |
| 5 | h03_mean_confidence | 0.0100 | Average detection confidence |
| 6 | h11_process_anomaly_flag | 0.0073 | Process tree anomaly indicator |
| 7 | wp01_trust_score | 0.0055 | Wolf Pack continuous trust score |
| 8 | h07_off_hours_flag | 0.0052 | Off-hours activity indicator |
| 9 | h14_credential_access_flag | 0.0050 | Credential theft attempt |
| 10 | h10_suspicious_dns_flag | 0.0048 | DNS anomaly (C2/DGA) |

### Risk Level Thresholds

| Risk Level | Confidence Range | Action |
|---|---|---|
| **CRITICAL** | >= 0.85 | Immediate response: isolate + alert SOC |
| **HIGH** | >= 0.65 | Elevated investigation: restrict + monitor |
| **MEDIUM** | >= 0.40 | Enhanced monitoring: watch closely |
| **LOW** | < 0.40 | Normal: standard monitoring |

---

## Error Analysis

### Key Observations

1. **Finding count dominates** (75.6% importance): The model relies heavily on the number of detection findings. This makes security sense — more findings = higher threat — but may need recalibration if detection systems evolve.

2. **Wolf Pack tier (2.4%) vs trust_score (0.6%)**: The discrete tier is more important than the continuous score, suggesting the model uses the traffic-light boundary crossings (GREEN→YELLOW, etc.) as strong signals.

3. **Stacking provides marginal but consistent improvement**: The ensemble improves risk accuracy from 99.22-99.35% (individual) to 99.31% (combined). The real value is **robustness** — no single base learner fails silently.

4. **Edge cases at risk boundaries**: The 0.69% error rate (138/20,000 samples) concentrates at decision boundaries (0.40, 0.65, 0.85) where small confidence differences change the risk level. This is expected behavior for continuous-to-categorical mapping.

5. **Identity features are complementary, not dominant**: Wolf Pack features (wp01-wp06) contribute ~3% of total importance. They refine threat assessment rather than drive it. This is correct behavior — behavioral signals should dominate, with identity context providing amplification/dampening via the adaptive sensitivity layer (see Integration section).

### Where the Model Might Struggle

- **Novel attack patterns**: If new detection types are added to Hunter that don't map to the existing 15 features, the model won't utilize them until retrained.
- **Trust score drift**: If Wolf Pack's scoring distribution shifts over time, the wp01-wp06 features may need recalibration.
- **Low-finding-count threats**: Sophisticated attackers generating few but high-confidence findings may be underweighted relative to noisy attackers with many low-confidence findings.

---

## Wolf Pack → Hunter Integration

### How It Works (4 Integration Points)

#### 1. Wolf Pack Scores Feed into Hunter

Wolf Pack's 6 output values become Hunter ML model input features (positions 16-21 of the 21-feature vector):

```python
wolfpack_features = [
    trust_score / 100.0,                          # wp01: normalized 0-1
    TIER_NUMERIC[tier],                            # wp02: GREEN=1.0, YELLOW=0.66, ORANGE=0.33, RED=0.0
    base_scores["XGBoost"] / 100.0,               # wp03: XGBoost base score
    base_scores["LightGBM"] / 100.0,              # wp04: LightGBM base score
    base_scores["CatBoost"] / 100.0,              # wp05: CatBoost base score
    min(variance_of_base_scores / 100.0, 1.0),    # wp06: base model disagreement
]
combined_features = hunter_15_features + wolfpack_features  # → 21 total
```

#### 2. Score >= 70: Hunter INCREASES Detection Sensitivity

When Wolf Pack trust score is >= 70 (GREEN/high YELLOW):
- `anomaly_threshold` is lowered from 0.50 → **0.35**
- Confidence is **boosted**: `adjusted = raw × (0.50 / 0.35) = raw × 1.43x`
- Rationale: A trusted user showing even mild anomalies is highly unusual — the baseline is clean, so deviations matter

#### 3. Score <= 30: Hunter DECREASES Sensitivity

When Wolf Pack trust score is <= 30 (RED/low ORANGE):
- `anomaly_threshold` is raised from 0.50 → **0.70**
- Confidence is **dampened**: `adjusted = raw × (0.50 / 0.70) = raw × 0.71x`
- Rationale: The user is already flagged by Wolf Pack. Only fire on high-confidence, multi-signal detections to avoid alert fatigue

#### 4. Handoff via NATS Mifal Bus

```
wolf_pack.trust.score  ──[NATS]──>  Hunter._on_trust_score()
                                        │
                                        ├── 1. Adjust anomaly_threshold
                                        ├── 2. Extract Wolf Pack features for ML
                                        ├── 3. Run Hunter ML (21 features)
                                        ├── 4. Apply sensitivity scaling
                                        │
                                        └──>  hunter.threat.assessed  ──[NATS]──> downstream
```

**NATS subjects**:
- `wolf_pack.trust.score` — Wolf Pack publishes, Hunter subscribes
- `hunter.threat.assessed` — Hunter publishes, downstream agents subscribe
- `hunter.anomaly` — Hunter publishes sensitivity adjustment events
- NATS port: **4222**

### Sensitivity Adjustment Example

Same raw ML confidence (0.55) produces different outcomes depending on trust:

| User Type | Trust Score | Threshold | Scale Factor | Adjusted Confidence | Is Anomalous? |
|---|---|---|---|---|---|
| Trusted (GREEN) | 85 | 0.35 | 1.43x | 0.7857 | **YES** (alert!) |
| Default (YELLOW) | 55 | 0.50 | 1.00x | 0.5500 | YES |
| Untrusted (RED) | 18 | 0.70 | 0.71x | 0.3929 | **NO** (suppressed) |

### Integration Client Classes

Located in `hunter_sagemaker_deploy/invoke_from_hunter.py`:

| Class | Purpose |
|---|---|
| `SensitivityConfig` | Derives anomaly_threshold from Wolf Pack trust score |
| `WolfPackTrustScorer` | SageMaker client for Wolf Pack endpoint (22 features → trust score) |
| `HunterBehavioralScorer` | SageMaker client for Hunter endpoint with sensitivity adjustment |
| `WolfPackHunterPipeline` | NATS-based production pipeline (subscribe → adjust → predict → publish) |

---

## Deployment — AWS SageMaker

### Endpoint Details

| Property | Value |
|---|---|
| **Endpoint Name** | `hunter-behavioral-threat` |
| **Instance Type** | `ml.m5.xlarge` |
| **Framework** | SKLearn 1.4-2, Python 3 |
| **Region** | `us-east-1` |
| **Status** | InService |

### SageMaker Files

| File | Purpose |
|---|---|
| `hunter_sagemaker_deploy/inference.py` | SageMaker 4-hook handler (model_fn, input_fn, predict_fn, output_fn) |
| `hunter_sagemaker_deploy/deploy_to_sagemaker.py` | Deployment script (preflight, cleanup stale, upload, deploy, smoke test) |
| `hunter_sagemaker_deploy/package_model.py` | Creates `model.tar.gz` (model.pkl + code/ + wheels) |
| `hunter_sagemaker_deploy/test_endpoint.py` | Tests 4 scenarios (benign, suspicious, moderate, high threat) |
| `hunter_sagemaker_deploy/setup_aws.py` | IAM role + S3 bucket setup |
| `hunter_sagemaker_deploy/invoke_from_hunter.py` | Integration client with sensitivity adjustment + NATS pipeline |
| `hunter_sagemaker_deploy/deploy_config.json` | Deployment configuration |

### SageMaker Resources

| Resource | Value |
|---|---|
| **IAM Role** | `arn:aws:iam::707075084132:role/WolfPackSageMakerRole` |
| **S3 Bucket** | `s3://wolfpack-sagemaker-707075084132-us-east-1` |
| **Model Artifact** | `s3://wolfpack-sagemaker-707075084132-us-east-1/hunter-behavioral-threat/model.tar.gz` |

### How to Deploy to SageMaker

```bash
# 1. Package model
cd hunter_sagemaker_deploy
python package_model.py

# 2. Setup AWS (first time only)
python setup_aws.py

# 3. Deploy
python deploy_to_sagemaker.py

# 4. Test
python test_endpoint.py --endpoint-name hunter-behavioral-threat
```

### SageMaker Request/Response Format

**Request** (POST, `application/json`):
```json
{
  "instances": [
    [0.25, 0.75, 0.60, 0.33, 0.45, 0.22, 1.0, 1.0, 0.0, 1.0, 0.0, 0.0,
     1.0, 0.0, 0.55, 0.43, 0.33, 0.42, 0.44, 0.43, 0.02]
  ]
}
```

**Response**:
```json
{
  "predictions": [
    {
      "threat_confidence": 0.7823,
      "risk_level": "HIGH",
      "base_scores": {"XGBoost": 0.78, "LightGBM": 0.80, "CatBoost": 0.77}
    }
  ]
}
```

---

## Deployment — AWS EKS

### Endpoint Details

| Property | Value |
|---|---|
| **EKS Cluster** | `zyberpol-east` |
| **Namespace** | `default` |
| **Deployment** | `hunter-behavioral-threat` |
| **Replicas** | 1 |
| **Service Type** | LoadBalancer (internal NLB) |
| **LoadBalancer Hostname** | `k8s-default-hunterbe-eeedce4623-94e918cc7b413e88.elb.us-east-1.amazonaws.com` |
| **Service Port** | 80 (target: 8080) |
| **Status** | Running (1/1 Ready) |

### EKS Files

| File | Purpose |
|---|---|
| `hunter_eks_deploy/app.py` | Flask inference server (/health, /predict, /) |
| `hunter_eks_deploy/Dockerfile` | Python 3.10-slim, gunicorn, non-root user |
| `hunter_eks_deploy/requirements.txt` | Python dependencies |
| `hunter_eks_deploy/buildspec.yml` | CodeBuild specification |
| `hunter_eks_deploy/deploy_to_eks.py` | 7-step EKS deployment script |
| `hunter_eks_deploy/k8s/deployment.yaml` | Kubernetes Deployment manifest |
| `hunter_eks_deploy/k8s/service.yaml` | Kubernetes Service manifest (internal NLB) |

### EKS Resources

| Resource | Value |
|---|---|
| **AWS Account** | `707075084132` |
| **Region** | `us-east-1` |
| **EKS Cluster** | `zyberpol-east` (6 nodes) |
| **ECR Repository** | `707075084132.dkr.ecr.us-east-1.amazonaws.com/zyberpol/hunter-behavioral-threat` |
| **Image Tag** | `latest` |
| **CodeBuild Project** | `hunter-behavioral-threat-build` |
| **S3 Build Source** | `s3://wolfpack-sagemaker-707075084132-us-east-1/hunter-codebuild-source/source.zip` |

### Kubernetes Security Configuration

The deployment complies with Kyverno cluster policies:

- `runAsNonRoot: true` (UID 1000, `appuser`)
- `readOnlyRootFilesystem: true` (with `/tmp` emptyDir for writable temp)
- `allowPrivilegeEscalation: false`
- `capabilities: drop ALL`
- Labels: `app.kubernetes.io/name`, `app.kubernetes.io/part-of: zyberpol`

### Resource Limits

| Resource | Request | Limit |
|---|---|---|
| CPU | 250m | 1000m |
| Memory | 512Mi | 2Gi |

### How to Deploy to EKS

```bash
# Full automated deployment:
cd darwin
python hunter_eks_deploy/deploy_to_eks.py --cluster zyberpol-east --namespace default

# Or manual steps:
# 1. Build Docker image (via CodeBuild)
aws codebuild start-build --project-name hunter-behavioral-threat-build --region us-east-1

# 2. Configure kubectl
aws eks update-kubeconfig --name zyberpol-east --region us-east-1

# 3. Apply manifests
kubectl apply -f hunter_eks_deploy/k8s/deployment.yaml
kubectl apply -f hunter_eks_deploy/k8s/service.yaml

# 4. Wait for rollout
kubectl rollout status deployment/hunter-behavioral-threat

# 5. Test (via port-forward since NLB is internal)
kubectl port-forward svc/hunter-behavioral-threat 8081:80
curl http://localhost:8081/health
curl -X POST http://localhost:8081/predict \
  -H "Content-Type: application/json" \
  -d '{"instances": [[0.50, 0.88, 0.75, 0.30, 0.50, 0.20, 1.0, 1.0, 1.0, 1.0, 0.0, 1.0, 1.0, 1.0, 0.85, 0.15, 0.0, 0.12, 0.18, 0.14, 0.08]]}'
```

### EKS Request/Response Format

Same as SageMaker — both serve the identical model with the same API:

- `GET /health` → `{"status": "healthy"}`
- `POST /predict` → `{"predictions": [...], "latency_ms": N}`
- `GET /` → service metadata

---

## Endpoints, Clusters & Resource Paths

### Live Endpoints

| Endpoint | Platform | URL/Name | Access |
|---|---|---|---|
| Hunter ML (SageMaker) | SageMaker | `hunter-behavioral-threat` | Via `boto3 sagemaker-runtime invoke_endpoint` |
| Hunter ML (EKS) | EKS | `k8s-default-hunterbe-eeedce4623-94e918cc7b413e88.elb.us-east-1.amazonaws.com` | Internal NLB (VPC only) |
| Wolf Pack (SageMaker) | SageMaker | `wolfpack-trust-score` | Via `boto3 sagemaker-runtime invoke_endpoint` |
| Wolf Pack (EKS) | EKS | Via `wolfpack-trust-score` K8s service | Internal NLB (VPC only) |

### AWS Resources

| Resource | ARN / ID |
|---|---|
| AWS Account | `707075084132` |
| Region | `us-east-1` |
| EKS Cluster | `zyberpol-east` |
| IAM Role (SageMaker) | `arn:aws:iam::707075084132:role/WolfPackSageMakerRole` |
| S3 Bucket | `wolfpack-sagemaker-707075084132-us-east-1` |
| ECR Repo (Hunter) | `707075084132.dkr.ecr.us-east-1.amazonaws.com/zyberpol/hunter-behavioral-threat` |
| ECR Repo (Wolf Pack) | `707075084132.dkr.ecr.us-east-1.amazonaws.com/zyberpol/wolfpack-trust-score` |
| CodeBuild (Hunter) | `hunter-behavioral-threat-build` |

### NATS Mifal Bus

| Subject | Publisher | Subscriber | Port |
|---|---|---|---|
| `wolf_pack.trust.score` | Wolf Pack Agent | Hunter Agent | 4222 |
| `hunter.threat.assessed` | Hunter Agent | Fire Factory, Fire Weaver, Alchemist | 4222 |
| `hunter.anomaly` | Hunter Agent | Monitoring/Logging | 4222 |

---

## File Inventory

### Model Training & Data

| File | Description |
|---|---|
| `build_hunter_behavioral_dataset.py` | Dataset generator (100K rows, 21 features) |
| `train_hunter_behavioral_model.py` | Stacked ensemble training script |
| `hunter_behavioral_dataset.csv` | Training dataset (100K rows x 25 columns) |
| `hunter_behavioral_model.pkl` | Trained model bundle (~4.5 MB) |
| `hunter_behavioral_metrics.json` | Machine-readable evaluation metrics |
| `hunter_behavioral_report.txt` | Human-readable performance report |

### EKS Deployment

| File | Description |
|---|---|
| `hunter_eks_deploy/app.py` | Flask inference server |
| `hunter_eks_deploy/Dockerfile` | Docker image definition (non-root, slim) |
| `hunter_eks_deploy/requirements.txt` | Python dependencies |
| `hunter_eks_deploy/buildspec.yml` | AWS CodeBuild specification |
| `hunter_eks_deploy/deploy_to_eks.py` | Automated 7-step EKS deployment |
| `hunter_eks_deploy/k8s/deployment.yaml` | K8s Deployment manifest |
| `hunter_eks_deploy/k8s/service.yaml` | K8s Service manifest (internal NLB) |

### SageMaker Deployment

| File | Description |
|---|---|
| `hunter_sagemaker_deploy/inference.py` | SageMaker 4-hook inference handler |
| `hunter_sagemaker_deploy/deploy_to_sagemaker.py` | SageMaker deployment script |
| `hunter_sagemaker_deploy/package_model.py` | Creates model.tar.gz for SageMaker |
| `hunter_sagemaker_deploy/test_endpoint.py` | Endpoint test with 4 threat scenarios |
| `hunter_sagemaker_deploy/setup_aws.py` | AWS prerequisites setup (IAM + S3) |
| `hunter_sagemaker_deploy/invoke_from_hunter.py` | Integration client (sensitivity + NATS) |
| `hunter_sagemaker_deploy/deploy_config.json` | Deployment configuration |

---

## How to Reproduce

### Full Pipeline (End to End)

```bash
cd /Users/imranahamed/Documents/vezran_development/repo/darwin

# macOS: Ensure libomp is available for XGBoost
export DYLD_LIBRARY_PATH="/Users/imranahamed/Library/Python/3.9/lib/python/site-packages/torch/lib:$DYLD_LIBRARY_PATH"

# Step 1: Generate dataset
python build_hunter_behavioral_dataset.py --rows 100000

# Step 2: Train model
python train_hunter_behavioral_model.py

# Step 3: Deploy to SageMaker
cd hunter_sagemaker_deploy
python package_model.py
python deploy_to_sagemaker.py
python test_endpoint.py

# Step 4: Deploy to EKS
cd ../hunter_eks_deploy
python deploy_to_eks.py

# Step 5: Test integration
cd ../hunter_sagemaker_deploy
python invoke_from_hunter.py  # Runs 4 demo scenarios
```

### Python Dependencies

```
numpy
pandas
scikit-learn
xgboost
lightgbm
catboost
boto3
sagemaker
flask
gunicorn
nats-py       # For NATS Mifal Bus integration
```

---

## Troubleshooting

### Common Issues

| Issue | Cause | Fix |
|---|---|---|
| `XGBoostError: libxgboost.dylib could not be loaded` | libomp not found on macOS | `export DYLD_LIBRARY_PATH=".../torch/lib:$DYLD_LIBRARY_PATH"` |
| `429 Too Many Requests` (Docker build) | Docker Hub rate limit | Use `public.ecr.aws/docker/library/python:3.10-slim` as base image |
| `FileNotFoundError: hunter_behavioral_model.pkl` | Wrong working directory | Run from `darwin/` parent directory, not from `hunter_eks_deploy/` |
| `AccessDeniedException: ecr-public:GetAuthorizationToken` | CodeBuild role missing permission | Add `ecr-public:GetAuthorizationToken` and `sts:GetServiceBearerToken` to CodeBuild IAM role |
| `0/6 nodes are available: Insufficient cpu` | EKS cluster CPU-constrained | Reduce CPU requests (250m) and replicas (1) |
| Kyverno `PolicyViolation: require-run-as-nonroot` | Pod running as root | Add `securityContext.runAsNonRoot: true` and create non-root user in Dockerfile |
| Kyverno `PolicyViolation: require-readonly-rootfs` | Writable root filesystem | Add `securityContext.readOnlyRootFilesystem: true` + emptyDir at `/tmp` |
| Kyverno `PolicyViolation: require-labels` | Missing K8s labels | Add `app.kubernetes.io/name` and `app.kubernetes.io/part-of` labels |
| `kubectl: aws not found` | AWS CLI not on PATH for kubectl auth | Symlink aws to `~/.local/bin/aws` and add to PATH |

### Verifying Deployments

```bash
# SageMaker
aws sagemaker describe-endpoint --endpoint-name hunter-behavioral-threat --region us-east-1 --query 'EndpointStatus'

# EKS
kubectl get deployment hunter-behavioral-threat
kubectl get pods -l app=hunter-behavioral-threat
kubectl logs -l app=hunter-behavioral-threat

# Port-forward for testing (EKS NLB is internal)
kubectl port-forward svc/hunter-behavioral-threat 8081:80
curl http://localhost:8081/health
```
