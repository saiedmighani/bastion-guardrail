# Bastion-Guardrail: Roadmap to a SOTA, Publishable Input Guardrail

**Status:** Draft v0.1 — April 2026
**Owner:** @saiedmighani
**Goal:** An end-to-end toolkit that (a) ingests an arbitrary Hugging Face base model and a natural-language **policy template**, (b) synthesizes an adversarial training set via a multi-agent red-team loop against a user-provided LLM endpoint, (c) fine-tunes the base model into a **policy-conditioned input guardrail**, and (d) ships a reproducible deployment artifact. Target: a result strong and novel enough to submit to ICML / NeurIPS / ICLR / COLM.

---

## 1. Research Thesis

Existing open-source input guardrails (LlamaGuard-2/3, ShieldGemma, WildGuard, Aegis-Guard, NeMo-Guardrails classifiers) are trained against a **fixed, author-chosen taxonomy**. Deployers routinely need custom policies (brand safety, regulated-industry rules, tenant-specific ToS) and pay a heavy tax to re-label data and re-train.

We propose **Bastion** — a **policy-conditioned, adversarially-distilled input guardrail** with three contributions aimed at a top venue:

1. **Policy-as-Prompt Distillation (PaPD).** Treat the policy text as an input variable at both training and inference time, so a single checkpoint generalizes to unseen policies without retraining. Evaluate compositional generalization across held-out policy clauses.
2. **Self-Play Red-Team Curriculum (SPaRC).** A multi-agent adversarial data-generation loop (Attacker / Paraphraser / Policy-Lawyer / Judge) with difficulty-adaptive sampling and a verifier-in-the-loop that yields a provably non-collapsing curriculum. Formalize via a minimax objective with KL-regularized attacker and prove a convergence bound under mild assumptions.
3. **Certified-Margin Guardrail (CMG).** Combine fine-tuning with randomized-smoothing / conformal abstention over embeddings to give a per-request robustness certificate against bounded paraphrase perturbations — a first for open guardrails.

Any **one** of these is a plausible top-tier paper; together they form a cohesive system paper plus 1-2 workshop spin-offs.

---

## 2. Landscape & Positioning (as of Apr 2026)

| System | Policy-configurable? | Adversarial training? | Certified? | Open weights? |
|---|---|---|---|---|
| LlamaGuard-2 / 3 | Partially (fixed taxonomy) | Limited | No | Yes |
| ShieldGemma | No | No | No | Yes |
| WildGuard | No | Partial | No | Yes |
| Aegis-Guard | No | No | No | Yes |
| NeMo Guardrails | Rule-based | No | No | Yes |
| OpenAI Moderation | No | Undisclosed | No | No |
| **Bastion (ours)** | **Yes (text policy)** | **Yes (multi-agent curriculum)** | **Yes (conformal + smoothing)** | **Yes** |

**Novelty vector** = policy-conditioning × adversarial curriculum × certification. Reviewers will ask about PolicyLM, RuleBERT, Constitutional-AI, and RAIL; we need a section positioning against each.

---

## 3. System Architecture (target)

