# HAVEN — Project Trade-Offs and Decisions
**Project:** HAVEN — AI Care Marketplace Intelligence
**Author:** Samveda Sharma
**Last Updated:** April 8, 2026

> This file logs every meaningful product and technical decision made during the HAVEN build — including the question that prompted it, the options considered, and the reasoning behind the choice. It is a living document. Add to it whenever a non-obvious decision is made.

---

## How to Read This Log

Each entry has:
- **The question** — what was being decided
- **Options considered** — what was on the table
- **Decision** — what was chosen
- **Reasoning** — why, including any tradeoffs acknowledged

---

## Decision 1 — Model Selection: Agent 2 and Agent 3

**Date:** April 6, 2026

**The question:**
Should Agent 2 use Haiku (cheaper, faster) and Agent 3 use Sonnet — or Sonnet on Agent 2 and Haiku on Agent 3?

**Options considered:**

| Option | Agent 2 | Agent 3 |
|---|---|---|
| A | Haiku | Sonnet |
| B | Sonnet | Haiku |
| C (chosen) | Sonnet | Sonnet |

**Decision:** Sonnet on both Agent 2 and Agent 3.

**Reasoning:**

*Why not Haiku on Agent 2?*
Agent 2 contains a chain of thought reasoning step for deal-breaker evaluation. This is not structured extraction — the model must infer whether a deal-breaker applies based on indirect signals in a helper's bio (e.g., inferring "youth in demeanor" from age + bio language rather than an explicit statement). On clear cases, Haiku would likely work. On adversarial family cases with vague or mixed signals, Haiku is more likely to miss a deal-breaker that requires multi-signal inference. The scoring task is straightforward; the deal-breaker step is not.

*Why not Haiku on Agent 3?*
Agent 3 produces the only output a real family ever reads. It has three simultaneous requirements: ground every claim in specific bio facts, mirror the family's own language and emotional register, and calibrate tone appropriately for a high-stakes care decision. Haiku produces adequate sentences. It defaults to generic care language ("experienced and caring") — exactly what the eval's "B — Generic" failure label catches. Emotional calibration and language mirroring are Sonnet-level tasks. This is also the output scored on Empathy Tone in the eval rubric, so model quality here directly affects eval outcomes.

*Why not run a cost vs. accuracy study first?*
For a 10-case portfolio eval, the cost difference between Haiku and Sonnet is negligible. In production at scale, a cost vs. accuracy tradeoff study would be the right next step — run both models on 50+ cases, compare eval scores, calculate cost per quality point. That is worth noting as a production consideration but not worth doing here.

**Production note:** If HAVEN were moving toward production, the right approach would be to benchmark Haiku vs. Sonnet on Agent 2 and Agent 3 separately across a larger eval set, calculate cost per quality point on each dimension, and set a quality floor below which the cheaper model is not acceptable.

---

## Decision 2 — Pipeline Data Flow: Agent 2 User Message

**Date:** April 8, 2026

**The question:**
Agent 2 needs two inputs — the care profile from Agent 1 (dynamic, changes per family) and the helpers pool (static, same data every run). How should helpers data be fed into Agent 2 without manually copy-pasting it each time?

**Options considered:**

| Option | How | Tradeoff |
|---|---|---|
| A | Hardcode helpers JSON directly in Agent 2's user message field | Simple, no extra nodes — but painful to update when scaling from 20 to 50–100 helpers |
| B | Fetch helpers.json from GitHub via HTTP Request node (chosen) | Slightly more setup — but updating helpers means updating one file on GitHub, not editing the n8n workflow |
| C | Read from local disk (Read/Write Files from Disk node) | Only works on n8n desktop/self-hosted, not cloud |

**Decision:** HTTP Request node fetching helpers.json from GitHub (Option B).

**Reasoning:**
The helpers pool is not truly static — as the eval scales from 20 to 50 or 100 helpers, the data will change. Hardcoding it in the user message means editing the n8n workflow every time. Fetching from GitHub decouples the data from the pipeline: update the file, the workflow picks it up on the next run automatically. This is also a real integration pattern — a portfolio artifact that shows awareness of data pipeline design, not just prompt engineering.

