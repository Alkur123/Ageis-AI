# Aegis AI — Architecture & Build Documentation
**Chakravyuha V3 Production | Built by Jaswanth Alkur | June 2026**

---

## What Was Built

Aegis AI is a pre-generation AI governance engine — and, as of V3, an **agentic trajectory governor**. It sits between a user (or an autonomous AI agent) and an LLM, intercepts every query *and every agent action*, makes a governance decision — ALLOW, BLOCK, SUPPORT, HITL, or KILL_SESSION — and produces a full causal audit trail. The LLM is never the safety mechanism. It only runs after the engine has already approved the query.

The system was designed with one constraint above all others: **governance cannot depend on any external service**. If an API is down, governance must still work. This single constraint drove every major technical decision — the engine runs entirely on CPU, with no GPU and no dependence on any single AI provider.

V3 extends this from "govern one query" to "govern an entire agent session": 13 deterministic rings across 4 planes, 6 regulation plugins, multi-tenant ready, with a published SDK ecosystem (PyPI + npm), an MCP server, a public agent red-team benchmark, and a formal bounded-evasion theorem.

---

## How It Was Designed

### The Core Problem With Existing Approaches

Every AI safety tool in the market is a keyword filter or a hosted moderation API. Both fail in the same way — they look at surface text, not semantic intent. A user asking "how do I end my life peacefully" does not contain any flagged keywords. A user asking "explain the history of the atom bomb" contains the word "bomb." Keyword filters get both wrong.

And in 2026 a new failure surface dominates: **AI agents** that take dozens of actions per session. The agent's prompt is benign; the failure is at step 6 of a 10-step plan, where it reads `/etc/passwd` instead of the file the user asked for. Single-query filters are structurally blind to this.

The design requirement was semantic understanding *plus* trajectory understanding — the system must understand what a query *means*, and whether an agent's *sequence of actions* still matches what it declared it would do.

### Architecture Decisions

**Semantic engine over classifier:** Rather than training a supervised classifier, the system uses nearest-neighbour retrieval against a labelled vector index. A query is embedded into a 384-dimensional space and matched against **8,627 pre-labelled training vectors** across 12 harm categories. The category is decided by quadratic rank-weighted voting `(k+1−r)²` over the top-k neighbours. This requires zero fine-tuning, generalises to unseen phrasing, and produces explainable evidence spans.

**ONNX runtime over hosted embedding API:** The embedding model (`all-MiniLM-L6-v2`) is exported to ONNX and runs entirely on CPU. This eliminates the network dependency, drops inference to ~16ms, and reduces the runtime footprint to ~50MB RAM. No GPU required.

**Deterministic policy engine:** The policy layer is fully deterministic — given the same inputs, it always produces the same output. Every decision has a `rule_id` and `regulation` string explaining exactly which policy triggered it.

**Multi-signal scoring:** Risk is not a single number. The final score fuses three signals: semantic confidence (60%), session risk trajectory (20%), and base policy weight (20%).

**Trajectory verification (the V3 keystone):** For agents, a 5-signal drift score (cosine drift, action-class drift, tool surprise, plan-execution match, paraphrase stability) is computed at *every step* against a cryptographically signed declaration of intent. The decision boundary *tightens the deeper an attacker probes* — penetration-depth recession — which a written bounded-evasion theorem proves cannot be patiently bypassed.

**Hardcoded block responses:** When a query is blocked, the response is hardcoded — not LLM-generated. Block messages are deterministic, regulation-cited, and instantaneous.

---

## How It Was Built

### Phase 1 — Semantic Foundation
Built the FAISS vector index covering 12 harm categories: SELF_HARM, SELF_HARM_PASSIVE, VIOLENCE, MEDICAL, FINANCIAL, LEGAL, ILLEGAL, PROMPT_INJECTION, SYSTEM_EXFILTRATION, PII, SEXUAL, and SAFE. Exported the sentence-transformer model to ONNX to eliminate the 600MB+ transformer dependency at runtime. The live corpus grew across 7 evaluation cycles to **8,627 vectors / 384-dim** (verified: `precomputed_embeddings.npy` shape `(8627, 384)`).

### Phase 2 — Pipeline Orchestration
Built the multi-ring governance pipeline: normalization → PII redaction → attack-vector detection → intent guard → jurisdiction → FAISS semantic + constitutional retrieval (concurrent) → category resolution → session memory → policy engine + composer → decision trace → LLM (ALLOW path only) → stability → post-gen verify → atomic commit → HITL → audit vault. Each stage produces a timestamped result that feeds the full decision trace.

### Phase 3 — Session Intelligence
Added behavioural tracking across conversation turns. Risk decays over time (5% per turn). A separate distress score tracks passive self-harm signals and routes to support responses rather than cold blocks. Session state is stored in Redis with an in-memory fallback.

### Phase 4 — Explainability & Compliance Layer
Every decision produces a causal trace: the winning signal, its runner-up, the confidence margin, which attack vectors were active, session influence, and a plain-English rationale. Six regulation plugins (97 clauses) emit per-decision citations. A 4-section compliance report maps penalty exposure for boards.

### Phase 5 — Frontend Console
Built a React 19 dashboard: chat on the left, a multi-tab intelligence panel on the right — RISK · SESSION · EVIDENCE · AGENTS · MONITOR · SYSTEM · HITL — wiring all **56** backend endpoints.

### Phase 6 — Adversarial Hardening
Systematic adversarial testing across **1,001 samples / 12 categories / 132 attack techniques** (direct, roleplay, academic framing, l33tspeak, Hindi/Hinglish code-switching, split-prompt, system exfiltration, regulatory evasion, and false-positive traps). Iterated thresholds until false positives reached zero.

