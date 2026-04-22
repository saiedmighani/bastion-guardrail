# Bastion-Guardrail: System Architecture (v0)

**Status:** Draft v0 — April 2026
**Owner:** @saiedmighani
**Related:** [`ROADMAP.md`](./ROADMAP.md) · [`CATEGORIES.md`](./CATEGORIES.md) · [`POLICY_SPEC.md`](./POLICY_SPEC.md)
**Purpose:** lock the package layout, module boundaries, core interfaces, and data flow so that code can land without retro-rewrites. Opinionated — change via PR, not improvisation.

---

## 1. Principles

1. **Config-as-code, code-as-library.** Every experiment is a `configs/*.yaml` pointing at library functions; the CLI is a thin Hydra-style wrapper. Nothing hard-coded in scripts.
2. **One unit of artifact.** A Bastion **release** is `(checkpoint, policy, calibration, eval_report)` bundled by content hash. Serving never accepts fewer than all four.
3. **Narrow module boundaries.** `policy` knows nothing about models. `train` knows nothing about servers. `certify` does not import `attack`. Violations are caught in CI by an import-graph test.
4. **Pure I/O at the edges.** Model calls, HTTP, disk I/O live in thin adapter modules; core logic is pure Python and unit-testable without GPUs or network.
5. **Reproducible or it didn't happen.** Seeds, data hashes, config hashes, and env captures are emitted as part of every run's manifest.
6. **Cheap smoke path.** Every subsystem has a SmolLM-135M / CPU smoke variant runnable on a laptop in <5 minutes. CI uses it.

---

## 2. Package Layout

```
bastion-guardrail/
├── pyproject.toml              # uv-managed, pinned
├── uv.lock
├── README.md
├── LICENSE                     # Apache-2.0
├── docs/
│   ├── ROADMAP.md
│   ├── CATEGORIES.md
│   ├── POLICY_SPEC.md
│   ├── ARCHITECTURE.md         # this file
│   └── SPARC.md                # red-team loop spec
├── src/bastion/
│   ├── __init__.py
│   ├── cli.py                  # Typer entrypoints: policy, attack, train, certify, serve, eval
│   │
│   ├── policy/                 # M1 — POLICY_SPEC.md is the contract
│   │   ├── schema.py           # Pydantic models for normalized JSON
│   │   ├── parser.py           # YAML + Markdown surfaces → normalized
│   │   ├── renderer.py         # prose / structured / minimal render modes + dropout
│   │   ├── linter.py           # validation rules from POLICY_SPEC §10
│   │   ├── compose.py          # boolean expression parser & evaluator
│   │   ├── diff.py             # semantic diff between policies
│   │   ├── categories.py       # registry derived from CATEGORIES.md
│   │   └── hash.py             # canonical hashing
│   │
│   ├── attack/                 # M2 — SPARC.md is the contract
│   │   ├── endpoints/          # pluggable LLM clients (OpenAI, Anthropic, vLLM, TGI, OpenRouter)
│   │   ├── roles/              # attacker.py, paraphraser.py, lawyer.py, judge.py
│   │   ├── scheduler.py        # elo + difficulty curriculum
│   │   ├── dedup.py            # MinHash + contamination checks
│   │   ├── budget.py           # token / $ tracking with hard caps
│   │   ├── corpus.py           # DatasetDict assembly + sharding
│   │   └── replay.py           # hard-example buffer
│   │
│   ├── data/
│   │   ├── schema.py           # record schema for SPaRC corpus
│   │   ├── loaders.py          # HF datasets integration
│   │   ├── splits.py           # train/val/cal/test with policy-holdout logic
│   │   └── contamination.py    # cross-check vs. public eval sets
│   │
│   ├── train/                  # M3 / M3.6
│   │   ├── student.py          # common student interface
│   │   ├── backbones/          # encoder.py (ModernBERT), decoder.py (Qwen/Llama/Gemma), ternary.py (Bonsai)
│   │   ├── heads.py            # binary / multi-label / span / abstain heads
│   │   ├── recipes/            # sft.py, dpo.py, kto.py, orpo.py, distill.py
│   │   ├── distill.py          # M3.6b specialist distillation with clause-gated soft targets
│   │   ├── policy_dropout.py   # PaPD regularizer
│   │   ├── callbacks.py        # W&B, checkpointing, EMA
│   │   └── trainer.py          # thin wrapper over `transformers.Trainer` / `trl`
│   │
│   ├── certify/                # M4
│   │   ├── smoothing.py        # randomized smoothing over synonym perturbations
│   │   ├── conformal.py        # CP over calibration split, per-category q̂
│   │   ├── bounds.py           # theoretical radius / sample-complexity utilities
│   │   └── risk_vector.py      # per-category calibrated risk aggregator
│   │
│   ├── serve/                  # M6
│   │   ├── server.py           # FastAPI /check, /batch, /health, /metrics
│   │   ├── backends/           # vllm.py, tgi.py, transformers.py, llamacpp.py
│   │   ├── sidecars/           # envoy.py, langchain.py, litellm.py, guardrails_ai.py
│   │   └── sdk.py              # Python SDK: Bastion.from_pretrained().check(...)
│   │
│   ├── eval/                   # M5
│   │   ├── harness.py          # unified runner over public benchmarks
│   │   ├── benchmarks/         # harmbench.py, jailbreakbench.py, wildguard.py, toxicchat.py, xstest.py, orbench.py, injecagent.py, tensortrust.py, policybench.py
│   │   ├── metrics.py          # AUROC, F1@FPR, ECE, over-refusal, ASR, certified-accuracy
│   │   ├── latency.py          # p50/p99 on reference hardware
│   │   └── report.py           # emits a paper-table-ready JSON + Markdown
│   │
│   ├── release/
│   │   ├── manifest.py         # (checkpoint, policy, calibration, eval_report) bundle spec
│   │   └── hubpush.py          # HF upload with model card + data sheet
│   │
│   └── util/
│       ├── seed.py
│       ├── env.py              # captures git sha, pip freeze, CUDA/HW info
│       ├── logging.py
│       └── hashing.py
│
├── configs/
│   ├── base.yaml               # defaults shared across experiments
│   ├── student/                # per-backbone configs
│   ├── attack/                 # per-category SPaRC configs
│   ├── train/                  # recipe configs
│   ├── certify/
│   ├── eval/
│   └── experiments/            # composed configs for named experiments
│
├── scripts/                    # not library code — one-off ops scripts
│   ├── bootstrap_env.sh
│   ├── smoke_e2e.py
│   └── release_paper_artifact.sh
│
├── tests/
│   ├── unit/                   # pure, fast, no GPU or network
│   ├── contract/               # interface contracts (e.g. parser round-trip)
│   ├── integration/            # end-to-end on SmolLM-135M, CPU-only
│   ├── golden/                 # deterministic snapshot tests on policy renders, eval metrics
│   └── fixtures/
│       ├── policies/
│       └── prompts/
│
├── notebooks/                  # exploration only; never imported
├── docker/
│   ├── Dockerfile              # runtime (inference)
│   ├── Dockerfile.train        # training (CUDA + distributed)
│   └── Dockerfile.paper        # reproduces main-paper tables
└── .github/workflows/
    ├── ci.yaml                 # lint + unit + contract
    ├── integration.yaml        # integration on self-hosted runner
    └── release.yaml
```