**Production note:** In a production system, helpers would be fetched dynamically from a database keyed to the family's geography, care type, and availability. The GitHub fetch in this prototype is the closest analogue achievable without a backend.

---

## Decision 3 — Agent 2 to Agent 3 Data Handoff

**Date:** April 8, 2026

**The question:**
Agent 2 returns a full JSON with two arrays: `excluded_by_deal_breaker` (audit log) and `top_3` (match data). Agent 3 only needs `top_3`. How do we pass only the relevant data forward without Agent 3 receiving noise?

**Options considered:**

| Option | How | Tradeoff |
|---|---|---|
| A | Pass full Agent 2 output to Agent 3, instruct it via prompt to use only `top_3` | No extra nodes — but relies on the model to ignore data it was given, which is a weak guarantee |
| B | Add a Code node between Agent 2 and Agent 3 to extract `top_3` (chosen) | One extra node — but Agent 3 never sees `excluded_by_deal_breaker` at all, which is a hard guarantee |

**Decision:** Code node between Agent 2 and Agent 3 (Option B).

**Reasoning:**
Telling a model to ignore part of its input is a soft instruction — it usually works, but it can fail on edge cases and is harder to debug when it does. Filtering at the pipeline level is a hard guarantee: Agent 3 receives exactly what it needs and nothing else. This also keeps concerns separated cleanly — audit data stays in Agent 2's output, match data flows to Agent 3. The extra node is worth the architectural clarity.

**Production note:** In production, this filtering would happen at the orchestration layer — a dedicated service would route the right data to the right downstream agent. The Code node is the prototype equivalent.

---

## Decision 4 — Temperature Setting on Agent 2

**Date:** April 11, 2026

**The question:**
Agent 2's scores fluctuate across runs — Rosa's `schedule_fit` scores between 2.4 and 2.5 depending on the run, changing her composite score and ranking. Should temperature be set to 0 to force deterministic scoring?

**Options considered:**

| Option | Temperature | Tradeoff |
|---|---|---|
| A | 0 (fully deterministic) | Eliminates score variation — but also eliminates the model's ability to make nuanced judgements on ambiguous inputs |
| B (chosen) | Default | Preserves subjectivity in interpretation — variation is an honest signal that the input is ambiguous |

**Decision:** Leave temperature at default. Do not force determinism.

**Reasoning:**
Rosa's schedule starts at 7:30am and the family stated "weekday mornings" without specifying exact hours. The model's uncertainty about whether this is a full match (2.5) or a minor gap (2.4) is not a bug — it is an accurate reflection of genuine ambiguity in the care profile. Forcing temperature to 0 would make the model pick one answer and stick to it, but that answer would be arbitrary, not correct. The variation is the system being honest about what it doesn't know. For a production system, the right fix is to prompt the family to clarify their start time — not to suppress the model's uncertainty. Scoring instability on ambiguous inputs will be documented in the failure taxonomy as a known limitation.

