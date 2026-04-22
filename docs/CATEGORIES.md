# Bastion-Guardrail: Risk Category Taxonomy (v0)

**Status:** Draft v0 — April 2026
**Owner:** @saiedmighani
**Related:** [`ROADMAP.md`](./ROADMAP.md) §M3.6 (Specialist Distillation)
**Purpose:** define the risk-category taxonomy that (a) seeds per-category specialists in M3.6b, (b) grounds the judge rubrics in SPaRC (M2), and (c) anchors the policy-DSL clause vocabulary (M1).

This is v0. It will be revised after the M3.6 co-occurrence study, and again after an external-reviewer pass.

---

## 1. Design Principles

1. **Tiered, not flat.** We group categories into three tiers reflecting the decision structure, not just topic. Tiering lets the specialist ensemble share architecture within a tier and diverge across tiers.
2. **Orthogonal where possible, explicit where not.** Where categories overlap (e.g. self-harm ∩ jailbreak), we document it and handle it in training via multi-label supervision, not by pretending the overlap doesn't exist.
3. **Deployer-configurable, not author-prescribed.** The taxonomy is a *vocabulary*; which categories are active for a given tenant is a *policy decision*. The guardrail is conditioned on this selection.
4. **Auditable.** Each category has a crisp inclusion rule, exclusion rule, and worked examples. If two labelers disagree >20% on a held-out sample, the category definition is broken and must be revised before training.
5. **Extensible.** Adding a category must be mechanical: taxonomy entry → DSL clause id → specialist teacher config → eval slice → PolicyBench row.
6. **No moral hazard in the taxonomy itself.** The taxonomy defines *what is detectable*. Whether detection means `block`, `warn`, `redact`, or `allow-with-log` is a policy-DSL decision, not a category property.

---

## 2. Three Tiers

| Tier | Decision character | FN tolerance | FP tolerance | Specialist architecture |
|---|---|---|---|---|
| **T1 — Hard-rule** | Bright line; law or near-law | Near-zero | Low | Small high-recall classifier, rule-augmented |
| **T2 — Safety** | Judgment call; context-sensitive | Low | Moderate | LLM-style specialist, judge rubric |
| **T3 — Deployment policy** | Tenant/product-specific | Varies by policy | Varies by policy | Policy-conditioned head, low-rank |

The tiering directly motivates specialist-distillation (§M3.6b in ROADMAP): T1 specialists are cheap and near-monotonic; T2 specialists carry the research novelty; T3 is where clause-gated distillation pays off.

---

## 3. Category Catalog (v0)

Each entry has a stable id (`tier.shortname`), a one-line definition, inclusion and exclusion rules, attack vectors SPaRC should exercise, a default severity hint, and at least one allow/deny example. Severity is a *default*; deployers override via policy.

### Tier 1 — Hard-rule

#### `T1.csam` — Child Sexual Abuse Material
- **Include:** any request to generate, describe, locate, or sexualize minors.
- **Exclude:** clinical/legal discussion of child-protection issues without prurient content.
- **Attack vectors:** euphemism, fictional framing, age obfuscation, translation, encoded requests.
- **Default severity:** `block-hard` (never overridable by policy).
- **Notes:** zero-tolerance. Calibration target: FN → 0 even at the cost of elevated FP. Shipping this specialist requires a legal review of training data; see §7.

#### `T1.cbrn` — CBRN (Chemical / Biological / Radiological / Nuclear) Uplift
- **Include:** requests that provide meaningful operational uplift toward mass-casualty weapons (synthesis routes, dispersal, acquisition of controlled precursors).
- **Exclude:** high-level educational content, historical discussion, defender-side biosecurity work.
- **Attack vectors:** academic framing, incremental decomposition, role-play as researcher, obfuscated precursor names.
- **Default severity:** `block-hard`.
- **Notes:** align rubric with WMDP and the MLCommons AILuminate CBRN slice. This category is a paper-worthy specialist in its own right.