```
┌──────────────────────────────────────────────────────────────────┐
│                       Bastion Pipeline                          │
│                                                                 │
│  ┌──────────┐   ┌───────────────┐   ┌────────────────────────┐  │
│  │ HF model │──▶│ Policy Parser │──▶│ Policy Embedding Head  │  │
│  └──────────┘   └───────────────┘   └────────────────────────┘  │
│                                                   │              │
│  ┌──────────────────────────────────────────────┐ │              │
│  │   Self-Play Red-Team Loop (SPaRC)            │ │              │
│  │   ┌──────────┐  ┌───────────┐  ┌──────────┐  │ │              │
│  │   │ Attacker │◀▶│ Paraphrase│◀▶│  Judge   │  │ │              │
│  │   └──────────┘  └───────────┘  └──────────┘  │ │              │
│  │          ▲                           │        │ │              │
│  │          └─── difficulty scheduler ──┘        │ │              │
│  └──────────────────────────────────────────────┘ │              │
│                             │                      │              │
│                             ▼                      ▼              │
│                   ┌────────────────────────────────────┐          │
│                   │  Policy-Conditioned Fine-Tuning    │          │
│                   │  (LoRA / full-rank / DoRA)         │          │
│                   └────────────────────────────────────┘          │
│                             │                                    │
│                             ▼                                    │
│                   ┌────────────────────────────────────┐          │
│                   │   Certification (Smoothing + CP)   │          │
│                   └────────────────────────────────────┘          │
│                             │                                    │
│                             ▼                                    │
│                   ┌────────────────────────────────────┐          │
│                   │   Serving (vLLM / TGI / TorchServe)│          │
│                   └────────────────────────────────────┘          │
└──────────────────────────────────────────────────────────────────┘
```

---

## 4. Milestones & Deliverables

### M0 — Repo & Infra bootstrap (Week 1-2)
- [ ] Package layout: `bastion/{policy,attack,train,certify,serve,eval}`, `configs/`, `scripts/`, `notebooks/`.
- [ ] Reproducibility scaffold: `pyproject.toml`, `uv.lock` or `poetry.lock`, pinned `transformers`, `trl`, `peft`, `vllm`, `datasets`.
- [ ] CI: unit tests + contract tests on a tiny HF model (e.g. `HuggingFaceTB/SmolLM-135M`) end-to-end.
- [ ] Experiment tracking: W&B + deterministic seeds + `artifact://` manifests.
- [ ] Data governance: license provenance, PII scan, content warning headers.

### M1 — Policy Template DSL & Parser (Week 2-4)
- [ ] YAML/Markdown hybrid policy spec: `name`, `scope`, `severity`, `clauses[]`, `examples{allow,deny}`, `references`.
- [ ] Parser → normalized JSON schema + natural-language rendering used as model input.
- [ ] Golden set of 12-20 hand-written policies (safety, brand, regulated, tenant-custom) and 3 synthetic mutations per policy for held-out tests.
- [ ] **Ship:** `bastion policy lint <file>` and `bastion policy render <file>`.

### M2 — Adversarial Data-Gen (SPaRC) v0 (Week 4-8)
- [ ] Pluggable LLM endpoint abstraction (OpenAI-compatible; support Anthropic, vLLM, TGI, OpenRouter).
- [ ] Roles: **Attacker** (generates jailbreaks), **Paraphraser** (GCG-style + LLM paraphrase), **Policy-Lawyer** (rewrites policy to expose ambiguities), **Judge** (LLM-as-judge ensemble + rule checks).
- [ ] Difficulty scheduler: elo-style pairings; promote examples the current student misclassifies with high confidence.
- [ ] Dedup + near-dup removal (MinHash / SemHash) and contamination check against public eval sets.
- [ ] **Ship:** `bastion attack gen --policy X --endpoint Y --budget N` producing a `DatasetDict`.

### M3 — Policy-Conditioned Fine-Tuning (Week 6-10, overlaps M2)
- [ ] Student architectures: encoder (ModernBERT-Large), encoder-decoder (T5-v1.1), decoder-only (Llama-3.2-1B/3B, Qwen3-1.7B, Gemma-3-1B).
- [ ] Heads: binary block/allow, multi-label clauses, span highlighting, abstain.
- [ ] Training recipes: SFT, DPO/KTO on judge-preferred traces, ORPO, and **policy-dropout** regularizer.
- [ ] LoRA / DoRA / full FT comparisons; FLOPs & carbon accounting.
- [ ] **Ship:** reproducible `bastion train` with config-as-code.

#### M3.5 — Ternary-Bonsai candidate track (gated, parallel to M3)

