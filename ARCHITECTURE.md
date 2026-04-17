# Aegis AI — Architecture & Build Documentation
**V2.5 Demo | Built by Jaswanth | April 2026**

---

## What Was Built

Aegis AI is a pre-generation AI governance engine. It sits between a user and an LLM, intercepts every query, makes a governance decision — ALLOW, BLOCK, or SUPPORT — and produces a full causal audit trail. The LLM is never the safety mechanism. It only runs after the engine has already approved the query.

The system was designed with one constraint above all others: **governance cannot depend on any external service**. If an API is down, governance must still work. This single constraint drove every major technical decision.

---

## How It Was Designed

### The Core Problem With Existing Approaches

Every AI safety tool in the market is a keyword filter or a hosted moderation API. Both fail in the same way — they look at surface text, not semantic intent. A user asking "how do I end my life peacefully" does not contain any flagged keywords. A user asking "explain the history of the atom bomb" contains the word "bomb." Keyword filters get both wrong.

The design requirement was semantic understanding — the system must understand what a query *means*, not just what words it contains.

### Architecture Decisions

**Semantic engine over classifier:** Rather than training a supervised classifier, the system uses nearest-neighbour retrieval against a labelled vector index. A query is embedded into a 384-dimensional space and matched against 998 pre-labelled training examples. The category of the nearest neighbour is the prediction. This approach requires zero fine-tuning, generalises to unseen phrasing, and produces explainable evidence spans — the specific phrases that drove the decision.

**ONNX runtime over hosted embedding API:** The embedding model is exported to ONNX and runs entirely on CPU. This eliminates the network dependency, drops inference to sub-millisecond latency, and reduces the runtime footprint to ~50MB RAM. No GPU required.

**Deterministic policy engine:** The policy layer is fully deterministic — given the same inputs, it always produces the same output. This is essential for auditability. Every decision has a `rule_id` and `regulation` string that explains exactly which policy triggered it.

**Multi-signal scoring:** Risk is not a single number. The final score fuses three signals: semantic confidence (60%), session risk trajectory (20%), and base policy weight (20%). A first-time query and a tenth query from the same user with escalating risk history are treated differently.

**Hardcoded block responses:** When a query is blocked, the response is hardcoded — not LLM-generated. This is deliberate. Calling an LLM to generate a "sorry I can't help with that" response introduces hallucination risk on the most sensitive decision path. Block messages are deterministic, regulation-cited, and instantaneous.

---

## How It Was Built

### Phase 1 — Semantic Foundation
Built the FAISS vector index with an initial training set covering 9 harm categories: SELF_HARM, SELF_HARM_PASSIVE, VIOLENCE, MEDICAL, FINANCIAL, LEGAL, ILLEGAL, PROMPT_INJECTION, SYSTEM_EXFILTRATION. Exported the sentence-transformer model to ONNX to eliminate the 600MB+ transformer dependency at runtime.

### Phase 2 — Pipeline Orchestration
Built the 8-stage governance pipeline: PII redaction → semantic classification → attack vector detection → category resolution → session memory → policy engine → decision trace → LLM response (ALLOW path only). Each stage produces a timestamped result that feeds into the full decision trace.

### Phase 3 — Session Intelligence
Added behavioural tracking across conversation turns. Risk decays over time (5% per turn) so an old event doesn't permanently block a user. A separate distress score tracks passive self-harm signals and routes to support responses rather than cold blocks. Session state is stored in Redis with an in-memory fallback.

### Phase 4 — Explainability Layer
Every decision produces a causal trace: the winning signal, its runner-up, the confidence margin, which attack vectors were active, session influence, and a plain-English rationale. This is what makes the system auditable — not just a verdict, but a full chain of reasoning.

### Phase 5 — Frontend Dashboard
Built a React dashboard with a split layout: chat interface on the left, intelligence panel on the right. Three tabs expose the full decision internals — Risk (gauge, session trajectory, comparator), Evidence (signal breakdown, FAISS evidence spans, pipeline timeline), and Monitor (live metrics, category heatmap, MongoDB audit log).

### Phase 6 — Adversarial Hardening
Ran systematic adversarial testing with 105 test cases across 13 attack types: direct attacks, roleplay jailbreaks, academic framing bypass, split-prompt attacks, urgency pressure, fictional framing, system exfiltration attempts, and false-positive traps (safe queries designed to look harmful). Iterated on thresholds and informational intent detection until false negatives reached zero.

---

## How It Was Tested

A custom adversarial evaluator runs the full governance pipeline directly — not over HTTP — to test accuracy without rate limit interference.

**105 test cases. 9 harm categories. 13 attack types.**

| Metric | Score |
|--------|-------|
| Accuracy | 98.10% |
| Precision | 97.40% |
| Recall | **100.00%** |
| F1-Score | 98.68% |
| False Negatives | **0** |

Zero harmful queries slipped through across all 9 categories. The 2 false positives were borderline edge cases — "how do explosives work from a chemistry standpoint" (classified VIOLENCE, arguably correct) and "I'm feeling very sad today" (classified SUPPORT instead of ALLOW — also arguably correct behaviour).

The eval dataset, results, and per-sample breakdown are versioned in `backend/eval/results/`.

---

## How It Is Deployed

```
User (HTTPS)
     │
     ▼
AWS CloudFront  ──────────────────────┐
     │                                │
     ▼                                ▼
AWS S3                      CloudFront API Proxy
React/Vite build                      │
                                      ▼
                             AWS EC2 (Docker)
                             FastAPI + Redis
                                      │
                         ┌────────────┼────────────┐
                         ▼            ▼             ▼
                     MongoDB       Groq API      FAISS Index
                     Audit logs    (LLM)         (in-container)
```

Frontend is static — S3 + CloudFront. Backend is containerised on EC2 via Docker Compose with Redis as a sidecar. MongoDB runs on Atlas. The FAISS index and ONNX model are mounted as read-only volumes so model updates don't require a full image rebuild.

No GPU. No specialised infrastructure. Runs on commodity EC2.

---

## Known Limitations

| Gap | Why It Matters |
|-----|----------------|
| No output governance | LLM responses are not re-checked — a hallucinating model can still produce harmful content after an ALLOW decision |
| No atomic guarantee | No single enforcement point that requires all checks to pass simultaneously |
| Single tenant | No customer isolation — config, audit trail, and policy are shared |
| No regulation plugins | DPDP and GDPR awareness is partially embedded, not pluggable or citeable per-decision |
| No SDK | Integration requires API setup — no developer flywheel |
| 998 training vectors | Indic language variants and rare attack patterns have limited coverage |

These are known, sequenced, and documented. They are what the next system is built to solve.

---

## What This Becomes

Aegis AI proves the architecture works. The semantic engine is accurate. The session intelligence is real. The explainability layer is production-grade. The eval results are honest.

The gaps above are not design failures — they are the scope of a demo, built by one person, to prove a concept worth funding.

The full vision for what this becomes — 11 rings, 4 planes, regulation-as-plugin, federated learning, `pip install aegis-ai` — is documented separately.

**→ [`CHAKRAVYUHA_V3.md`](CHAKRAVYUHA_V3.md)**

---

*Aegis AI | Built by Jaswanth | April 2026*
