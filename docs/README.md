# docs/ — Chakravyuha Documentation Index

This folder is the canonical reference for **Chakravyuha** — production AI governance infrastructure.

---

## Core Documents

| Document | What it covers |
|---|---|
| [`ARCHITECTURE.md`](ARCHITECTURE.md) | The 4 planes, 11-ring pipeline, semantic engine internals, atomic commit gate, audit vault, multi-tenancy, AWS deployment, security layers, observability, failure modes |
| [`DESIGN.md`](DESIGN.md) | Design philosophy, problem framing, every core decision and rationale, semantic engine design, regulation-as-plugin, atomic 3-way commit, audit vault, infrastructure trade-offs |
| [`REQUIREMENTS.md`](REQUIREMENTS.md) | 70+ functional, 25+ non-functional, regulatory, multi-tenant, SDK, API, frontend, infra, security, audit & compliance, data, and 12 acceptance scenarios |
| [`SYSTEM_ARCHITECTURE.md`](SYSTEM_ARCHITECTURE.md) | Concise system overview (legacy reference) |

---

## Visual Assets

| File | Purpose |
|---|---|
| `chakravyuha .gif` | Hero demo loop — full pipeline in action |
| `pipeline.gif`     | 11-ring pipeline animation |
| `agies _AI.png`    | Logo (used in README hero) |
| `PII-BLOCK.jpeg`   | Live BLOCK with regulation citation |
| `Safe-query.jpeg`  | Live ALLOW with full evidence panel |
| `Data-catalog.png` | Risk Dashboard — Signal Breakdown · Session Decay |
| `compailence-report.png` | Board-ready compliance report — penalty exposure mapped |

> Capture new screenshots from the live console at [https://dnrvkokqdpg2n.cloudfront.net](https://dnrvkokqdpg2n.cloudfront.net).
> For GIF recordings: [LICEcap](https://www.cockos.com/licecap/) (Windows) · [Kap](https://getkap.co/) (Mac).

---

## Where the Numbers Come From

All accuracy metrics in the root `README.md` and these documents derive from:

- [`backend/eval/results/FINAL_EVAL_REPORT_chakravyuha_v3.md`](../backend/eval/results/FINAL_EVAL_REPORT_chakravyuha_v3.md) — primary internal benchmark, complete pipeline analysis (1,001 samples · 97 attacks · 12 categories)
- [`backend/eval/results/advbench_report_20260421_184925.md`](../backend/eval/results/advbench_report_20260421_184925.md) — AdvBench external validation (Zou et al., 2023; 520 samples)
- [`backend/eval/results/comparison_report_20260421_171041.md`](../backend/eval/results/comparison_report_20260421_171041.md) — coverage comparison vs general content moderation
- [`backend/eval/results/arxiv_eval_report_chakravyuha_v3.md`](../backend/eval/results/arxiv_eval_report_chakravyuha_v3.md) — extended technical report

Headline: **99.30% accuracy · 100% precision · 99.20% recall · 0 false positives** on 1,001-sample adversarial benchmark.

---

*Back to [project root README](../README.md)*