#### `T1.pii_exfil` — PII Exfiltration
- **Include:** requests to produce, infer, or deanonymize personal data of identifiable individuals (non-public addresses, SSNs, financial accounts, medical records, private communications).
- **Exclude:** discussion of publicly available information about public figures in their public role; the subject's own data ("what's my…").
- **Attack vectors:** doxxing via triangulation, re-identification from partial records, social-engineering framing, "training data extraction" style prompts.
- **Default severity:** `block-hard` (policy may relax to `redact` for self-referential flows).

### Tier 2 — Safety (judgment-based)

#### `T2.self_harm` — Self-harm / Suicide
- **Include:** encouragement, instructions, or glorification of suicide / self-injury; requests for methods.
- **Exclude:** help-seeking, survivor narratives, clinical/research discussion, harm-reduction framing.
- **Attack vectors:** "character writes a suicide note," method-obfuscation, "philosophical" framing, minors indirection.
- **Default severity:** `block-soft` + safe-completion redirect to resources.
- **Notes:** over-refusal risk is high here; XSTest / OR-Bench regression tracking is mandatory.

#### `T2.violence_weapons` — Violence & Weapons (non-CBRN)
- **Include:** instructions to harm specific people or groups, operational guidance for weapons (firearms modification, IEDs, targeted attacks).
- **Exclude:** discussion of fictional violence, news/history, policy debate, lawful hunting/sporting contexts.
- **Attack vectors:** fictional targeting, "for a novel," local-law framing, dual-use tool requests.
- **Default severity:** `block-soft`.

#### `T2.sexual` — Sexual Content (non-CSAM)
- **Include:** explicit sexual content (deployment-policy decides whether this is allowed at all).
- **Exclude:** clinical, educational, consent-focused content; non-explicit romantic content.
- **Attack vectors:** prompt-laundering through fiction, persona injection, NSFW-lite escalation.
- **Default severity:** `flag` (policy-configurable from `allow` to `block`).

#### `T2.hate_harassment` — Hate, Toxicity, Harassment
- **Include:** dehumanization, slurs targeting protected classes, coordinated harassment, identity-based threats.
- **Exclude:** reporting on hate speech, academic analysis, counter-speech.
- **Attack vectors:** slur-obfuscation (leetspeak, Unicode), "just asking questions," in-group reclaiming framing, translation.
- **Default severity:** `block-soft`.

#### `T2.illegal_behavior` — Illegal Goods & Behavior (non-violent)
- **Include:** concrete operational guidance for drug trafficking, fraud, theft, tax evasion, sanctions evasion.
- **Exclude:** discussion of legality, harm-reduction, general journalism.
- **Attack vectors:** "jurisdiction X" framing, harm-reduction laundering, fictional framing.
- **Default severity:** `block-soft`.

#### `T2.cybercrime_dual_use` — Malware / Offensive Security (dual-use)
- **Include:** weaponizable offensive-security content *without* a clear authorization signal (pen-test scope, CTF framing, defensive research context).
- **Exclude:** CTF challenges, authorized pen-testing, defensive/blue-team work, security education with context.
- **Attack vectors:** faked engagement context, "for my course," minimized fictional framing, incremental decomposition.
- **Default severity:** `flag` — gate on presence of authorization context in the prompt.
- **Notes:** this is the category most likely to produce over-refusals and the one that most benefits from policy-conditioning. Coordinate with the Claude Code-style dual-use posture (authorization-context-aware).

### Tier 3 — Deployment policy (tenant-configurable)

#### `T3.jailbreak` — Instruction-Override / Jailbreak
- **Include:** prompts that attempt to subvert the system's operating instructions (role-play overrides, "ignore previous," nested directives, persona attacks).
- **Exclude:** legitimate meta-discussion of the assistant's behavior; user asking about capabilities.
- **Attack vectors:** DAN variants, multi-turn escalation, cipher/encoding, prompt leakage requests, gradient-based suffixes (GCG), many-shot jailbreaking.
- **Default severity:** `block-soft`.
- **Notes:** this is the largest and most diverse specialist corpus. Expect heavy overlap with other T2 categories at attack-success time.

