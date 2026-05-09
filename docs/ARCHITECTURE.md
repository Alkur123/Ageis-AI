# Chakravyuha — System Architecture

[![FastAPI](https://img.shields.io/badge/Backend-FastAPI_0.110-009688?style=flat-square&logo=fastapi)](https://fastapi.tiangolo.com)
[![ONNX](https://img.shields.io/badge/Inference-ONNX_Runtime-005CED?style=flat-square&logo=onnx&logoColor=white)](https://onnxruntime.ai)
[![FAISS](https://img.shields.io/badge/Retrieval-FAISS_IndexFlatIP-blue?style=flat-square)](https://github.com/facebookresearch/faiss)
[![React](https://img.shields.io/badge/Frontend-React_19_+_Vite-61DAFB?style=flat-square&logo=react)](https://react.dev)
[![Docker](https://img.shields.io/badge/Containers-Docker_+_ECR-2496ED?style=flat-square&logo=docker)](https://docker.com)
[![MongoDB](https://img.shields.io/badge/Audit-MongoDB_Atlas-47A248?style=flat-square&logo=mongodb)](https://mongodb.com/atlas)
[![Redis](https://img.shields.io/badge/Cache-Redis_7-DC382D?style=flat-square&logo=redis)](https://redis.io)
[![PostgreSQL](https://img.shields.io/badge/Tenants-PostgreSQL_15-4169E1?style=flat-square&logo=postgresql&logoColor=white)](https://postgresql.org)
[![CloudFront](https://img.shields.io/badge/CDN-CloudFront_+_S3-FF9900?style=flat-square&logo=amazonaws)]()
[![Latency](https://img.shields.io/badge/Decision_latency-~16ms_CPU-22c55e?style=flat-square)]()

> Full architecture reference for **Chakravyuha** — production AI governance infrastructure.
> 11 rings · 4 planes · 6 regulation plugins · 97 clauses · 35 routes · CPU-only.

---

## Table of Contents

- [System Overview](#system-overview)
- [The 4 Planes](#the-4-planes)
- [11-Ring Pipeline](#11-ring-pipeline)
- [Backend Engine Layout](#backend-engine-layout)
- [Semantic Engine Internals](#semantic-engine-internals)
- [Constitutional Retrieval (Ring 3)](#constitutional-retrieval-ring-3)
- [Policy Engine + Composer](#policy-engine--composer)
- [Atomic Commit Gate (Ring 8)](#atomic-commit-gate-ring-8)
- [Audit Vault (Ring 11)](#audit-vault-ring-11)
- [HITL Queue (Ring 10)](#hitl-queue-ring-10)
- [Multi-Tenant Architecture](#multi-tenant-architecture)
- [Frontend Architecture](#frontend-architecture)
- [SDK Architecture](#sdk-architecture)
- [Data Flow Diagrams](#data-flow-diagrams)
- [Database Schema](#database-schema)
- [AWS Deployment Architecture](#aws-deployment-architecture)
- [Security Architecture](#security-architecture)
- [Observability](#observability)
- [Failure Modes & Fallbacks](#failure-modes--fallbacks)

---

## System Overview

Chakravyuha is a four-plane system: **Edge**, **Control**, **Reasoning**, **Data**. Each plane is independently scalable; the interfaces between them are stable enough that any plane can be swapped without rewriting the others.

```mermaid
flowchart TD
    subgraph T1["EDGE PLANE — Global"]
        F1["<b>React 19 SPA</b><br/>Tailwind · Vite build · 5-tab intelligence console<br/>S3 + CloudFront (HTTPS · 1-year static cache)"]
        F2["<b>Python SDK v1.0.0</b><br/>pip install aegis-ai-sdk<br/>direct · sync · @govern decorator · CLI"]
    end

    subgraph T2["CONTROL PLANE — Stateless API"]
        A1["<b>FastAPI 0.110 + Gunicorn (UvicornWorker)</b><br/>35 routes · Pydantic validated<br/>4-layer middleware stack (LIFO)<br/>Rate-limited · X-Forwarded-For aware"]
    end

    subgraph T3["REASONING PLANE — 11 Rings"]
        E1["<b>engine/pipeline.py</b><br/>11-ring orchestrator · async fan-out"]
        E2["<b>Semantic + Retrieval</b><br/>FAISS IndexFlatIP · ONNX MiniLM-L6-v2"]
        E3["<b>Policy Composer</b><br/>9 deterministic rules · 6 regulation plugins"]
        E4["<b>Stability + Verify</b><br/>3-paraphrase concurrent · post-gen verifier"]
        E5["<b>LLM Service</b><br/>Groq Llama 4 Scout → Ollama fallback"]
    end

    subgraph T4["DATA PLANE — Stateful Stores"]
        direction LR
        D1[("<b>MongoDB Atlas</b><br/>Audit logs · Heatmap")]
        D2[("<b>RDS PostgreSQL 15</b><br/>Tenant registry")]
        D3[("<b>ElastiCache Redis 7</b><br/>Sessions · HITL queue · Rate limit")]
        D4[("<b>Audit Vault</b><br/>SHA-256 hash chain<br/>(MongoDB-backed)")]
    end

    T1 -- "HTTPS REST + X-API-Key" --> T2
    T2 --> T3
    T3 --> D1
    T3 --> D2
    T3 --> D3
    T3 --> D4

    classDef edge fill:#3b82f6,stroke:#1d4ed8,stroke-width:2px,color:#fff;
    classDef control fill:#10b981,stroke:#047857,stroke-width:2px,color:#fff;
    classDef reasoning fill:#8b5cf6,stroke:#6d28d9,stroke-width:2px,color:#fff;
    classDef data fill:#f59e0b,stroke:#b45309,stroke-width:2px,color:#000;

    class F1,F2 edge;
    class A1 control;
    class E1,E2,E3,E4,E5 reasoning;
    class D1,D2,D3,D4 data;
```

---

## The 4 Planes

| Plane | Statefulness | Scaling Strategy | Failure Domain |
|---|---|---|---|
| **Edge** | Stateless | CDN edge nodes, infinite horizontal | Per-region |
| **Control** | Stateless | ECS Fargate auto-scaling on CPU | Per-task |
| **Reasoning** | In-memory state (FAISS, ONNX) | Vertical first, then sharded by tenant | Per-pod |
| **Data** | Persistent | Managed services (Atlas, ElastiCache, RDS) | Per-cluster |

**Why split this way:** the Reasoning plane is the only place that holds large in-memory state (the FAISS index, ONNX model, and PII regex graph). Splitting it from the Control plane lets Chakravyuha autoscale lightweight HTTP handling without thrashing the embeddings cache.

---

## 11-Ring Pipeline

The orchestrator is `engine/pipeline.analyze_query()`. Order, concurrency, and skip conditions are explicit:

```mermaid
flowchart TD
    Q["<b>INPUT</b> { query, session_id, tenant_id }"]:::input

    subgraph Pre["PRE-RETRIEVAL"]
        R0["<b>Ring 0 · Normalizer</b><br/>NFKC · leet decode · Indic numerals · zero-width strip"]
        PII["<b>PII Redaction</b><br/>11 entity types — AADHAAR · PAN · IFSC · UPI · CARD · SSN · …"]
        ATK["<b>Attack Heuristics</b><br/>16 signal vectors (regex + keyword)<br/>jailbreak · system_exfil · regulatory_evasion · pii_seek · …"]
        IG["<b>Intent Guard</b><br/>3-layer multilingual<br/>known phrases → 15 Unicode scripts → non-ASCII ratio"]
        JUR["<b>Jurisdiction</b><br/>MaxMind + tenant-declared"]
    end

    subgraph Concurrent["RETRIEVAL — asyncio.gather()"]
        R2["<b>Ring 2 · FAISS Semantic + Claims</b><br/>2,416 vectors · 384-dim · k=10<br/>quadratic rank-weighted voting<br/>asyncio.to_thread"]
        R3["<b>Ring 3 · Constitutional</b><br/>97 clauses · 6 plugins<br/>importlib + pkgutil"]
    end

    RES["<b>Resolution</b><br/>semantic + signal fusion · category arbitration"]:::ring
    R4["<b>Ring 4 · Session Memory</b><br/>Redis · temporal-decay risk · cumulative tracker"]:::ring
    POL["<b>Policy Engine</b><br/>9 rules CRIT-01 → GEN-00<br/>+ 5-rule multi-reg composer"]:::ring
    TR["<b>Decision Trace</b><br/>causal · winner + runner-up · confidence margin"]:::ring

    R6["<b>Ring 6 · LLM Generation</b><br/>Groq Llama 4 Scout 17B → Ollama fallback"]:::llm
    R5["<b>Ring 5 · Stability</b><br/>3 paraphrases · concurrent · category disagreement → HITL"]:::ring
    R7["<b>Ring 7 · Post-Gen Verify</b><br/>re-classify LLM output · PII leakage scan"]:::ring
    R8["<b>Ring 8 · Atomic Commit</b><br/>3-way AND gate<br/>SHADOW · STRICT · HITL"]:::gate

    R10["<b>Ring 10 · HITL Enqueue</b><br/>Redis priority · SLA timer · claim/resolve/escalate"]:::hitl
    R11["<b>Ring 11 · Audit Vault</b><br/>SHA-256 hash chain · stream-verifiable · OOM-safe"]:::vault

    OUT["<b>OUTPUT</b> GovernanceDecision { decision, risk_score, regulations_triggered, citations, redactions, trace }"]:::output

    Q --> R0 --> PII --> ATK --> IG --> JUR --> R2 & R3
    R2 --> RES
    R3 --> RES
    RES --> R4 --> POL --> TR
    TR --> R6 --> R7
    TR -.-> R5
    R5 --> R8
    R7 --> R8
    R8 -- BLOCK / SUPPORT / HITL --> R10
    R8 --> R11
    R8 --> OUT

    classDef input fill:#3b82f6,stroke:#1d4ed8,stroke-width:2px,color:#fff;
    classDef pre fill:#0ea5e9,stroke:#0284c7,stroke-width:2px,color:#fff;
    classDef ring fill:#4f46e5,stroke:#3730a3,stroke-width:2px,color:#fff;
    classDef llm fill:#10b981,stroke:#047857,stroke-width:2px,color:#fff;
    classDef gate fill:#f59e0b,stroke:#b45309,stroke-width:2px,color:#000;
    classDef hitl fill:#ec4899,stroke:#be185d,stroke-width:2px,color:#fff;
    classDef vault fill:#8b5cf6,stroke:#6d28d9,stroke-width:2px,color:#fff;
    classDef output fill:#22c55e,stroke:#16a34a,stroke-width:2px,color:#fff;

    class R0,PII,ATK,IG,JUR pre;
    class R2,R3,RES,R4,POL,TR,R5,R7 ring;
    class R6 llm;
    class R8 gate;
    class R10 hitl;
    class R11 vault;
```

**Concurrency model:**
- FAISS retrieval is wrapped in `asyncio.to_thread()` so it never blocks the event loop.
- Ring 2 (semantic) and Ring 3 (constitutional retrieval) run inside `asyncio.gather()` — wall-clock latency is `max(t_semantic, t_ring3)`, not their sum.
- Ring 5 stability check fires three paraphrase passes concurrently.

**Risk score formula:**
```
risk = semantic(0.6) + session(0.2) + policy(0.2)
```

**Modes** (set via `GOVERNANCE_MODE`):
- `SHADOW` — log only; for A/B baselines and dark-launches.
- `STRICT` — production block-on-violation.
- `HITL` — every borderline decision routed to human reviewer with SLA timer.

---

## Backend Engine Layout

```mermaid
flowchart LR
    subgraph Backend["backend/"]
        direction TB
        S["<b>server.py</b><br/>FastAPI app · 35 routes<br/>middleware stack · lifespan startup"]

        subgraph Engine["engine/"]
            direction TB
            P["<b>pipeline.py</b><br/>11-ring orchestrator"]
            NORM["<b>normalizer.py</b><br/>NFKC · leet · Indic"]
            RED["<b>redactor.py</b><br/>11 PII regex types"]
            ATTK["<b>attack_heuristics.py</b><br/>16 signal vectors"]
            IG["<b>intent_guard.py</b><br/>multilingual 3-layer"]
            JUR["<b>jurisdiction.py</b><br/>MaxMind + tenant"]
            SE["<b>semantic_engine.py</b><br/>FAISS + ONNX"]
            R3R["<b>ring3_retrieval.py</b><br/>97-clause retriever"]
            CP["<b>claim_parser.py</b>"]
            SESS["<b>session.py</b><br/>Redis temporal decay"]
            POL["<b>policy.py</b><br/>9 deterministic rules"]
            COMP["<b>policy_composer.py</b><br/>5-rule composition"]
            R5["<b>ring5_stability.py</b>"]
            R7["<b>ring7.py</b><br/>post-gen verify"]
            R8["<b>ring8.py</b><br/>atomic commit gate"]
            R10["<b>ring10_hitl.py</b><br/>Redis priority queue"]
            R11["<b>audit_vault.py</b><br/>SHA-256 hash chain"]
            LLM["<b>llm_service.py</b><br/>Groq + Ollama"]
            TEN["<b>tenant.py</b><br/>PG-backed registry"]
            EXP["<b>explainability.py</b>"]
            FED["<b>federated.py</b>"]
            PROM["<b>prometheus_metrics.py</b>"]
        end

        subgraph Plugins["regulations/"]
            direction TB
            DPDP["dpdp_2023.py"]
            GDPR["gdpr.py"]
            EUAI["eu_ai_act.py"]
            HIPAA["hipaa.py"]
            CCPA["ccpa.py"]
            SEBI["sebi_rbi.py"]
        end

        subgraph EvalDir["eval/"]
            direction TB
            ADV["adversarial_dataset_v3.json<br/>1,001 samples · 97 attacks"]
            ABENCH["harmful_behaviors.csv<br/>AdvBench (Zou et al. 2023)"]
            RUN["run_*_eval.py · run_comparison_benchmark.py"]
            RES["results/<br/>FINAL_EVAL_REPORT_chakravyuha_v3.md"]
        end
    end

    S -->|invokes| P
    P --> NORM --> RED --> ATTK --> IG --> JUR
    P --> SE
    P --> R3R
    R3R -.dynamic load.-> Plugins
    P --> SESS
    P --> POL --> COMP
    COMP -.composes.-> Plugins
    P --> R5
    P --> LLM --> R7
    R5 & R7 --> R8 --> R10
    R8 --> R11
    S --> TEN
    S --> PROM

    classDef folder fill:#0f172a,stroke:#334155,stroke-width:2px,color:#fff;
    classDef file fill:#1e293b,stroke:#475569,stroke-width:1px,color:#fff;
    classDef orchestrator fill:#3b82f6,stroke:#2563eb,stroke-width:2px,color:#fff;
    classDef plugin fill:#8b5cf6,stroke:#6d28d9,stroke-width:2px,color:#fff;

    class Backend,Engine,Plugins,EvalDir folder;
    class S,NORM,RED,ATTK,IG,JUR,SE,R3R,CP,SESS,POL,COMP,R5,R7,R8,R10,R11,LLM,TEN,EXP,FED,PROM,DPDP,GDPR,EUAI,HIPAA,CCPA,SEBI,ADV,ABENCH,RUN,RES file;
    class P orchestrator;
```

---

## Semantic Engine Internals

`engine/semantic_engine.py` is the primary classifier.

```mermaid
flowchart TD
    Q["<b>Normalized Query</b>"]:::input
    EMB["<b>ONNX Tokenize + Embed</b><br/>all-MiniLM-L6-v2 · 384-dim · ~15ms CPU"]:::ring
    IDX["<b>FAISS IndexFlatIP</b><br/>2,416 labeled vectors · cosine sim<br/>k=10 nearest neighbors · <2ms"]:::ring
    RW["<b>Rank-Weighted Voting</b><br/>weight_r = (k+1−r)²<br/>top match = 100 units, rank-10 = 1 unit"]:::ring
    THR["<b>Per-Category Thresholds</b><br/>SELF_HARM 0.44 · SEXUAL 0.40 · PII 0.40<br/>SYSTEM_EXFIL 0.38 · MEDICAL 0.48 · …"]:::ring
    HARD["<b>Hard-Block Categories</b><br/>SELF_HARM, SEXUAL — informational dampening disabled"]:::ring
    OUT["<b>Category + Confidence</b><br/>{ winner, runner_up, margin, signals[] }"]:::output

    Q --> EMB --> IDX --> RW --> THR --> HARD --> OUT

    classDef input fill:#3b82f6,stroke:#1d4ed8,stroke-width:2px,color:#fff;
    classDef ring fill:#4f46e5,stroke:#3730a3,stroke-width:2px,color:#fff;
    classDef output fill:#22c55e,stroke:#16a34a,stroke-width:2px,color:#fff;
```

**Why rank-weighted voting?** Standard FAISS k-NN classifiers use flat similarity-sum voting, which suffers from *cluster bias*: a cluster of moderately-similar harmful examples can outvote a single high-confidence SAFE match. Quadratic rank weighting `(k+1−rank)²` makes the top match 100× more influential than the least-similar match — driving false positives from 35 → 4 in a single change.

**Dynamic per-tenant thresholds:** every tenant can supply `threshold_overrides` to relax or tighten any category, while still going through the legal-floor composition rules.

---

## Constitutional Retrieval (Ring 3)

`engine/ring3_retrieval.py` runs FAISS over the regulatory corpus rather than the harm corpus.

```mermaid
flowchart LR
    NQ["Normalized Query"]
    RING3["<b>Ring 3 Retriever</b><br/>FAISS IndexFlatIP over 97 clauses<br/>k=5 most-relevant clauses"]

    subgraph Plugins["6 Regulation Plugins (importlib + pkgutil)"]
        direction TB
        DPDP["<b>dpdp_2023.py</b><br/>Aadhaar §4 · biometric §9"]
        GDPR["<b>gdpr.py</b><br/>Art 4(1) · Art 9 · Art 22"]
        EUAI["<b>eu_ai_act.py</b><br/>Art 5 prohibited · Art 6 high-risk"]
        HIPAA["<b>hipaa.py</b><br/>§164.514 PHI · safe harbor"]
        CCPA["<b>ccpa.py</b><br/>§1798 sale-of-data"]
        SEBI["<b>sebi_rbi.py</b><br/>Insider Trading Regs · UPI Frd."]
    end

    OUT["<b>Citations[]</b><br/>{ regulation, clause_id, severity, penalty, text }"]

    NQ --> RING3 --> Plugins --> OUT

    classDef ring fill:#4f46e5,stroke:#3730a3,stroke-width:2px,color:#fff;
    classDef plugin fill:#8b5cf6,stroke:#6d28d9,stroke-width:2px,color:#fff;
    classDef output fill:#22c55e,stroke:#16a34a,stroke-width:2px,color:#fff;
    class RING3 ring;
    class DPDP,GDPR,EUAI,HIPAA,CCPA,SEBI plugin;
    class OUT output;
```

**Plugin contract:** each plugin exposes `clauses: list[Clause]` and `match(category, signals) -> list[CitedClause]`. New regulations are added by dropping a file in `regulations/` — no core changes.

---

## Policy Engine + Composer

`engine/policy.py` runs nine deterministic rules; `engine/policy_composer.py` composes results across plugins.

| Rule ID | Trigger | Default Action |
|---|---|---|
| `CRIT-01` | Hard-block category + high confidence | BLOCK + cite |
| `CRIT-02` | PII exploitation pattern | BLOCK + cite |
| `HIGH-01` | Regulatory evasion signals | BLOCK + cite |
| `HIGH-02` | System exfiltration / jailbreak | BLOCK + cite |
| `MED-01` | Borderline + ambiguous intent | HITL or SUPPORT |
| `MED-02` | Session escalation pattern | HITL |
| `LOW-01` | Soft policy tag | ALLOW + redact |
| `INFO-01` | Educational framing on non-hard-block | ALLOW |
| `GEN-00` | Default | ALLOW |

**5-rule composition** when multiple plugins fire on the same query:
1. **Most-restrictive wins** — DPDP_CRITICAL beats GDPR_PERSONAL
2. **Legal floor** — no plugin can lower another's minimum bar
3. **Tenant restricts only** — tenant config can tighten, never relax
4. **All applicable plugins audited** — not just the winner
5. **Full citation in every block** — every block message names its article

---

## Atomic Commit Gate (Ring 8)

`engine/ring8.py` is the final 3-way AND gate. Without it, an attacker could walk a query around any *single* check.

```mermaid
flowchart LR
    IN["<b>Inputs</b><br/>(a) input governance verdict<br/>(b) Ring 5 stability verdict<br/>(c) Ring 7 post-gen verifier verdict"]
    GATE{"<b>3-way AND</b><br/>all must agree"}
    SHADOW["<b>SHADOW</b><br/>log only · pass-through"]
    STRICT["<b>STRICT</b><br/>BLOCK on disagreement"]
    HITL["<b>HITL</b><br/>enqueue with SLA"]

    IN --> GATE
    GATE -->|GOVERNANCE_MODE=SHADOW| SHADOW
    GATE -->|GOVERNANCE_MODE=STRICT| STRICT
    GATE -->|GOVERNANCE_MODE=HITL| HITL

    classDef gate fill:#f59e0b,stroke:#b45309,stroke-width:2px,color:#000;
    class GATE gate;
```

**Disagreement is the safety surface.** A single ring approving with high confidence is not enough — an adversarially crafted query can fool any one ring, but rarely all three simultaneously.

---

## Audit Vault (Ring 11)

`engine/audit_vault.py` writes every decision to a SHA-256 hash chain so any post-hoc tampering is detectable.

```
entry_n.hash = SHA-256( entry_n.payload || entry_{n-1}.hash )
```

**OOM-safe construction:**
- Only `_chain_head` (32 bytes) is held in memory
- Verification reads MongoDB cursor in a stream — never loads the chain into RAM
- Independent verifier exposed at `GET /api/vault/verify`
- Legal discovery export at `GET /api/vault/export` (admin-key + date range)

```mermaid
sequenceDiagram
    participant P as Pipeline
    participant V as audit_vault.py
    participant M as MongoDB

    P->>V: append(entry)
    V->>V: prev_hash = _chain_head
    V->>V: new_hash = sha256(entry || prev_hash)
    V->>M: insertOne({ ...entry, prev_hash, hash })
    V->>V: _chain_head = new_hash

    Note over V,M: Verification (stream-only)
    V->>M: cursor = find().sort(_id, ASC)
    loop for each entry
        V->>V: recompute hash, compare
    end
    V-->>P: verified · breaks_at?
```

---

## HITL Queue (Ring 10)

`engine/ring10_hitl.py` is a Redis priority queue with SLA timers.

| State | Description |
|---|---|
| `pending` | Newly enqueued, not yet claimed |
| `in_review` | Claimed by an admin |
| `resolved` | Approved or denied with note |
| `breached` | SLA timer expired |
| `escalated` | Bumped to higher priority lane |

Endpoints: `/api/hitl/{queue,stats,breaches,claim,resolve}` — all admin-key gated, all audit-logged.

---

## Multi-Tenant Architecture

`engine/tenant.py` is a PostgreSQL-backed registry with a bounded LRU cache.

```mermaid
flowchart LR
    REQ["Request<br/>X-API-Key: ..."] --> AUTH
    AUTH["<b>Auth Middleware</b><br/>SHA-256(key + pepper)<br/>compare to PG row"] --> CACHE
    CACHE["<b>_BoundedTTLCache</b><br/>maxsize=1000<br/>TTL=5min"] --> PG[("RDS PostgreSQL 15<br/>tenant_schema.sql")]
    AUTH --> P["Pipeline · tenant_id pinned"]

    classDef ring fill:#4f46e5,stroke:#3730a3,stroke-width:2px,color:#fff;
    classDef store fill:#f59e0b,stroke:#b45309,stroke-width:2px,color:#000;
    class AUTH,CACHE ring;
    class PG store;
```

- **Hashing:** SHA-256 + per-tenant pepper (defends against BOLA / rainbow tables)
- **Bounded cache:** LRU eviction prevents memory growth on hostile tenant churn
- **Demo fallback:** if `POSTGRES_URL` is unset the registry runs in single-tenant demo mode (the live demo runs in this mode today; production deployment switches it on)
- **Per-tenant policy patch:** `PATCH /api/admin/tenants/{id}/policy` updates threshold overrides + plugin enablement; all changes audited.

---

## Frontend Architecture

```mermaid
flowchart LR
    subgraph FE["frontend/src/"]
        direction TB
        APP["<b>App.jsx</b><br/>Owns ALL state · 5-tab routing"]

        subgraph Left["Chat Pane"]
            CW["ChatWindow · ChatMessage · DecisionBanner<br/>InputBox · RiskBar"]
        end

        subgraph Right["5-Tab Intelligence Console"]
            T1["<b>RISK</b><br/>RiskGauge · DecisionComparator<br/>SessionPanel · RiskGraph"]
            T2["<b>EVIDENCE</b><br/>SignalBreakdown · IntentGuardSummary<br/>PolicyComposedSummary · EvidencePanel<br/>Ring3 clauses · Timeline"]
            T3["<b>MONITOR</b><br/>MetricsPanel · HeatmapPanel · AuditLogs"]
            T4["<b>SYSTEM</b><br/>StatsPanel · VaultPanel · RegulationsPanel"]
            T5["<b>HITL</b><br/>HitlPanel — queue · breaches · claim/resolve"]
        end

        API["<b>api.js</b><br/>fetch wrapper · all 35 endpoints<br/>VITE_API_BASE · X-API-Key"]

        APP --> CW
        APP --> T1 & T2 & T3 & T4 & T5
        CW & T1 & T2 & T3 & T4 & T5 -.-> API
    end

    classDef root fill:#f43f5e,stroke:#e11d48,stroke-width:2px,color:#fff;
    classDef pane fill:#0ea5e9,stroke:#0284c7,stroke-width:2px,color:#fff;
    classDef util fill:#10b981,stroke:#047857,stroke-width:2px,color:#fff;

    class APP root;
    class CW,T1,T2,T3,T4,T5 pane;
    class API util;
```

**State ownership:** `App.jsx` owns *everything*. No component manages state it doesn't own — every panel is independently testable and prop-driven.

**Build target:** Vite → `dist/` → `aws s3 sync dist/ s3://<bucket> --delete` → CloudFront invalidate. Long-lived `/assets/*` cache, no-store on the API behavior.

---

## SDK Architecture

```mermaid
flowchart LR
    subgraph SDK["sdk/python/aegis/"]
        direction TB
        CLI["<b>_cli.py</b><br/>aegis-health"]
        CL["<b>client.py</b><br/>AegisClient<br/>analyze · analyze_sync · @govern"]
        MOD["<b>models.py</b><br/>AnalysisResult · IntentGuardResult<br/>JurisdictionInfo · PolicyComposed<br/>ClaimAnalysis · RegulationInfo<br/>VaultStats · HitlStats · EngineStats"]
        SH["<b>openai_shim.py</b><br/>govern() decorator wrappers"]
        VER["<b>_version.py</b>"]
    end

    APP["User App<br/>(OpenAI / Anthropic / custom)"] --> CL
    CL --> HX["httpx<br/>3-retry exp. backoff<br/>TLS-required (non-loopback)"]
    HX --> SRV["FastAPI<br/>POST /api/analyze"]

    classDef pkg fill:#3b82f6,stroke:#1d4ed8,stroke-width:2px,color:#fff;
    classDef ext fill:#8b5cf6,stroke:#6d28d9,stroke-width:2px,color:#fff;

    class CLI,CL,MOD,SH,VER pkg;
    class APP,HX,SRV ext;
```

**Optional extras** in `pyproject.toml`:
- `[openai]` → adds `openai>=1.0.0`
- `[anthropic]` → adds `anthropic>=0.25.0`
- `[full]` → both shims

---

## Data Flow Diagrams

### Decision flow — query governance

```mermaid
flowchart TD
    U["User types query"] --> API["POST /api/analyze"]
    API --> PRE["Pre-retrieval rings<br/>(R0 · PII · Attack · Intent · Jurisdiction)"]
    PRE --> CONC["asyncio.gather(R2, R3)"]
    CONC --> RES["Resolution + R4 session"]
    RES --> POL["Policy + Composer"]

    POL --> FAST{"risk < 40?"}
    FAST -- "yes (cold ALLOW)" --> SKIP["Skip Ring 5 + Ring 7<br/>(fast-path ALLOW)"]
    FAST -- "no" --> R5["Ring 5 stability"]

    SKIP --> R6["Ring 6 LLM"]
    R5 --> R6
    R6 --> R7["Ring 7 post-gen verify"]

    R7 --> R8["Ring 8 atomic AND"]
    R5 --> R8

    R8 -- ALLOW --> OK["Verified response → user"]
    R8 -- BLOCK / SUPPORT --> BLOCK["Block message + citation"]
    R8 -- HITL --> Q["Ring 10 HITL queue"]

    R8 --> V["Ring 11 audit vault"]

    classDef ring fill:#4f46e5,stroke:#3730a3,stroke-width:2px,color:#fff;
    classDef ok fill:#22c55e,stroke:#16a34a,stroke-width:2px,color:#fff;
    classDef block fill:#ef4444,stroke:#b91c1c,stroke-width:2px,color:#fff;
    classDef store fill:#8b5cf6,stroke:#6d28d9,stroke-width:2px,color:#fff;
    class PRE,CONC,RES,POL,R5,R6,R7,R8 ring;
    class OK ok;
    class BLOCK block;
    class Q,V store;
```

### Compliance report flow

```mermaid
flowchart LR
    REQ["GET /api/compliance/report<br/>?from=...&to=...&format=html"] --> AGG["Aggregator<br/>scan MongoDB audit collection"]
    AGG --> SECS["4 sections<br/>1. Summary · 2. Per-regulation status<br/>3. Penalty exposure · 4. Remediation"]
    SECS --> RNDR{"format?"}
    RNDR -- json --> J["application/json"]
    RNDR -- html --> H["application/html<br/>(printable, board-ready)"]
```

---

## Database Schema

### MongoDB · `governance_logs`

**`audit_logs`**
```json
{
  "_id": "ObjectId",
  "tenant_id": "acme-corp",
  "session_id": "sess-...",
  "query": "raw query (PII redacted before write)",
  "decision": "BLOCK",
  "risk_score": 0.94,
  "confidence": 0.88,
  "regulations_triggered": ["DPDP_2023", "GDPR"],
  "category": "PII",
  "pii_entities": ["AADHAAR"],
  "explanation": "DPDP Act 2023 §4 — biometric exploitation…",
  "trace": { "winner": "...", "runner_up": "...", "margin": 0.21 },
  "timestamp": "2026-05-09T10:30:00Z",
  "processing_time_ms": 16,
  "ring_timings_ms": { "R0":0.1, "R2":15.4, "R3":12.1, "R7":13.8, "R8":0.2 }
}
```

**`audit_vault`** (hash chain)
```json
{
  "_id": "ObjectId",
  "seq": 423901,
  "tenant_id": "acme-corp",
  "payload_hash": "sha256:...",
  "prev_hash": "sha256:...",
  "hash": "sha256:..."
}
```

**`heatmap_counters`** — per-(tenant, regulation, category, decision, day) counters.

### PostgreSQL · `aegis`

`engine/tenant_schema.sql` defines `tenants`, `tenant_keys`, `tenant_policies`, `tenant_audit`. Bounded LRU cache fronts hot reads.

---

## AWS Deployment Architecture

```mermaid
flowchart TD
    User(("🌐 End User<br/>Browser"))

    subgraph Dev["Developer Machine"]
        DB1["docker build<br/>+ docker push"]
        NB1["npm run build<br/>aws s3 sync dist/"]
    end

    subgraph AWSCloud["AWS Cloud (ap-south-1 / us-east-1)"]
        direction TB

        subgraph CDN["CloudFront Edge Network"]
            CFF["<b>CF Frontend</b><br/>Distribution E51JSAKEC4QL9<br/>1-year static cache"]
            CFA["<b>CF API Proxy</b><br/>Distribution E8DVT0NN727K6<br/>no-store"]
        end

        ECR["<b>ECR Registry</b><br/>aegis-backend:latest"]
        S3B["<b>S3 Bucket</b><br/>ages-ai-frontend"]
        ALB["<b>ALB</b><br/>HTTPS 443 · ACM cert"]

        subgraph Compute["ECS Fargate / EC2 Docker"]
            APP["<b>chakra-backend</b><br/>FastAPI + Gunicorn (2 workers)<br/>Health: /api/health · 90s start period"]
            REDIS["<b>chakra-redis</b><br/>redis:7-alpine"]
        end

        SM["<b>Secrets Manager</b>"]
        EC[("<b>ElastiCache Redis</b>")]
        RDS[("<b>RDS PostgreSQL 15</b>")]
    end

    subgraph Ext["External SaaS"]
        Mongo[("<b>MongoDB Atlas</b>")]
        Groq["<b>Groq Cloud</b><br/>Llama 4 Scout 17B"]
    end

    DB1 -.push.-> ECR
    NB1 -.sync.-> S3B
    ECR -.pull.-> APP

    User -- HTTPS --> CFF & CFA
    CFF --> S3B
    CFA --> ALB --> APP
    APP <--> REDIS
    APP --> EC & RDS & Mongo & Groq
    SM -. inject .-> APP

    classDef dev fill:#475569,stroke:#334155,stroke-width:2px,color:#fff;
    classDef aws fill:#f59e0b,stroke:#b45309,stroke-width:2px,color:#000;
    classDef compute fill:#3b82f6,stroke:#1d4ed8,stroke-width:2px,color:#fff;
    classDef store fill:#10b981,stroke:#047857,stroke-width:2px,color:#fff;
    classDef user fill:#ec4899,stroke:#be185d,stroke-width:2px,color:#fff;
    classDef ext fill:#8b5cf6,stroke:#6d28d9,stroke-width:2px,color:#fff;

    class DB1,NB1 dev;
    class CFF,CFA,ECR,ALB,SM aws;
    class APP,REDIS compute;
    class S3B,EC,RDS store;
    class User user;
    class Mongo,Groq ext;
```

### Deployment pipeline

```mermaid
flowchart LR
    subgraph Local["Local"]
        B1["docker build -t aegis-backend ./backend"]
        B2["docker tag → ECR"]
        B3["docker push"]
        F1["npm run build"]
        F2["aws s3 sync dist/ s3://ages-ai-frontend --delete"]
        F3["aws cloudfront create-invalidation"]
        B1 --> B2 --> B3
        F1 --> F2 --> F3
    end

    subgraph AWSCloud["AWS"]
        ECR["ECR<br/>aegis-backend:latest"]
        ECS["ECS Fargate task<br/>force-new-deployment"]
        S3B["S3 ages-ai-frontend"]
        CF["CloudFront"]
    end

    B3 --> ECR --> ECS
    F2 --> S3B
    F3 --> CF
```

Full step-by-step in [`backend/AWS_DEPLOYMENT_PLAN.md`](../backend/AWS_DEPLOYMENT_PLAN.md).

---

## Security Architecture

```mermaid
flowchart TD
    subgraph Sec["LAYERED DEFENSES"]
        direction TB
        L1["<b>Layer 1 — Edge</b><br/>CloudFront enforces TLS 1.2+ · HTTP→HTTPS redirect"]
        L2["<b>Layer 2 — WAF</b><br/>infra/waf.tf — managed rule sets · SQLi · XSS · bot mgmt"]
        L3["<b>Layer 3 — Body cap</b><br/>RequestSizeLimitMiddleware (pure ASGI · 2 MiB)"]
        L4["<b>Layer 4 — Sanitize</b><br/>SanitizeBodyMiddleware (control char strip)"]
        L5["<b>Layer 5 — Rate limit</b><br/>slowapi · X-Forwarded-For aware (ALB-friendly)<br/>2000/min default"]
        L6["<b>Layer 6 — Auth</b><br/>X-API-Key · SHA-256 + per-tenant pepper<br/>Admin via ADMIN_API_KEY"]
        L7["<b>Layer 7 — CORS</b><br/>CORS_ORIGINS allowlist · LIFO middleware order<br/>(CORS added last → outermost)"]
        L8["<b>Layer 8 — Tenant isolation</b><br/>tenant_id pinned on every query · audit-scoped reads"]
        L9["<b>Layer 9 — Audit vault</b><br/>SHA-256 hash chain · stream-verifiable"]
        L10["<b>Layer 10 — Container isolation</b><br/>Backend + Redis in separate containers · Redis private subnet"]
        L1-->L2-->L3-->L4-->L5-->L6-->L7-->L8-->L9-->L10
    end

    classDef layer fill:#1e293b,stroke:#cbd5e1,stroke-width:2px,color:#fff;
    class L1,L2,L3,L4,L5,L6,L7,L8,L9,L10 layer;
```

**Secret hygiene:** all secrets via `.env` (gitignored), AWS Secrets Manager in production. `.dockerignore` excludes `.env` from the image. No secrets in code, no secrets in image.

---

## Observability

| Channel | Endpoint / Surface |
|---|---|
| **JSON metrics** | `GET /api/metrics` — requests, errors, P50/P95/P99 latency |
| **Prometheus** | `GET /metrics` — text exposition (counter / histogram / gauge) |
| **Per-ring timings** | embedded in every `audit_logs` entry (`ring_timings_ms`) |
| **Heatmap** | `GET /api/heatmap` — per-(regulation, category, decision) counters |
| **Vault verification** | `GET /api/vault/verify` — full hash-chain integrity check |
| **HITL stats** | `GET /api/hitl/stats` — queue depth, SLA breach count |
| **Health** | `GET /api/health` — engine_ready, env status, tenant |
| **Stats** | `GET /api/stats` — FAISS vectors loaded, Ring3 clauses, plugin count |

---

## Failure Modes & Fallbacks

Every dependency has a defined degradation path. **The engine never returns 500 because a backing service is down.**

```mermaid
flowchart LR
    F1["MongoDB write fails"] --> B1["Audit dropped silently<br/>request served · pipeline continues"]
    F2["Groq API unreachable"] --> B2["Ollama fallback<br/>or template explanation"]
    F3["Redis unavailable"] --> B3["In-memory session fallback<br/>rate limit relaxed (best-effort)"]
    F4["FAISS load fails"] --> B4["Pipeline refuses to start<br/>(fail-closed at boot)"]
    F5["MaxMind DB missing"] --> B5["Tenant-declared jurisdiction<br/>(audit logs the fallback)"]
    F6["POSTGRES_URL unset"] --> B6["Demo single-tenant mode<br/>(no multi-tenant features)"]

    classDef fail fill:#ef4444,stroke:#b91c1c,stroke-width:2px,color:#fff;
    classDef ok fill:#f59e0b,stroke:#b45309,stroke-width:2px,color:#000;
    class F1,F2,F3,F4,F5,F6 fail;
    class B1,B2,B3,B4,B5,B6 ok;
```

**The one fail-closed rule:** if the FAISS index can't load at startup, the process refuses to serve traffic. There is no degraded governance — either we're live with full pipeline, or we don't accept queries. This is enforced by `engine/startup_validator.py`.

---

*Chakravyuha V3 — Production AI governance infrastructure · CPU-only · ~16ms decisions · 6 regulations · 35 routes · [Back to README](../README.md)*