**Production note:** In production, ambiguous schedule inputs would trigger a clarification prompt to the family (already handled by Agent 1's `clarification_needed` field). Once the family provides a specific start time, the scoring variation disappears.

---

## Decision 5 — Test Case Design: Edge Cases and Ethical Guardrails in families.json

**Date:** April 11, 2026

**The question:**
Beyond the standard clear/medium/hard taxonomy, what additional family cases should be included to stress-test the pipeline on edge cases the real product would encounter?

**Decision:** Add four additional cases (F014–F017) covering budget precision, budget impossibility, gender preference, and race-based preference.

**Reasoning per case:**

*F014 — Strict single-number budget ($28/hr, no range):*
Most families state a range. A single hard number tests whether Agent 1 correctly extracts it as a fixed value and whether Agent 2's budget gate (upper_bound × 1.20) handles a single number correctly. Upper bound of $28 is $28 itself, so the threshold is $33.60. This is a real scenario — some families have an exact approved amount from insurance or a fixed income allocation.

*F015 — Budget below all helper rates ($13-15/hr):*
All helpers in the pool start at $16/hr. Agent 2's budget gate will exclude every helper. The pipeline must handle an empty top_3 gracefully — returning a clear explanation rather than crashing or hallucinating matches that do not exist. This tests system resilience at a boundary condition.

*F016 — Gender preference (female helper only):*
A legitimate and common family preference. The helper pool does not have a gender field, which means Agent 2 cannot score or filter on this dimension — it is a data gap. Agent 1 should capture it as a must_have. Agent 2 should flag it in profile_gaps for every helper. This tests how the system handles a preference it cannot operationalise with available data.

*F017 — Race-based preference (ethical guardrail):*
The family has framed a racial preference as a cultural comfort request — a common real-world pattern. This is against platform ethics and cannot be honoured. The system must handle it without being preachy or dismissive. The correct behaviour: Agent 1 captures the cultural familiarity preference as a must_have (framing it as cultural fit, not race), and Agent 2 ignores racial signals entirely while scoring on relevant dimensions. The golden set for this case will define what "handled skilfully" means — acknowledging the family's intent while not operationalising the racial preference.

**What this demonstrates in an interview:**
Including these cases shows you thought about what happens at the edges — not just the happy path. The ethics guardrail case in particular shows product judgement: you identified a scenario where user input conflicts with platform values and designed a test to validate the system's response.

---

## Decision 6 — golden_set.json Schema: Difficulty Field Excluded

**Date:** April 12, 2026

**The question:**
Should `golden_set.json` include a `difficulty` field for each family case?

**Decision:** No — difficulty field excluded from golden_set.json.

**Reasoning:**
Difficulty is already captured in `families.json` and linked to the golden set via `family_id` — duplicating it adds no new information. More importantly, including it in the golden set would encourage the LLM-as-Judge to lower its standards for hard cases, which is the wrong design. The golden set entry itself defines what good looks like for each case — the bar is already adjusted at the entry level by writing different golden entries for different case types. A hard case with a vague input has a simpler, lower-expectation golden entry than a clear case. If the judge needs a difficulty label to calibrate, it means the golden entries are not specific enough. The fix is to write better golden entries, not add a difficulty modifier.

---

## Decision 7 — LLM-as-Judge Model Selection: GPT-4.1 over Claude Sonnet 4.6

**Date:** April 21, 2026

**The question:**
Which model should be used as the LLM-as-Judge for evaluating Agent 3's match explanations — and should it be from the same model family as the generator?

**Options considered:**

| Option | Model | Family | Tradeoff |
|---|---|---|---|
| A | Claude Sonnet 4.6 | Anthropic (same as generator) | Simplest setup — but same model family evaluating its own outputs introduces self-serving bias |
| B | GPT-4.1 | OpenAI (chosen) | Different model family eliminates self-serving bias; strongest instruction following for structured eval tasks |
| C | GPT-4o | OpenAI | Proven in eval frameworks, cost-effective — slightly less precise than GPT-4.1 on nuanced structured reasoning |
| D | Gemini 2.5 Pro | Google | Strong reasoning and long context — JSON output less consistent across runs than GPT-4.1 |

**Decision:** GPT-4.1 (OpenAI)

**Reasoning:**

*Why not Claude Sonnet 4.6?*
Using the same model family as the generator creates self-serving bias — the model is evaluating outputs it would itself produce, which leads to inflated scores on dimensions like Empathy Tone and Specificity. This is a well-documented failure mode in LLM-as-Judge pipelines. The eval loses its independence and becomes a validation loop rather than a genuine quality check. For a portfolio that demonstrates eval design understanding, using the same model would be the wrong signal.

*Why GPT-4.1 over GPT-4o?*
The judge prompt requires a strict sequence: quote the explanation → reason → score, for four dimensions, returning a precise JSON schema. Instruction following consistency across multiple runs matters more than raw reasoning power here. GPT-4.1 is OpenAI's current strongest model for structured instruction adherence. GPT-4o is a reasonable alternative and significantly cheaper — if cost becomes a concern at scale, switching to GPT-4o is the first optimisation to evaluate.

*Why GPT-4.1 over Gemini 2.5 Pro?*
Gemini 2.5 Pro has strong reasoning but less consistent JSON output behaviour. For an eval pipeline that parses judge output programmatically, schema consistency across runs is non-negotiable. GPT-4.1's JSON mode is more reliable.

**Production note:** In a production eval system, model selection for the judge would be validated empirically — run the same eval set through multiple judge models, compare inter-rater agreement against human annotations, and select the model whose scores correlate most closely with human judgement. For this prototype, GPT-4.1 is the principled default pending that validation.

---

## Decision 8 — Agent 2 Qualifications Scoring: Relevance-Strict over Volume-Friendly

**Date:** April 20, 2026

**The question:**
The Qualifications Match dimension in Agent 2 was rewarding helpers for holding many certifications regardless of whether those certifications matched the care profile's needs. Should the dimension be rewritten to score strictly on relevance to the care profile?

**Options considered:**

| Option | Approach | Tradeoff |
|---|---|---|
| A | Keep volume-friendly scoring — more certifications = higher score | Simple rubric — but inflates scores for over-qualified helpers on low-acuity families |
| B | Relevance-strict scoring anchored to care profile's explicit and inferred qualifications (chosen) | Accurate signal — a helper with 5 irrelevant certs scores 0, a helper with 1 relevant cert scores 1.8–2.0 |
| C | Dynamic dimension weights by care type | Most accurate architecture — but requires care_type classification logic upstream. Flagged for v7. |

**Decision:** Relevance-strict scoring (Option B), with dynamic weights flagged as a known future improvement.

**Reasoning:**
The core problem with volume-friendly scoring is that it rewards credential accumulation rather than fit. A helper with CDP + CADDCT + CHPNA caring for a companion-only family (where inferred_qualifications = []) was scoring 1.5+ on qualifications despite none of those certifications being relevant. This distorted rankings — over-qualified helpers were beating better-fit helpers on a dimension that should have contributed nothing for that family type.

The relevance-strict fix anchors scoring entirely to the care profile's `explicit_qualifications` and `inferred_qualifications` fields, which are set by Agent 1 per family. A helper only earns qualifications score for certifications that appear in those fields. Volume is explicitly irrelevant — 5 relevant certifications scores the same as 1.

The known side effect: for companion care families where `inferred_qualifications = []`, the qualifications dimension collapses to 0 for all helpers, dropping the effective composite maximum to 8.0. This makes cross-family score comparisons misleading. The correct long-term fix is dynamic dimension weights based on care_type — qualifications weight drops for companion care, elevates for post-surgical and dementia care. This is the v7 improvement.

**Production note:** Dynamic dimension weights would require Agent 1's `care_type` output to map to a defined weight table. Agent 1 already extracts `care_type` — the infrastructure is in place. The v7 prompt change is purely in Agent 2's scoring logic.

---

## Decision 9 — Eval Infrastructure: Jupyter Notebooks over Dedicated Eval Platforms

**Date:** April 22, 2026

**The question:**
Should the HAVEN eval pipeline be built in Jupyter Notebooks or on a dedicated LLM eval platform like LangSmith, Braintrust, or Arize Phoenix?

**Options considered:**

| Option | Platform | Tradeoff |
|---|---|---|
| A | Jupyter Notebooks (chosen) | Fully transparent, portable, no vendor dependency — but no persistent experiment tracking or team dashboard |
| B | Braintrust | Purpose-built for LLM evals, native LLM-as-judge, prompt versioning, run comparison UI — requires account and platform setup |
| C | LangSmith | Best-in-class LLM call tracing and experiment tracking, strong LangChain integration — more complex setup, better suited for multi-run production workflows |
| D | Arize Phoenix | Strong production observability and drift monitoring — overkill for a prototype eval |

**Decision:** Jupyter Notebooks

**Reasoning:**

*Why not a dedicated platform for this prototype?*
At 17 families and a single prompt version per agent, the overhead of configuring LangSmith or Braintrust is not justified. The primary audience for this eval is a portfolio reviewer — not a product team running weekly eval cycles. Jupyter keeps the logic fully visible and runnable with just Python and two API keys. No accounts, no platform context needed to interpret results.

Building the eval logic from scratch also demonstrates understanding of what these platforms do under the hood — not just the ability to configure a UI.

*Why dedicated platforms are the right call in production?*
Jupyter does not solve three production-critical problems:
1. **Experiment tracking** — comparing Agent 2 v5 scores vs. v6 scores side by side requires manually managing output files. LangSmith and Braintrust track this automatically per run.
2. **Team visibility** — sharing eval results requires re-running the notebook. A dashboard gives the whole team access without code.
3. **Production monitoring** — Arize Phoenix and LangSmith can alert when live scores drop below a threshold. Jupyter cannot monitor production traffic.

*Which platform for production?*
Braintrust is the closest architectural match to what was built here — it has a native LLM-as-judge scorer, supports custom rubrics, versions prompts, and compares runs. Migrating from this Jupyter setup to Braintrust would mean replacing the judge notebook with Braintrust's scorer API and pointing it at the same GPT-4.1 judge prompt. The prompt logic stays identical — only the infrastructure changes.

**Production note:** In a production HAVEN system, the eval pipeline would run on Braintrust for offline prompt evaluation, LangSmith for tracing every live agent call, and Arize Phoenix for monitoring score drift over time across real family requests.

---

## Infrastructure Findings & Configuration Learnings

---

### Finding 1 — Agent 3 output truncated mid-word ("attness")

**Date:** April 8, 2026

**What happened:**
Agent 3's explanation for rank 3 helper contained "attness" — a truncated version of "attentiveness." The word was cut mid-character in the output.

**Root cause:**
n8n's Anthropic node has a default Max Tokens setting that caps output length. When the output hits that limit, the model's response is cut off at whatever character the limit falls on — even mid-word. This is a node configuration issue, not a prompt failure.

**Fix:**
In the Agent 3 node settings → Max Tokens → increase from default (1024) to **2048**. Three explanations of 3–4 sentences each do not require more than 2048 tokens, but 1024 is not always sufficient depending on sentence length.

**Learning:**
Always check Max Tokens when a model output looks abruptly cut off. This applies to all three agent nodes. If outputs start truncating in later runs (longer family descriptions, more verbose helpers), increase Max Tokens on the affected node. The fix is always in node configuration, never in the prompt.

---

## Decision 10 — Agent 2 Scoring Dimension Weights: Schedule Fit Elevated to Match Care Experience

**Date:** April 27, 2026

**The question:**
After the v7 eval run (8/17 rank-1 accuracy), analysis of the remaining failures showed helpers with stronger care experience consistently outranking helpers with better schedule coverage — even when schedule was the binding constraint. Should Schedule Fit carry more weight?

**Options considered:**

| Option | Schedule Fit | Care Experience | Skills Match | Quals Match | Total |
|---|---|---|---|---|---|
| A — original | 0–2.5 | 0–3.0 | 0–2.5 | 0–2.0 | 10.0 |
| B — chosen | 0–3.0 | 0–3.0 | 0–2.0 | 0–2.0 | 10.0 |
| C — schedule dominant | 0–3.5 | 0–2.5 | 0–2.0 | 0–2.0 | 10.0 |

**Decision:** Option B — Schedule Fit raised to 0–3.0 equal to Care Experience. Skills Match reduced from 2.5 to 2.0. Composite maximum stays at 10.0.

**Reasoning:**
The original weighting created a structural bias: a helper with high care experience and poor schedule fit would consistently outrank a helper with strong schedule fit and moderate experience. This produced Type B failures (F002, F005, F016) where the right helper was in the top 3 but wrongly ranked second. Elevating Schedule Fit to 3.0 makes both primary matching dimensions equal — neither can systematically dominate the other. Skills Match reduction from 2.5 to 2.0 funds the increase; skills are partly a consequence of care experience and the two dimensions were redundant in weight.

**Tradeoff acknowledged:**
Equal weighting means Schedule Fit cannot be overridden by Care Experience even in cases where experience matters more (e.g., MS care). The mitigation is the deal-breaker check in STEP 2 — experience requirements the family stated explicitly are enforced as hard exclusions before scoring. Within the surviving pool, equal weighting is the correct default.

**Production note:** Dynamic dimension weights per care_type remain the architecturally correct long-term fix. Equal weighting is a principled approximation that works across care types without requiring per-care-type rubric configuration.

---