---

## 3. Module Boundaries (dependency rules)

Enforced by a CI import-graph test (`tests/unit/test_imports.py`):

```
policy       → util
attack       → policy, data, util
data         → policy, util
train        → policy, data, util
certify      → data, train (interface only), util
serve        → certify, train (interface only), util
eval         → policy, data, util   # NEVER depends on train/attack internals
release      → train, certify, eval, util
cli          → everything
```

- `eval` must not import from `train` or `attack` — it must be runnable against any checkpoint by anyone, including reviewers who don't have our training infrastructure.
- `certify` depends on `train` only through the exported student interface, not internals.
- `attack` must not depend on `train` — the red-team loop must be runnable without a local training rig (it queries a separate LLM endpoint).

---

## 4. Core Interfaces

All core types live in `schema.py` files per module. Pydantic v2 for data; `typing.Protocol` for behavior.

### 4.1 Policy

```python
# src/bastion/policy/schema.py
class Policy(pydantic.BaseModel):
    api_version: Literal["bastion/v1"]
    metadata: PolicyMetadata
    scope: Scope
    defaults: Defaults
    clauses: list[Clause]
    composition: Composition
    authorization_contexts: list[AuthContext] = []
    calibration: CalibrationTargets
    hash: str                                       # sha256 of canonical JSON

class Clause(pydantic.BaseModel):
    id: str
    category: CategoryId                            # validated vs. registry
    severity: Severity
    rule: dict[Locale, str]                         # localized rule text
    examples: Examples
    references: list[str] = []
```

### 4.2 Student

```python
# src/bastion/train/student.py
class Student(Protocol):
    def score(self, prompt: str, policy: RenderedPolicy) -> CategoryScores: ...
    def decide(self, scores: CategoryScores, policy: Policy) -> Decision: ...
    @classmethod
    def from_pretrained(cls, path: str | Path) -> "Student": ...
    def save(self, path: str | Path) -> None: ...

class CategoryScores(pydantic.BaseModel):
    per_category: dict[CategoryId, float]           # P(violation | category)
    embedding: torch.Tensor | None                  # for smoothing; optional
```