PrismML released **Ternary Bonsai** (Apr 16 2026) — a family of end-to-end 1.58-bit ternary LMs at 8B / 4B / **1.7B**, Apache-2.0, available on Hugging Face (`prism-ml/*`). Every tensor (embeddings, attention, MLP, LM head) is ternary {-s, 0, +s} with a shared FP16 scale per 128-weight group. The 8B is ~1.75 GB; the 1.7B should land in the 300-500 MB range — **CPU-resident, sub-10ms latency territory**. Reported 8B avg: 75.5, beating models 9-10× larger in the same class.

**Why it matters for Bastion.** Adds a third point on the Pareto frontier alongside ModernBERT (encoder) and Qwen3-1.7B (BF16 decoder): `accuracy × latency × memory`. Enables a **"CPU-only / edge sidecar" deployment story** — compelling headline: *"first certified, policy-conditioned guardrail runnable under 500 MB on CPU."*

**Gated — commit compute only after three smoke tests pass:**

1. **Fine-tunability check (2-3 days).** End-to-end ternary weights break the standard BF16 backward-pass assumption in `transformers` / `peft` / `trl`. Confirm a working QAT / STE SFT path — most likely LoRA on the FP16 group scales, or a two-stage recipe (dequant → BF16 LoRA → requantize). If no FT path is viable, this track is cancelled.
2. **Policy-following at 1.7B ternary (1 day).** Prompt with a multi-clause policy + 50 held-out inputs; measure whether the model discriminates at the clause level (not just global toxicity). Threshold: ≥ 0.85 F1 on a ToxicChat subsample with a minimal PaPD prompt. Below that, drop to a footnote.
3. **Certification compatibility (2 days).** Verify randomized-smoothing decision surface is non-degenerate under ternary quantization; fall back to **conformal-only** certification for this variant if certified radii are vacuous. Report both empirical and certified robustness honestly.

**Non-critical path.** Qwen3-1.7B stays the default decoder baseline. Ternary Bonsai is an *additive* experiment — if the gates fail, the main paper is unaffected; if they pass, it likely becomes the serving-tier headline and a strong broader-impact narrative (accessible safety tooling).

- [ ] Gate 1 — ternary-compatible FT recipe validated on SmolLM-scale smoke task.
- [ ] Gate 2 — policy-conditioned smoke eval ≥ 0.85 F1.
- [ ] Gate 3 — certification regime selected (smoothing vs. conformal-only).
- [ ] If all gates pass: add `prism-ml/Ternary-Bonsai-1.7B` to the M7 sweep as an independent size class.
- [ ] **Ship (conditional):** CPU-only reference deployment + `bastion serve --backend llama.cpp` path.

#### M3.6 — Specialist Distillation into a Policy-Conditioned Student

Inspired by Hinton, Vinyals & Dean (2015, §5-6 — "specialist" ensembles trained on confusable subsets of a large label space and distilled into a single model). We adapt the idea to guardrails by treating **risk categories as the confusable subsets**.

**Category taxonomy (v0, revisit after M1).** jailbreak · toxicity/hate · self-harm · sexual/CSAM · PII exfiltration · prompt-injection · IP/copyright · brand-safety · regulatory-compliance (GDPR/HIPAA/PCI). Expect category overlap; run a **co-occurrence study** in M5 and collapse or merge categories before specializing.

**Two tracks, staged:**

- **M3.6a — Multi-head baseline (first cut).** Single shared backbone, one classification head per category + a fusion head. Cheap, clean, and becomes the within-paper baseline for the specialist-distillation variant.
- **M3.6b — Specialist-distillation (SOTA path).** For each category, train a dedicated teacher on a category-specific SPaRC corpus (category-specialized Attacker + Judge). Distill the teacher ensemble into a single policy-conditioned student using soft targets, with two key twists:
  - **Clause-gated distillation:** during training the student only receives soft-target weight from teachers whose category is active in the current policy rendering. This ties PaPD directly to the distillation signal and is the novel contribution.
  - **Per-category temperature:** tune `T_c` per teacher — categories with higher label noise (e.g. brand-safety) get softer targets than hard-rule categories (e.g. PII).

