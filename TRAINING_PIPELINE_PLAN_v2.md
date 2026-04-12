# Zyberpol Training Pipeline Plan v2

**Date:** April 9, 2026
**Supersedes:** Lambda Cloud pipeline (terminated April 8, 2026)
**Status:** Ready to execute — all infrastructure built, compute not yet provisioned

---

## 1. Executive Summary

Zyberpol trains 5 custom cybersecurity LLMs across 4 deployment tiers (20 model variants). The Lambda Cloud 8x H100 cluster was terminated on April 8. This plan replaces self-hosted GPU training with **AWS SageMaker** (fine-tuning) + **GCP Vertex AI** (multi-model tournament) + **AWS Bedrock** (production inference).

**Total cost to train all 5 models:** $11,537 one-time
**Monthly operations:** $1,500
**Time to first model (CyberLlama):** ~48 hours from provisioning

---

## 2. What's Already Built

| System | Status | Purpose |
|--------|--------|---------|
| **FORGE Data Engine** | Deployed | 12 parsers, 28 categories, 222 GCS dirs classified, 8.7TB curated data |
| **VIVISECT** | Production-ready | 100 files, 939 tests — model teardown + layer transplant platform |
| **CITADEL** | Complete | 6-layer ops monitoring (Watchtower → War Room) |
| **BASTION** | Complete | 5-phase release gates with CHRONICLE hash chain audit |
| **CRUCIBLE** | Complete | 6-layer testing furnace + TRIBUNAL dashboard |
| **FEEDBACK** | Complete | RLHF pipeline, reward models, HITL review queue, 24 API endpoints |
| **Training Data** | 8.7TB indexed | 400+ datasets, 66 GCS dirs, 11M records, L1-L5 quality levels |
| **VIVISECT-BENCH** | 800 items | 8 categories × 4 difficulty levels, machine-scorable |
| **Distillation Prompts** | 938 prompts | Alert triage (300), detection (200), hunting (150), intel (100), IR (100), risk (88) |

---

## 3. The 5 Models

### 3.1 Model Specifications

| Model | Purpose | Base (Tier 1) | Active Params | Training Data |
|-------|---------|---------------|---------------|---------------|
| **CyberLlama** | General cybersecurity assistant | Qwen 3.5-35B-A3B | 3B (MoE) | 1.7M records |
| **SigmaForge** | Detection rule generation | Qwen 3.5-122B-A10B | 10B (MoE) | 500K records |
| **AttackDNA** | MITRE ATT&CK classification | Nemotron 3 Super 120B-A12B | 12B (MoE) | 400K records |
| **HuntMind** | Threat hunting query generation | Mistral Large 3 675B | 675B (dense) | 300K records |
| **ThreatOracle** | Multilingual threat intelligence | Qwen 3.5 397B-A17B | 17B (MoE) | 500K records |

### 3.2 Training Data Per Model

| Model | Primary Sources | Format | Composition |
|-------|----------------|--------|-------------|
| **CyberLlama** | Fenrir v2, CyberLlama dataset, ShareGPT cybersec, Heimdall, 32K instruct, CVE | ChatML/instruction | 40% distilled CoT + 30% curated + 20% domain + 10% adversarial |
| **SigmaForge** | Sigma rules, YARA, Snort/Suricata, Elastic detection, Splunk queries, Big-Vul | Instruction (log → rule) | 40% distilled CoT + 30% curated + 20% domain + 10% adversarial |
| **AttackDNA** | MITRE ATT&CK STIX, Trendyol, Fenrir, pentest, red team tools | ChatML/instruction | 40% distilled CoT + 30% curated + 20% domain + 10% adversarial |
| **HuntMind** | Threat hunter playbook, OTRF datasets, Splunk BOTS, hunt hypotheses | Instruction (query gen) | 40% distilled CoT + 30% curated + 20% domain + 10% adversarial |
| **ThreatOracle** | Reddit 33.9M cybersec, APT notes, CTI datasets, MITRE groups | Multilingual instruction | 40% distilled CoT + 30% curated + 20% domain + 10% adversarial |