### 4.3 Specialist + Distillation (M3.6)

```python
# src/bastion/train/distill.py
class SpecialistTeacher(Protocol):
    category: CategoryId
    temperature: float                              # per-category T_c
    def logits(self, prompt: str) -> torch.Tensor: ...

def clause_gated_distill(
    teachers: list[SpecialistTeacher],
    student: Student,
    batch: Batch,
    policy: Policy,                                 # used to mask inactive teachers
    *,
    hard_label_weight: float = 0.5,
) -> Loss: ...
```

The `policy`-argument is what makes distillation clause-gated: only teachers whose category is active in the rendered policy contribute to the soft-target loss for that example.

### 4.4 Attack (SPARC)

```python
# src/bastion/attack/roles/*.py
class Attacker(Protocol):
    category: CategoryId
    def propose(self, policy: Policy, seed: AttackSeed, budget: Budget) -> list[AttackCandidate]: ...

class Judge(Protocol):
    def verdict(self, prompt: str, policy: Policy) -> Verdict: ...
    # Judge is an ensemble internally; `verdict.source` reports per-judge votes.
```

### 4.5 Certifier

```python
# src/bastion/certify/conformal.py
class RiskVector(pydantic.BaseModel):
    per_category: dict[CategoryId, float]           # calibrated q̂_c
    decision: Decision
    margin: float
    alpha: float

class Certifier(Protocol):
    def calibrate(self, student: Student, cal_set: Dataset, policy: Policy) -> CalibrationArtifact: ...
    def certify(self, prompt: str, policy: Policy, student: Student) -> RiskVector: ...
```

### 4.6 Evaluation

```python
# src/bastion/eval/harness.py
class Benchmark(Protocol):
    name: str
    slices: list[str]
    def examples(self) -> Iterator[EvalExample]: ...
    def score(self, predictions: list[Prediction]) -> MetricReport: ...
```

New benchmarks register via a `@register_benchmark` decorator; `eval.report.build_paper_tables` iterates the registry.

---

## 5. End-to-End Data Flow

```
┌───────────────┐
│ Policy (YAML) │
└──────┬────────┘
       │  policy.parser + linter
       ▼
┌───────────────────────┐
│ Normalized Policy JSON│ ─── hash ──→ release.manifest
└──────┬────────────────┘
       │  policy.renderer (prose|structured|minimal + dropout)
       ▼
┌─────────────────────┐         ┌──────────────────────────────┐
│ Rendered policy str │ ←────── │ attack (SPARC) per-category  │──→ DatasetDict
└──────┬──────────────┘         │ (roles + scheduler + dedup)  │
       │                        └──────────────────────────────┘
       │                                         │
       │                                         ▼
       │                                ┌────────────────────┐
       │                                │ data.splits        │
       │                                │ (train/val/cal/tst)│
       │                                └────────────────────┘
       │                                         │
       ▼                                         ▼
┌───────────────────────────────────────────────────────────┐
│ train                                                     │
│   M3 SFT/DPO + policy_dropout                             │
│   M3.6b specialists → clause-gated distillation → student │
└──────────────────────┬────────────────────────────────────┘
                       │
                       ▼
              ┌────────────────────┐
              │ certify.calibrate  │ (per-category conformal on cal split)
              └────────┬───────────┘
                       │
                       ▼
              ┌────────────────────────────────────┐
              │ release.manifest bundle            │
              │ (checkpoint + policy + cal + eval) │
              └────────┬───────────────────────────┘
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
   eval.harness    serve.server    hubpush (HF)
```

Every arrow is a pure function with a canonical hash input/output. `release.manifest` verifies the chain at load time.

---

## 6. Configuration

- **Hydra / OmegaConf** for config composition; `configs/base.yaml` + per-subsystem overrides. Example: `bastion train student=qwen3-1p7b recipe=distill attack=full_curriculum`.
- No magic string lookups — every config value resolves to a typed Pydantic model before use.
- Experiment configs in `configs/experiments/` are the only files allowed to pin everything end-to-end; these are what reviewers reproduce from.
- A config hash goes into every artifact. If you change a config value you change the hash; no silent drift.

---

## 7. Release Artifact

A Bastion release is a directory:

```
release-<hash>/
├── manifest.json           # schema + content hashes
├── policy.json             # normalized policy
├── checkpoint/             # HF-format weights
├── calibration/            # per-category q̂, conformal artifacts
├── eval_report.json        # metrics on pinned benchmarks
├── eval_report.md          # human-readable
├── model_card.md
├── data_sheet.md
└── env.json                # pip freeze, git sha, hw info
```