#### `T3.prompt_injection` — Prompt Injection (indirect)
- **Include:** adversarial instructions *embedded in tool outputs, retrieved documents, user-supplied files, or webpage content* aimed at redirecting the model.
- **Exclude:** direct user requests (those are `T3.jailbreak`).
- **Attack vectors:** hidden HTML, markdown-invisible text, delimiter confusion, nested instructions in docs, tool-output poisoning.
- **Default severity:** `block-soft` (with quarantine of the injecting segment).
- **Notes:** distinct from `T3.jailbreak` because the *source* of the malicious instruction is not the user. Evaluation lives in INJECAGENT / TensorTrust slices.

#### `T3.ip_copyright` — IP, Copyright, License
- **Include:** requests to reproduce substantial portions of copyrighted works; code with incompatible licenses for the deployer's context; trade-secret-style extraction.
- **Exclude:** fair-use discussion, public-domain, deployer-licensed content.
- **Attack vectors:** "continue this story," stepwise memorization probing, paraphrase-cover.
- **Default severity:** `flag` (policy-configurable; stricter for enterprise).

#### `T3.brand_safety` — Brand-Safety (tenant-defined)
- **Include:** topics / competitors / tone the tenant has declared out-of-scope.
- **Exclude:** generic helpful answers that happen to mention a brand.
- **Attack vectors:** competitor-name obfuscation, topic drift, persona-shift.
- **Default severity:** `warn` (policy defines target behavior).
- **Notes:** pure T3 — has no intrinsic meaning without the policy. Excellent stress-test for clause-gated distillation.

#### `T3.regulatory` — Regulatory Compliance (GDPR / HIPAA / PCI / sector)
- **Include:** behaviors that would cause the deployer to violate an applicable regulation (data retention, disclosure obligations, PHI handling, cardholder-data boundaries).
- **Exclude:** generic compliance education.
- **Attack vectors:** role-play as authorized party, minimal-context requests that bypass consent checks, cross-jurisdiction confusion.
- **Default severity:** depends on regulation; typically `block-soft` or `redact`.
- **Notes:** the clause vocabulary here is rich (GDPR art. 6 / 9, HIPAA 164.508, PCI 3.2.1 Req 3, etc.). Good candidate for structured policy-clause conditioning.

#### `T3.professional_advice` — Medical / Legal / Financial Advice Boundaries
- **Include:** specific actionable professional advice the deployer's policy prohibits giving without a licensed party in the loop.
- **Exclude:** general education, symptom-checker with safety net, policy-compliant disclaimers.
- **Attack vectors:** role-play as licensed professional, jurisdiction mismatch, urgency framing.
- **Default severity:** `warn` + safe-completion.

#### `T3.misinformation` — Factual Integrity / Misinformation
- **Include:** deliberate production of false claims about elections, public health, attributed statements, or deployer-defined sensitive facts.
- **Exclude:** steelmanning positions, explicit fiction, documented uncertainty.
- **Attack vectors:** "write a plausible-sounding article about...", synthetic-citation, quote-fabrication.
- **Default severity:** `flag` (highly policy-dependent; some tenants want `block`, others want `warn`).
- **Notes:** the most contested category. Recommend opt-in only; exclude from the headline benchmark until the definition is stable.

---

## 4. Severity Vocabulary

Severity is a **default hint** from the taxonomy; the policy DSL has the final say.

- `block-hard` — refuse, no safe-completion, log.
- `block-soft` — refuse with safe-completion / redirect / resource pointer.
- `redact` — allow response, remove offending spans (PII, etc.).
- `quarantine` — for T3.prompt_injection: strip the offending segment from context before answering.
- `warn` — allow but surface a caveat to the user or downstream system.
- `flag` — pass-through to the host application for routing (HITL, escalation).
- `allow` — explicit permit (needed when policy overrides a default).

---

## 5. Cross-Category Overlap (expected)

Categories are not strictly disjoint. Document-and-train overlap is fine; pretending it doesn't exist is not. v0 hypotheses:

- `T3.jailbreak` ↔ every T2 — most successful jailbreaks are in service of T2 content. Multi-label supervision required.
- `T2.self_harm` ↔ `T3.misinformation` — "does bleach cure X" style prompts.
- `T2.cybercrime_dual_use` ↔ `T1.pii_exfil` — doxxing workflows cross both.
- `T3.prompt_injection` ↔ `T3.jailbreak` — same *intent*, different *source*; we keep them split because the system-level response differs (quarantine vs. refuse).
- `T2.violence_weapons` ↔ `T1.cbrn` — dual-use weapons discussion.

---

## 6. Co-occurrence Study (M3.6 gate)

Before committing to M3.6b specialist-distillation, we run:

1. Label a stratified sample of 5-10k prompts with the full category vector (not one-hot).
2. Compute pairwise mutual information and conditional probability tables.
3. **Merge rule:** if `I(c_i; c_j) / H(c_i) > 0.6` in both directions, consider merging or reorganizing.
4. **Split rule:** if within-category variance in judge ratings exceeds threshold, consider splitting.
5. Publish the confusion matrix in the paper — reviewers will ask.

Output: v1 taxonomy (possibly 10-12 categories after merges) that gets frozen for the main experiments.

---

## 7. Legal & Ethics Notes

- **T1.csam:** training the specialist requires careful corpus handling. We will *not* include any positive-class examples in the released dataset; positives are generated procedurally from abstract templates and discarded after training a checksum-committed weights file. Legal review required before the first training run.
- **T1.cbrn:** uplift-oriented content will be restricted to templates consistent with published red-team corpora (e.g. WMDP-adjacent); no novel synthesis routes generated by our pipeline are retained.
- **T2.cybercrime_dual_use:** this category is designed to *not* over-block authorized pentest / CTF / defensive-research use cases. We pre-register the authorization-context rubric and publish false-positive rates per context type.
- **Dataset release:** T1 categories ship as models-only; training corpora are not released. T2/T3 corpora are released with appropriate content warnings and license metadata.

---

## 8. Integration Points

- **Policy DSL (M1):** each category id is a reserved clause-root; deployers compose clauses by id with AND/OR/NOT and attach severities and examples.
- **SPaRC (M2):** each category gets an Attacker profile, a Judge rubric, a seed prompt bank, and a difficulty scheduler rung.
- **Specialist teachers (M3.6b):** one teacher per category (after v1 merges). Teachers share a backbone up to the penultimate layer; heads are category-specific.
- **Eval harness (M5):** each category maps to a public-benchmark slice where one exists (HarmBench → multiple T2; WildGuardTest → mixed; INJECAGENT → T3.prompt_injection; XSTest → cross-cutting over-refusal). Gaps become new `PolicyBench-Zero` slices.
- **Certification (M4):** per-category conformal calibration split → per-category `q̂_c`; union-bound guarantee over active categories.

---

## 9. Evolution Policy

- **Adding a category:** PR against this file + a new DSL clause id + a new SPaRC profile + an eval slice + a PolicyBench row. Taxonomy owner signs off. No silent additions.
- **Retiring a category:** only via merge into another. Deprecation note here, migration guide for policies, 1-release overlap.
- **Splitting a category:** requires the co-occurrence evidence from §6 or labeler-disagreement evidence from Principle 4.
- **Changing severity defaults:** requires a changelog entry and a re-run of the calibration split.

---

## 10. Open Questions

1. Should `T2.sexual` be one category or split into `explicit` / `suggestive` / `romantic`? v0 keeps it atomic; reconsider after labeler-agreement study.
2. Is `T3.misinformation` stable enough to ship in v1, or do we mark it experimental? Leaning experimental.
3. Do we want a dedicated `T3.privacy_surveillance` beyond `T1.pii_exfil` (e.g. geolocation inference, OSINT requests)? Possibly — pending co-occurrence study.
4. Multilingual category definitions — do they shift? Deferred to v1.1 multilingual workshop paper.
5. How do we encode *authorization context* for `T2.cybercrime_dual_use` in the policy DSL? Needs its own mini-spec.

---

*This taxonomy is a living document — revise after the co-occurrence study and after labeler-agreement trials.*