### Phase 7 — Agentic Governance (Ring 12 + Ring 13 + the agentic plane)
Built the trajectory verifier and the agentic supporting cast:
- **Ring 12 — Trajectory Verifier:** 5 drift signals + penetration-depth recession (the "Chakravyuha formation"), plus declaration-independent hard-block rings (secret→egress flow, provenance/pipeline-poisoning, capability-assembly, privilege-escalation).
- **Ring 13 — Jayadratha Gate:** egress closure — once Ring 12 fires WARN/KILL, the session is frozen and no further tool call can reinforce the behaviour.
- **Sandhi:** Ed25519-signed pre-session declarations (goal + plan + allowed tools/classes) — tamper-detectable, multi-pod.
- **Conch (Pāñchajanya):** cross-tenant IOC broadcast — one tenant's novel kill raises every other tenant's vigilance for 24h.
- **Vyuha Selector:** adaptive pre-pipeline formation router.
- **Sequential Engagement:** rotating detector pools with non-overlapping blindspots, auto-quarantine/promote on HITL outcomes.

### Phase 8 — Ecosystem & Proof
Published the SDKs (`aegis-ai-sdk`, `aegis-ring12` on PyPI; `@aegis.org/sdk`, `@aegis.org/mcp` on npm), the `agentic-redteam-benchmark`, a Zenodo research preprint, and a formal `BOUNDED_EVASION_THEOREM.md`.

---

## How It Is Tested

A custom adversarial evaluator runs the full governance pipeline directly — not over HTTP — to test accuracy without rate-limit interference.

**Internal adversarial set — 1,001 samples · 12 harm categories · 132 attack techniques:**

| Metric | Score |
|--------|-------|
| Accuracy | **99.30%** |
| Precision | **100.00%** |
| Recall | 99.20% |
| F1-Score | 99.60% |
| False Positives | **0 / 130 SAFE** |

**External validation — AdvBench (Zou et al., 2023, zero training overlap):** 99.62% detection recall (518/520). **+34.96pp accuracy over OpenAI's Moderation API** on the same adversarial set.

**Agentic set — `agentic-redteam-benchmark`:** 500+ hand-authored adversarial agent trajectories (513 authored), 5 drift modes, per-step ground truth, scored on four metrics — F1, kill-session latency, false-alarm rate on benign steps, and cost per 1k steps. Public release with full baselines: **9 July 2026**.

The eval datasets, results, and per-sample breakdowns are versioned in `backend/eval/results/`; the agent benchmark and its design doc live in `agentic-redteam-benchmark/`.

---

## How It Is Deployed

```
User / AI Agent (HTTPS)
     │
     ▼
AWS CloudFront  ──────────────────────┐
     │                                │
     ▼                                ▼
AWS S3                      ALB / API Proxy
React/Vite build                      │
                                      ▼
                             AWS EC2 / ECS Fargate (Docker)
                             FastAPI + Gunicorn + Redis
                                      │
              ┌──────────────┬────────┼────────┬──────────────┐
              ▼              ▼        ▼         ▼              ▼
          MongoDB        Groq API  FAISS    PostgreSQL     Control Plane
          Audit vault    (LLM)     Index    Tenant reg.    (Redis: sessions ·
          + hash chain                                      Conch IOC · Sandhi)
```

Frontend is static — S3 + CloudFront. Backend is containerised on EC2/Fargate via Docker Compose with Redis as the control plane (sessions, HITL queue, Conch IOC store, Sandhi key registry — all multi-pod). MongoDB runs on Atlas. The FAISS index and ONNX model are mounted as read-only volumes so model updates don't require a full image rebuild. Live deployment: backend v1.1.0 on EC2 (`http://3.229.242.16:8000`), frontend on CloudFront (`https://d1qmmfoekk6xmp.cloudfront.net/`).

No GPU. No specialised infrastructure. Runs on commodity EC2.

---

## Known Limitations

| Gap | Why It Matters |
|-----|----------------|
| Live demo runs single-tenant in-memory | `REDIS_URL` / `POSTGRES_URL` not wired on the free demo box; multi-pod code is built and tested, but cross-pod propagation isn't exercised against live Redis yet |
| Classifier core is kNN + tuned thresholds | A trained calibration head (prototype at AUC 0.983) is the real ML moat; tuned constants are the interim |
| Agent benchmark headline still finalising | Theorem, engine, and harness all exist; the final powered Ring 12 vs baseline measurement is the next step, not new R&D |
| Federated learning is an in-memory demo | Gradient submission + FedAvg + ε-budget exist; real distributed training is roadmap |
| MaxMind GeoIP DB not bundled | Jurisdiction falls back to tenant-declared |
| No external pen-test / SOC2 yet | Required for enterprise; on the funded path |

These are known, sequenced, and documented. They are what funding closes.

---

## What This Becomes

Aegis AI proves the architecture works. The semantic engine is accurate. The session intelligence is real. The trajectory verifier is novel and theorem-backed. The explainability layer is production-grade. The eval results are honest.

The gaps above are not design failures — they are the scope of a pre-seed proof-of-concept, built by one person, to prove a category worth funding.

The full design rationale — why 13 rings, why FAISS over a vector DB, why a 3-way commit gate, why penetration-depth recession — is documented separately.

**→ [`DESIGN.md`](DESIGN.md) · [`REQUIREMENTS.md`](REQUIREMENTS.md) · [`docs/BOUNDED_EVASION_THEOREM.md`](docs/BOUNDED_EVASION_THEOREM.md)**

---

*Aegis AI — Chakravyuha V3 · Built by Jaswanth Alkur · June 2026*
*Live demo: https://d1qmmfoekk6xmp.cloudfront.net/ · Research: https://doi.org/10.5281/zenodo.19690403 · Writing: https://ajaswanth.substack.com*