Serving refuses to load a release missing any of these. This is the reproducibility spine; everything else is convenience.

---

## 8. Testing Strategy

- **Unit (`tests/unit/`)** — pure, no GPU, no network, <1s each. Parser round-trip, compose evaluator, hash determinism, import graph.
- **Contract (`tests/contract/`)** — interface compliance. E.g. every `Student` implementation passes the same 30-test suite; every `Benchmark` satisfies the protocol.
- **Integration (`tests/integration/`)** — full pipeline on SmolLM-135M, CPU-only, <5 min. Runs on every PR via self-hosted runner.
- **Golden (`tests/golden/`)** — snapshot policy renders, eval metric outputs on frozen inputs. Breaks loudly when semantics drift; requires explicit update.
- **Property-based** — Hypothesis for the composition grammar parser and the linter; fuzz policies against round-trip identity.
- **Determinism test** — same seed + same config + same data hash → bitwise-identical checkpoint (CPU path). GPU variant uses tolerance.

Coverage is not a target; the above suites are. No coverage number will save a broken contract test.

---

## 9. Reproducibility

- `util.seed.seed_everything(seed, deterministic=True)` called once in every entrypoint.
- `util.env.capture()` writes `env.json` at run start.
- W&B is the truth for experiment tracking; the W&B run id is referenced in the release manifest.
- A `docker run ghcr.io/saiedmighani/bastion:paper-v1` command reproduces every main-paper table from raw seeds.

---

## 10. Deployment Targets

Priority order:

1. **vLLM / TGI** — GPU decoder student (primary serving path).
2. **`transformers` native** — encoder student (ModernBERT).
3. **`llama.cpp`** — CPU / edge path (Ternary Bonsai 1.7B if M3.5 gates pass).
4. **TorchServe / SageMaker / Modal** — reference templates in `serve/sidecars/`.

All backends expose the same `Bastion.check(prompt, policy)` SDK surface; swap is one config line.

---

## 11. Extension Points

- **New category:** PR to `CATEGORIES.md` → regen `policy/categories.py` registry → add SPaRC profile under `configs/attack/<cat>.yaml` → add eval slice → ship.
- **New student backbone:** subclass `train/backbones/base.py`, pass the 30-test contract suite, register in `configs/student/`.
- **New attacker strategy:** subclass `attack/roles/attacker.py`, register via entry points (`bastion.attackers` group).
- **New benchmark:** implement `Benchmark` protocol, `@register_benchmark`, add `configs/eval/<name>.yaml`.

---

## 12. Dependencies (target)

Pinned in `pyproject.toml`:

- **Core:** `python>=3.11`, `pydantic>=2.7`, `typer`, `omegaconf`, `rich`.
- **ML:** `torch`, `transformers`, `datasets`, `accelerate`, `peft`, `trl`.
- **Serving:** `vllm`, `fastapi`, `uvicorn`.
- **Eval / data:** `numpy`, `pandas`, `scikit-learn`, `datasketch` (MinHash), `crepes` (conformal).
- **Dev:** `pytest`, `hypothesis`, `ruff`, `mypy`, `pre-commit`.

No `requirements.txt`. `uv` only. `uv.lock` committed.

---

## 13. Anti-Goals

Things we explicitly are **not** building, to stay focused:

- A **general** policy engine. Bastion is for LLM-input guardrails; OPA/Cedar handle other policy domains better.
- A **rule-based** classifier framework. If rules alone suffice, use Microsoft Presidio for PII or a keyword filter.
- An **output moderation** system. Bastion is input-side; output moderation is a separate project with a different threat model and will not be shoehorned in.
- A **training platform**. We assume a cluster exists; we don't manage GPUs, schedulers, or S3.
- An **observability product**. We emit OpenMetrics and logs; Grafana / Datadog / whatever consumes them.

---

## 14. Open Questions

1. **Hydra vs. plain OmegaConf + Typer?** Leaning Hydra for composition, but it's heavier. Revisit after first experiment config.
2. **Pydantic v2 everywhere** or **attrs** for performance-critical paths? Default Pydantic; profile before switching.
3. **Package under `bastion/` or `bastion_guardrail/`?** Shorter wins; `bastion` on PyPI is taken — reserve `bastion-guardrail` as the PyPI name, import as `bastion`.
4. **Single repo vs. monorepo split** (e.g. `bastion-core` / `bastion-attack` / `bastion-serve`)? Single repo until a concrete coupling pain appears.
5. **Async throughout `serve/` or sync with threadpool?** Async; FastAPI + vLLM's async engine line up.

---

*This architecture is a living document — revise before writing code for any module that doesn't yet exist.*