---

## 4. Four-Tier Deployment Architecture

### Tier 1: GLOBAL (Commercial, Cost-Optimized)

| Model | Base | Params | Active | Provider | Cost/1M tokens |
|-------|------|--------|--------|----------|---------------|
| AttackDNA | Nemotron 3 Super 120B-A12B | 120B | 12B | NVIDIA | $0.10/$0.50 |
| CyberLlama | Qwen 3.5-35B-A3B | 35B | 3B | Alibaba | $0.16/$1.30 |
| SigmaForge | Qwen 3.5-122B-A10B | 122B | 10B | Alibaba | $0.26/$2.08 |
| HuntMind | Mistral Large 3 675B | 675B | 675B | Mistral (FR) | $0.50/$1.50 |
| ThreatOracle | Qwen 3.5 397B-A17B | 397B | 17B | Alibaba | $0.39/$2.34 |

### Tier 2: GLOBAL PRO (Enterprise Maximum, Chinese Models Allowed)

| Model | Base | Params | Provider |
|-------|------|--------|----------|
| AttackDNA | DeepSeek V3.2 | 685B | DeepSeek (CN) |
| CyberLlama | Qwen 3.5 397B-A17B | 397B | Alibaba (CN) |
| SigmaForge | MiniMax M2.5 | ~456B | MiniMax (CN) |
| HuntMind | GLM-5 | ~675B | Zhipu (CN) |
| ThreatOracle | Kimi K2.5 | 1T+ | Moonshot (CN) |

### Tier 3: SOVEREIGN (US/Five Eyes/NATO, Zero Chinese)

| Model | Base | Params | Provider |
|-------|------|--------|----------|
| AttackDNA | Nemotron 3 Super 120B-A12B | 120B | NVIDIA (US) |
| CyberLlama | Mistral Large 3 675B | 675B | Mistral (FR/EU) |
| SigmaForge | Devstral 2 2512 | ~120B | Mistral (FR/EU) |
| HuntMind | gpt-oss-120b | 120B | OpenAI (US) |
| ThreatOracle | Nemotron Ultra 253B | 253B | NVIDIA (US) |

### Tier 4: SOVEREIGN PRO (Maximum Power, NATO-Only)
Same as Tier 3 — no upgrade path without Chinese models.

---

