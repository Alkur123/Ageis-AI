# Changelog — The Founder's Build Journal

> This is not just a version history. It is the story of how a B.Tech student figured out that the gap between AI capability and AI accountability was the largest infrastructure opportunity in India's technology history.

---

## The Origin

**2024** — Started with a simple observation: every large language model can be made to produce harmful content. The safety mechanisms companies claimed to have were keyword filters — brittle, bypassable, and entirely unfit for regulated industries.

A keyword filter does not understand context. It does not know whether a user asking about medication dosages is a nurse or someone at risk. It does not track behavioral signals across a conversation. It generates no audit trail. It cites no regulation.

A keyword filter is theater.

The question became: what would real governance infrastructure look like?

---

## V1 — Proof That the Problem Is Real

**Late 2024**

Built a basic rule-based classifier to demonstrate the failure modes of keyword filtering.

- `MEDICAL_ADVICE` → keyword list, blocked anything with drug names
- `VIOLENCE` → keyword list, blocked war documentaries
- False positive rate: catastrophic
- False negative rate: any creative rephrasing bypassed it

**Result:** Confirmed that the industry's standard approach was fundamentally broken. The gap was not a configuration problem. It was an architecture problem.

---

## V2 — Semantic Understanding

**Early 2025**

Replaced the keyword approach entirely. Started with what the problem actually required: semantic understanding of user *intent*, not surface pattern matching.

**What changed:**
- FAISS vector index for semantic similarity search
- sentence-transformers embeddings (384-dimensional)
- ONNX export for sub-millisecond CPU inference
- Multi-signal scoring: `semantic + session + policy`
- Session risk tracking with temporal decay
- PII redaction (Aadhaar, PAN, phone, email, IFSC)
- Causal decision trace — every block explains itself
- React dashboard with dark theme, audit logs, risk graphs

**Published findings at FoCS 2025.**

**Key realization:** Governing input is necessary but not sufficient. An LLM can hallucinate harmful content in its *response* even when the input was clean. The entire industry was governing the wrong side.

---

## V2.5 — Adversarial Hardening (Current Demo)

**April 2026**

Systematic adversarial testing revealed the attack surface. Built a 105-sample adversarial dataset covering:

- Direct attacks across 8 harm categories
- Euphemism and obfuscation variants
- Roleplay and fictional framing
- Academic framing bypass attempts
- Split-prompt compound attacks
- Prompt injection and jailbreak attempts
- System exfiltration attempts
- False positive traps (safe queries designed to look harmful)

**Evaluated. Iterated. Reached:**

| Metric | Score |
|--------|-------|
| Accuracy | 98.10% |
| Recall (no harmful query missed) | **100%** |
| F1-Score | 98.68% |
| False Negatives | **0** |

Zero harmful queries slipped through.

**What V2.5 proved:** The semantic architecture works. The session intelligence works. The explainability works. The demo is not a mockup — it is a working governance engine.

**What V2.5 does not have yet:** Output verification (Ring 7), the atomic commit gate (Ring 8), multi-tenancy, and the Python SDK. These are what funding builds.

---

## V3 — What Gets Built With Funding

**Target: H2 2026**

The 36-page V3 architecture blueprint documents the complete system. The summary:

- **Ring 7:** Post-generation verification — govern the LLM's *output*, not just the input
- **Ring 8:** Atomic 3-way commit gate — the mathematical guarantee
- **Multi-tenancy:** Isolated governance per customer, per jurisdiction
- **Regulation plugins:** DPDP 2023, GDPR, EU AI Act, HIPAA, CCPA, SEBI as loadable modules
- **Python SDK:** `pip install aegis-ai` — working governance in one decorator
- **Federated learning:** The network effect — improves accuracy across all tenants without sharing any raw data
- **On-premise:** Air-gapped deployment for banking, government, and defence
- **Compliance reports:** Board-ready PDFs with penalty exposure calculations

---

## Why Now

The EU AI Act enforcement deadline is August 2026.  
DPDP enforcement is active now.  
No Indian company has built the technical depth to be the compliance layer for both.

The window for being first is measured in months, not years.

---

## The One Line That Has Not Changed

> *"Control AI Before It Controls Outcomes."*

---

*Jaswanth | Aegis AI | April 2026*