**Certification interaction.** Each specialist gets its own conformal calibration split → the student inherits a **calibrated risk vector** `(q̂_c)_c` rather than a scalar; the §4 CMG bound generalizes via a union bound over categories. Write this up as a sub-contribution in the theory section.

**Paper framing (how to position this, given it's rooted in 2015 work).**
- Cite Hinton'15 as inspiration for the specialist-ensemble scaffold.
- Differentiate against recent baselines: Aegis-Guard's multi-head taxonomy, ShieldGemma-2's category heads, WildGuard's joint head, and task-arithmetic / model-merging safety work.
- The novel claim is **clause-gated distillation with policy conditioning** — not specialists per se, nor distillation per se.

**Deliverables:**
- [ ] Category taxonomy doc (`docs/CATEGORIES.md`) with inclusion/exclusion criteria and overlap notes.
- [ ] Per-category SPaRC profile (attacker prompts, judge rubric, eval slice) — extends M2.
- [ ] M3.6a multi-head baseline trained and benchmarked.
- [ ] M3.6b specialist teachers trained; clause-gated distillation recipe implemented.
- [ ] Ablations (feed into M7): specialists-vs-multi-head, with/without clause gating, per-category temperature sweep, hard-label vs. soft-label student, distillation vs. MoE-at-inference (small-scale).
- [ ] Theory addendum: per-category conformal → calibrated risk vector with union-bound guarantee.

### M4 — Certification Layer (Week 10-13)
- [ ] Randomized smoothing over token-level synonym perturbations (WordNet + MLM-substitution).
- [ ] Conformal risk control on a calibration split; emit per-request `(decision, margin, q̂)`.
- [ ] Theoretical section: bound on certified radius vs. smoothing σ; sample-complexity result.
- [ ] **Ship:** `bastion certify` CLI + FastAPI `/check` endpoint returning the certificate.

### M5 — Evaluation Harness (Week 8-14, continuous)
Public benchmarks (all must be reported):
- **Harmful-prompt / jailbreak:** HarmBench, JailbreakBench, AdvBench, WildGuardTest, DoAnythingNow, StrongREJECT.
- **Toxicity / moderation:** ToxicChat, OpenAI Mod Eval, XSTest (over-refusal), OR-Bench.
- **Prompt injection:** TensorTrust, INJECAGENT, PromptBench.
- **General capability regression of host LLM with Bastion in front:** MT-Bench, AlpacaEval-2, MMLU-Redux.
- **Policy generalization (our new benchmark):** *PolicyBench-Zero* — held-out policies unseen in training; *PolicyBench-Compose* — AND/OR/NOT clause composition.

Metrics: AUROC, F1@FPR=1%, ECE, over-refusal rate, certified-accuracy, robustness-under-attack (ASR↓), latency p50/p99, tokens/s.

### M6 — Serving & SDK (Week 12-16)
- [ ] vLLM / TGI model server with quantized (AWQ/GPTQ/FP8) and BF16 variants.
- [ ] Python SDK: `Bastion.from_pretrained(...).check(prompt, policy=...)`.
- [ ] Sidecar reference deployments: Envoy filter, LangChain/LangGraph middleware, LiteLLM hook, Guardrails-AI validator.
- [ ] Streaming + batched inference path; first-token latency budget <50ms on A10G.

### M7 — Paper Experiments & Ablations (Week 14-20)
- [ ] Full benchmark sweep (≥3 seeds) across 4 base sizes × 3 recipes × 4 attack budgets.
- [ ] Ablations: policy-dropout, judge ensemble size, attacker diversity (n-gram, MAUVE), curriculum vs. uniform sampling, smoothing σ.
- [ ] Specialist-distillation ablations (from M3.6): multi-head vs. specialist-distilled student; clause-gated vs. unconditional distillation; per-category temperature sweep; hard- vs. soft-target; number of specialists (collapsed taxonomies); distillation vs. MoE-at-inference on a single size class.
- [ ] Qualitative: failure-mode taxonomy + human agreement study (Cohen's κ on 500 samples, 3 raters).
- [ ] Compute budget statement + energy / CO2 eq (MLCO2 methodology).

### M8 — Paper Writing & Submission (Week 18-24)
- [ ] Draft → internal reviews ×2 → camera-ready template ready.
- [ ] Reproducibility checklist (NeurIPS/ICML style), model card, data sheet, broader-impact.
- [ ] Anonymized artifact with `docker run` one-shot.

---

## 5. Timeline (Gantt-ish)

```
Week:       1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24
M0 Infra    ████
M1 Policy      █████
M2 SPaRC          ██████████████
M3 Train                █████████████
M4 Certify                           █████████
M5 Eval              (continuous) ░░░░░░░░░░░░░░░░░░░░░░░░
M6 Serving                              ████████
M7 Exp                                             ████████████
M8 Paper                                                     ████████████
```

Target first submission: **NeurIPS 2026** (abstract reg ~mid-May, full paper ~late-May 2026). If we miss it, fall-through to COLM → EMNLP → ICLR.

---

## 6. Target Venues (as of Apr 2026)

### Main-track fits
| Venue | Typical deadline | Why it fits | Notes |
|---|---|---|---|
| **NeurIPS 2026** | ~mid-May 2026 | Datasets & Benchmarks or Main track (safety) | **Primary target**; aggressive but feasible if M5 starts early |
| **COLM 2026** | ~late-Mar 2026 (likely passed) / check 2027 | LM-first venue, friendly to guardrail/safety | Back-up; strong community fit |
| **EMNLP 2026** | ~mid-Jun 2026 | NLP safety, policy-as-prompt framing | Solid fallback after NeurIPS |
| **ICLR 2027** | ~late-Sep/early-Oct 2026 | Methodological novelty of SPaRC & certification | Best for the theory-heavy version |
| **AAAI 2027** | ~Aug 2026 | Trustworthy AI track | Broader audience |
| **ACL 2027** | ~Feb 2027 | NLP + safety | ARR rolling submission pipelines help |
| **NAACL 2027** | ~Oct 2026 | Same as ACL | |
| **ICML 2027** | ~Jan/Feb 2027 | Safety + theory angle | Good home for CMG-heavy version |

### Security / systems venues (strong for certification + red-team angle)
| Venue | Deadline cadence | Fit |
|---|---|---|
| **IEEE S&P 2027** | rolling: Jun/Sep/Dec 2026 | Adversarial ML + certified defenses |
| **USENIX Security 2027** | multi-cycle (Jun/Oct 2026 etc.) | Systems red-team + deployment |
| **ACM CCS 2026** | ~late-Apr/early-May 2026 (tight) | LLM security focus |
| **NDSS 2027** | ~Jul 2026 | Red-team / prompt-injection story |
| **SaTML 2027** | ~Oct 2026 | Safe and Trustworthy ML — perfect home |

### Workshops / spin-offs (lower risk, fast feedback)
- **NeurIPS 2026 SafeGenAI / SoLaR / RegML workshops** (deadlines ~Aug 2026)
- **ICML 2026 NextGenAISafety / AdvML-Frontiers** (if proceedings open retroactively)
- **ICLR 2027 SeT LLM / R2-FM workshops**
- **ACL 2026 TrustNLP**
- **COLM 2026 workshops**

### Journals (fallback / extended)
- **TMLR** (rolling, Certified ML papers welcomed)
- **JMLR** (theory-heavy extension of CMG)
- **ACM TOPS / TISSEC** (security extension)

**Recommended strategy:** aim **NeurIPS 2026 main track** first; in parallel submit a **reduced adversarial-only workshop paper** to a NeurIPS workshop for community feedback; extend the winning camera-ready to **TMLR** or **SaTML 2027**.

---

## 7. Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| LLM endpoint costs blow budget during SPaRC | High | Med | Cache + dedup aggressively; distill into open 8B attacker mid-way; budget-aware scheduler |
| Evaluation contamination (train/test leaks) | Med | High | Hash-based dedup against all public eval sets; hold out an internal `PolicyBench-Zero` we never release pre-submission |
| Over-refusal regression vs. base guardrails | Med | High | Include XSTest/OR-Bench as first-class metric; penalize in training |
| Judge-LM bias in preference data | High | Med | Ensemble of 3 judges (different families) + human spot-check |
| Certification bound too loose to matter | Med | Med | Report *both* certified and empirical robustness; use CP as fallback when smoothing vacuous |
| Base model licensing (Llama/Gemma) | Low | Med | Ensure Apache/MIT track (Qwen3, SmolLM3, OLMo-2) for headline release |
| Reviewers see "yet another guardrail" | Med | High | Lead with **policy-conditioning + certification** as the story; relegate benchmarks to tables |
| Dual-use / safety-washing concerns | Low | High | Write thoughtful broader-impact; red-team artifacts gated; responsible-release plan |
| Ternary-Bonsai track fails its gates (new, unvetted pretrain; ternary FT tooling immature) | Med | Low | Parallel/optional track — Qwen3-1.7B remains default; drop to footnote if any M3.5 gate fails |
| Category taxonomy too leaky for specialist distillation (overlap between jailbreak / self-harm / injection) | Med | Med | Co-occurrence study before M3.6b; collapse categories when mutual information exceeds threshold; fall back to M3.6a multi-head as paper baseline |
| Specialists add training complexity with marginal gain over multi-head | Med | Med | Pre-register multi-head as baseline; commit to specialist-distillation only if M3.6b beats M3.6a by a pre-agreed margin on PolicyBench-Compose |

---

## 8. Reproducibility & Open-Science Commitments

- Apache-2.0 code; CC-BY-4.0 data where legally possible.
- Full W&B project (public) with every run.
- Model card (Hugging Face) + data sheet (Gebru et al.) + impact statement.
- `docker run ghcr.io/.../bastion:paper` reproduces all main-paper tables.
- Pre-register `PolicyBench-Zero` on OpenReview/archive before eval.

---

## 9. Team & Compute (to size)

- **Research lead:** policy-conditioning + paper.
- **Infra eng:** SPaRC loop + serving.
- **Eval eng:** benchmark harness + PolicyBench.
- **Theorist (advisor or collaborator):** certification bounds.
- **Compute:** ~8×H100 for 4-8 weeks for the sweep; ~1×A10G for smoke tests. Estimate: 3-5k GPU-hours. Plan for Lambda / Together / Crusoe backup.

---

## 10. Immediate Next Actions (this week)

1. Land this roadmap on `claude/guardrail-roadmap-plan-90kPu` and open a discussion issue.
2. Scaffold `bastion/` package + CI (M0).
3. Draft the policy DSL schema (`docs/POLICY_SPEC.md`) and get a second opinion from one safety practitioner.
4. Stand up the smallest end-to-end loop: SmolLM-135M + one policy + 200 synthetic adversarial samples + eval on ToxicChat → confirms plumbing before scaling.
5. Begin related-work deep-dive (target: 60 papers tracked in Zotero) and identify the 5 closest baselines to reimplement.

---

## 11. Open Questions (for discussion)

- Do we target **one large paper** or a **two-paper arc** (SPaRC @ NeurIPS, CMG @ ICML/S&P)?
- Is policy-conditioning best done via prompt, prefix-tuning, adapter, or a small policy-encoder tower?
- Should the released model be **decoder-only** (easier to adopt) or **encoder** (lower latency)?
- How do we handle multilingual policies at v1? (recommend: English-only v1, multilingual as v1.1 workshop paper.)
- Governance: who approves release of the red-team corpus?

---

*This roadmap is a living document — revise after each milestone review.*