## 5. Twelve-Stage Training Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│                    12-STAGE TRAINING PIPELINE                    │
│                                                                  │
│  Stage 1:  TEARDOWN ────────── Dissect competitor models         │
│            Tool: VIVISECT     (VIVISECT)                         │
│            Status: CODE READY, NOT EXECUTED                      │
│                                                                  │
│  Stage 2:  DATA ASSEMBLY ───── 5+ TB curated corpus             │
│            Tool: FORGE        from GCS                           │
│            Status: ✅ COMPLETE (8.7TB, 222 dirs)                 │
│                                                                  │
│  Stage 3:  DISTILLATION ────── 70B+ teacher generates           │
│            Tool: Bedrock       chain-of-thought data             │
│            Status: 938 PROMPTS READY, NOT EXECUTED               │
│                                                                  │
│  Stage 4:  QUALITY FILTER ──── Rejection sampling               │
│            Tool: FORGE QA      keep top 70-80%                   │
│            Status: CODE READY, NOT EXECUTED                      │
│                                                                  │
│  Stage 5:  DATA MIXING ─────── Blend distilled + curated        │
│            Tool: FORGE          + synthetic (40/30/20/10)        │
│            Status: METHODOLOGY DEFINED, NOT EXECUTED             │
│                                                                  │
│  Stage 6:  BASE SELECTION ──── Pick SOTA base per tier          │
│            Plan: 4-Tier        × 5 models                       │
│            Status: ✅ COMPLETE (all bases selected)              │
│                                                                  │
│  Stage 7:  TRAINING ─────────── QLoRA fine-tuning               │
│            Tool: Unsloth        (single GPU) or Axolotl          │
│            Compute: SageMaker   (multi-GPU)                      │
│            Status: NOT EXECUTED — NO GPU PROVISIONED             │
│                                                                  │
│  Stage 8:  WEIGHT SURGERY ──── Transplant layers from           │
│            Tool: VIVISECT      teardown analysis                 │
│            Status: CODE READY, DEPENDS ON STAGE 1               │
│                                                                  │
│  Stage 9:  DARE-TIES MERGE ─── Combine capabilities             │
│            Tool: mergekit       from multiple fine-tunes         │
│            Status: METHODOLOGY DEFINED, NOT EXECUTED             │
│                                                                  │
│  Stage 10: EVALUATION ──────── CS-Eval + head-to-head           │
│            Tool: VIVISECT-BENCH + 6Shatan6 adversarial          │
│            Status: 800 BENCH ITEMS READY, NOT EXECUTED           │
│                                                                  │
│  Stage 11: QUANTIZATION ────── GGUF + ONNX + FP8               │
│            Tool: llama.cpp      Q4_K_M, Q6_K, Q8_0             │
│            Status: NOT EXECUTED                                  │
│                                                                  │
│  Stage 12: DEPLOYMENT ──────── 3-tier serving                   │
│            Agent sidecars (GGUF) + shared (vLLM) + API          │
│            Status: ARCHITECTURE DEFINED, NOT DEPLOYED            │
└─────────────────────────────────────────────────────────────────┘
```

**Summary:** Stages 2 and 6 complete. Stage 1 code ready. Stages 3-12 not executed.

---

## 6. Compute Strategy (Post-Lambda)

### 6.1 Training Compute

| Model | Platform | Instance | GPU | Est. Time | Est. Cost |
|-------|----------|----------|-----|-----------|-----------|
| **CyberLlama** (Priority 1) | AWS SageMaker | `ml.p4d.24xlarge` | 8×A100 (320GB) | 40h | $1,300 |
| **SigmaForge** (Priority 2) | AWS SageMaker | `ml.p4d.24xlarge` | 8×A100 (320GB) | 20h | $650 |
| **AttackDNA** | AWS SageMaker | `ml.g5.12xlarge` | 4×A10G (96GB) | 10h | $120 |
| **HuntMind** | GCP Vertex AI | `a2-ultragpu-8g` | 8×A100 (640GB) | 30h | $1,000 |
| **ThreatOracle** | GCP Vertex AI | `a2-ultragpu-8g` | 8×A100 (640GB) | 25h | $830 |
| Quantization (all) | AWS SageMaker | `ml.g5.xlarge` | 1×A10G (24GB) | 20h | $100 |
| **Total Training** | | | | **~145h** | **$4,000** |

### 6.2 Inference Compute (Production)

| Tier | Platform | Instance | Models | Monthly Cost |
|------|----------|----------|--------|-------------|
| Sidecar (GGUF) | EKS (CPU) | Agent pods | AttackDNA Q4_K_M | $0 (runs on agent pods) |
| Shared (vLLM) | SageMaker | `ml.inf2` (Inferentia2) | CyberLlama, SigmaForge | $300/mo |
| API (Bedrock) | AWS Bedrock | Serverless | Claude Haiku, Nova Micro | $200/mo |
| Fallback (GPU) | SageMaker | `ml.g5` | HuntMind, ThreatOracle | Variable |
| **Total Monthly** | | | | **~$1,500/mo** |

### 6.3 Current Bedrock Models in Production

| Model | Bedrock ID | Use | Cost/1K tokens |
|-------|-----------|-----|---------------|
| Claude Haiku 4.5 | `us.anthropic.claude-haiku-4-5-20251001-v1:0` | Primary classifier (FORGE) | $0.25 |
| Claude Sonnet 4 | `anthropic.claude-sonnet-4-20250514-v1:0` | Labeling (FORGE) | $3.00 |
| Gemini 3 Flash Lite | `gemini-3-flash-lite` (Vertex) | Bulk labeling fallback | $0.05 |
| DeepSeek V3.2 | `deepseek.v3.2` | Code analysis, multilingual | $0.14 |
| Qwen 3.5 32B | `qwen.qwen3-32b-v1:0` | Code analysis, reasoning | $0.16 |
| Kimi K2.5 | `moonshotai.kimi-k2.5` | Long context (131K) | $0.12 |
| Titan Embed V2 | `amazon.titan-embed-text-v2:0` | Embeddings (1024-d) | $0.02 |

---

## 7. Training Hyperparameters

| Parameter | SFT | DPO | GRPO |
|-----------|-----|-----|------|
| LoRA rank | 64 | 64 | 64 |
| LoRA alpha | 128 | 128 | 128 |
| Learning rate | 1.5-2.0e-5 | 3-5e-6 | 0.5-1.0e-6 |
| Batch size | 2-4 (effective 256 with accumulation) | 2-4 | 2-4 |
| Epochs | 1-2 (large), 3-5 (small) | 1-2 | 1-2 |
| Max seq length | 4096 (8192 for cybersec) | 4096 | 4096 |
| Warmup ratio | 0.03 | 0.03 | 0.03 |
| Scheduler | Cosine | Cosine | Cosine |
| Precision | BF16 | BF16 | BF16 |
| Gradient checkpointing | true | true | true |
| Flash Attention | true | true | true |
| DPO beta | — | 0.1-0.15 | — |
| GRPO num_generations | — | — | 8 |
| KL coefficient | — | — | 0.05 |

### Training Tools

| Tool | Use Case | When |
|------|----------|------|
| **Unsloth** | Single-GPU QLoRA (≤120B) | AttackDNA, CyberLlama, SigmaForge |
| **Axolotl** | Multi-GPU training (253B+) | HuntMind (675B), ThreatOracle (397B) |
| **DeepSpeed ZeRO Stage 3** | Memory optimization | With Axolotl for 675B+ |
| **mergekit** | DARE-TIES, SLERP merge | Stage 9 post-training |
| **TransformerLens** | Mechanistic interpretability | Stage 1 teardown |
| **baukit** | Activation patching | Stage 8 weight surgery |

---

## 8. Evaluation Framework

### 8.1 Benchmarks

| Benchmark | Items | Purpose | Target |
|-----------|-------|---------|--------|
| **CS-Eval** | Standard | Same benchmark Trendyol used (rank #3 on leaderboard) | >91.03 (beat Trendyol) |
| **VIVISECT-BENCH** | 800 | 8 categories × 4 difficulty levels, machine-scorable | Custom rubric per item |
| **Head-to-Head** | 100 | LLM-as-judge blind comparison vs competitors | Win rate >60% |
| **6Shatan6 Arena** | 47+ models | Adversarial tournament — detect attacks others miss | >70% detection rate |
| **Task-Specific** | Per model | TP/FP accuracy, Sigma validity, hunt hypothesis quality | >95% TP/FP, >90% Sigma compile |

### 8.2 VIVISECT-BENCH Categories

| Category | L1 (Foundational) | L2 (Applied) | L3 (Complex Scenario) | L4 (Expert) |
|----------|-------------------|--------------|----------------------|-------------|
| Malware Analysis | 25 items | 25 items | 25 items | 25 items |
| Incident Response | 25 | 25 | 25 | 25 |
| Detection Engineering | 25 | 25 | 25 | 25 |
| Threat Intelligence | 25 | 25 | 25 | 25 |
| Vulnerability Research | 25 | 25 | 25 | 25 |
| Red Team Operations | 25 | 25 | 25 | 25 |
| Forensics | 25 | 25 | 25 | 25 |
| Cloud Security | 25 | 25 | 25 | 25 |

---

## 9. Deployment Architecture (3-Tier)

```
┌─────────────────────────────────────────────────────────────┐
│                    3-TIER MODEL SERVING                       │
│                                                               │
│  TIER 1: AGENT SIDECARS (GGUF via llama.cpp)                 │
│  ├── AttackDNA Q4_K_M on every agent pod                     │
│  ├── <100ms inference, zero API cost                         │
│  ├── CPU-based, no GPU needed                                │
│  └── 16 agent pods × 1 model = 16 instances                 │
│                                                               │
│  TIER 2: SHARED SERVICES (vLLM on SageMaker/EKS)            │
│  ├── CyberLlama — general cybersec queries                   │
│  ├── SigmaForge — detection rule generation                  │
│  ├── HuntMind — threat hunting queries                       │
│  ├── <2s inference, shared across agents                     │
│  └── SageMaker Inferentia2 or EKS GPU nodes                 │
│                                                               │
│  TIER 3: API (Bedrock / hosted)                              │
│  ├── ThreatOracle — multilingual threat intel                │
│  ├── Claude Opus 4.6 — complex reasoning                    │
│  ├── <10s for complex analysis                               │
│  └── Pay-per-token, no infrastructure                        │
└─────────────────────────────────────────────────────────────┘
```

### Quantization Formats

| Format | Use | Size (70B) | Speed | Quality |
|--------|-----|-----------|-------|---------|
| Q4_K_M (GGUF) | Agent sidecars | ~40GB | Fast (CPU) | Good |
| Q6_K (GGUF) | High-quality local | ~55GB | Medium | Better |
| Q8_0 (GGUF) | Maximum quality local | ~70GB | Slower | Best |
| FP8 | GPU inference (vLLM) | ~70GB | Fastest | Near-lossless |
| ONNX | Cross-platform | ~70GB | Variable | Lossless |
| AWQ | Fast GPU inference | ~40GB | Very fast | Good |

---

## 10. Multi-Model Router (Production Inference)

### 10.1 Five-Model Hybrid Router

| Model | Role | Strength | Traffic % | Cost/1M |
|-------|------|----------|-----------|---------|
| Claude Opus 4.6 | Architect | Complex reasoning, self-correction | 3% | $15.00 |
| Gemini 3.1 Pro | Researcher | 2M+ token window, document ingestion | 7% | $1.25 |
| Nemotron 3 Super | Specialist | Agentic reasoning, tool use | 10% | $0.15 |
| Mistral Small 4 | Speedster | Real-time interaction, structured output | 20% | $0.10 |
| DeepSeek V3.2 | Worker | Best intelligence-per-dollar, bulk processing | 60% | $0.27 |

**Blended cost:** ~$0.80/1M tokens (95% reduction vs Claude-only)

### 10.2 Sovereign Router Alternative

Replace DeepSeek V3.2 (Chinese) with Llama 4 Scout (Meta/US) — only substitution needed for US/Five Eyes/NATO compliance. All other router models are Western-origin.

---

## 11. Execution Plan

### Phase 1: Foundation ($537, Week 1)

| Task | Service | Cost | Outcome |
|------|---------|------|---------|
| Titan V2 embeddings for RAG | Bedrock | $100 | Searchable knowledge base |
| Qdrant vector store deployment | EKS | $37 | Semantic search for NEXIS |
| Bedrock Guardrails setup | Bedrock | $0 | PII filtering, content safety |
| Nova Micro alert triage pilot | Bedrock | $200 | Automated alert classification |
| Claude Haiku NEXIS copilot | Bedrock | $200 | AI copilot with RAG |

### Phase 2: Custom Model Training ($4,000, Weeks 2-3)

**Priority 1 — CyberLlama:**
1. Provision SageMaker `ml.p4d.24xlarge`
2. Load Qwen 3.5-35B-A3B base from HuggingFace
3. Load 1.7M training records from GCS via FORGE manifest
4. Run QLoRA fine-tune with Unsloth (40h, $1,300)
5. Evaluate on VIVISECT-BENCH + CS-Eval
6. Quantize to GGUF Q4_K_M + FP8
7. Deploy to SageMaker Inferentia2 endpoint

**Priority 2 — SigmaForge:**
1. Same SageMaker instance (reuse)
2. Load Qwen 3.5-122B-A10B base
3. Load 500K detection rule training records
4. Run QLoRA fine-tune (20h, $650)
5. Evaluate: Sigma compilation rate, YARA validity, FP rate
6. Deploy to shared vLLM service

**Priority 3-5 — AttackDNA, HuntMind, ThreatOracle:**
- AttackDNA: SageMaker `ml.g5.12xlarge` (10h, $120)
- HuntMind: Vertex AI `a2-ultragpu-8g` (30h, $1,000)
- ThreatOracle: Vertex AI `a2-ultragpu-8g` (25h, $830)

### Phase 3: Adversarial Tournament ($3,000, Week 4)

| Task | Service | Cost |
|------|---------|------|
| Model Garden — host 47+ models for 6Shatan6 | Vertex AI | $1,500 |
| Agent Builder — autonomous evaluation agents | Vertex AI | $500 |
| AutoML — ensemble optimization | Vertex AI | $500 |
| Gemini 3.1 Pro — 2M context log analysis | Vertex AI | $500 |

### Phase 4: Analytics ($2,000, Weeks 4-5)

| Task | Service | Cost |
|------|---------|------|
| BigQuery ML — SQL-based anomaly detection | GCP | $800 |
| Document AI — automated report parsing | GCP | $500 |
| Vision AI — malware screenshot analysis | GCP | $400 |
| Cloud Run — serverless NEXIS RAG | GCP | $300 |

### Phase 5: MLOps ($2,000, Week 5)

| Task | Service | Cost |
|------|---------|------|
| SageMaker Pipelines — automated retraining | AWS | $500 |
| SageMaker Ground Truth — human labeling | AWS | $500 |
| Model Monitor — drift detection | AWS | $300 |
| Agent deployment (16 sidecars) | EKS | $500 |
| A/B testing infrastructure | FEEDBACK | $200 |

---

## 12. Timeline

```
Week 1:  Foundation ─────── Embeddings, RAG, guardrails, alert triage pilot
Week 2:  CyberLlama ─────── Fine-tune + evaluate + quantize + deploy
Week 3:  SigmaForge ─────── Fine-tune + evaluate + deploy
         AttackDNA ────────── Fine-tune (parallel, cheaper GPU)
Week 4:  HuntMind ────────── Fine-tune on Vertex AI
         ThreatOracle ────── Fine-tune on Vertex AI  
         6Shatan6 ─────────── Adversarial tournament
Week 5:  MLOps ──────────── Pipelines, monitoring, agent deployment
         Analytics ────────── BigQuery, Document AI
```

**First model in production:** End of Week 2 (CyberLlama)
**All 5 models deployed:** End of Week 4
**Full MLOps operational:** End of Week 5

---

## 13. Risk Mitigations

| Risk | Mitigation |
|------|-----------|
| SageMaker capacity unavailable | Fallback: GCP Vertex AI has same A100 instances |
| Training quality insufficient | Stage 4 quality filter rejects bottom 20-30%, retrain with cleaned data |
| Quantization degrades quality | Test Q6_K and Q8_0 before settling on Q4_K_M for sidecars |
| Cost overrun | SageMaker Spot instances (70% discount), shut down immediately after training |
| Model drift in production | SageMaker Model Monitor + FEEDBACK review queue + VIVISECT continuous analysis |
| Bedrock API changes | Inference profiles + multi-provider router (Bedrock + Vertex fallback) |

---

## 14. Success Criteria

| Metric | Target | How Measured |
|--------|--------|-------------|
| CS-Eval score | >91.03 (beat Trendyol #3) | lm-evaluation-harness |
| VIVISECT-BENCH pass rate | >80% across all 8 categories | Machine-scored rubric |
| Head-to-head win rate | >60% vs base models | LLM-as-judge (100 prompts) |
| 6Shatan6 detection rate | >70% | Adversarial tournament |
| Sigma rule compilation rate | >90% | sigma-cli validate |
| Alert triage TP/FP accuracy | >95% | Production metrics |
| First model deployment | ≤14 days from start | Calendar |
| Monthly operating cost | ≤$1,500 | AWS/GCP billing |
| Inference latency (sidecar) | <100ms | p99 |
| Inference latency (shared) | <2s | p99 |
