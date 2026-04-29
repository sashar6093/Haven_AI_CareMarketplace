# HAVEN — AI Care Marketplace Intelligence System
### Product Requirements Document (PRD) v2.0
**Author:** Samveda Sharma | **Date:** April 28, 2026 | **Status:** Prototype Complete — Eval Validated

> **v2.0 scope:** This PRD documents the full HAVEN product vision (four-agent production architecture) alongside the working three-agent prototype built and evaluated in this repository. Sections clearly distinguish between **[BUILT]** (prototype, eval-validated) and **[PLANNED]** (production roadmap). The prototype demonstrates the core matching intelligence pipeline end-to-end — need articulation, helper matching and ranking, and explanation generation — with a documented eval framework and real performance results.

---

## Table of Contents
1. [Approvals](#approvals)
2. [Abstract](#abstract)
3. [Business Objectives](#business-objectives)
4. [KPIs](#kpis)
5. [Success Criteria](#success-criteria)
6. [User Journeys](#user-journeys)
7. [Multi-Agent System Design](#multi-agent-system-design)
8. [Functional Requirements](#functional-requirements)
9. [Model Requirements](#model-requirements)
10. [Data Requirements](#data-requirements)
11. [Prompt Requirements](#prompt-requirements)
12. [Eval Spec — Testing and Measurement](#eval-spec)
13. [Eval Results — Prototype](#eval-results)
14. [Eval Gates — Risks and Mitigations](#eval-gates)
15. [AI Costs and Latency](#ai-costs-and-latency)
16. [Assumptions and Dependencies](#assumptions-and-dependencies)
17. [Compliance / Privacy / Legal](#compliance--privacy--legal)
18. [GTM / Rollout Plan](#gtm--rollout-plan)

---

## 👍 Approvals

| ROLE | TEAMMATE | REVIEWED | STATUS |
|---|---|---|---|
| Product | Samveda Sharma | 2026-04-28 | Approved |
| Engineering | TBD | — | Pending |
| UX | TBD | — | Pending |
| Legal / Privacy | TBD | — | Pending |

---

## 📝 Abstract

A care marketplace has a matching problem that is fundamentally different from every other marketplace. A family searching for care for an elderly parent with dementia, or a sibling with a disability, is not making a transactional purchase decision. They are making a trust decision — inviting a stranger into their most private space to care for someone they love. The cost of a bad match is not a return request. It is distress for a vulnerable person, broken trust in the platform, and in many cases, a family who abandons online care altogether.

Most care marketplace platforms treat matching as a search-and-filter problem: location, availability, hourly rate. This surface-level approach fails at scale because the information required for a good care match is neither structured nor symmetric. Families struggle to articulate care needs — not because they don't know what they need, but because the language of care is personal, situational, and emotionally loaded. And neither side has enough information about the other to build the trust required to take the first step.

**HAVEN** is an AI intelligence system that resolves the information asymmetry on both sides of the care marketplace — transforming a family's free-text, emotionally complex care description into a structured profile, matching it against a helper pool with explicit deal-breaker logic and multi-dimensional scoring, and synthesizing the result into match explanations that families trust enough to act on.

### Production Vision (Four-Agent Architecture)

| AGENT | ROLE |
|---|---|
| **Agent 1 — Care Need Articulator** | Transforms a family's free-text care description into a structured, semantically rich care profile through empathetic multi-turn conversation |
| **Agent 2 — Helper Profile Intelligence** | Continuously scores helper profiles for completeness and market fit; generates specific, actionable profile improvement recommendations |
| **Agent 3 — Trust Signal Engine** | Aggregates behavioral signals — re-booking rates, cancellation patterns, review sentiment, response times — into an explainable trust score surfaced in human language |
| **Agent 4 — Match Quality Agent** | Takes the structured care profile, enriched helper profiles, and trust scores as inputs; produces a ranked, explained match list grounded in specific, verifiable data |

### Prototype (Three-Agent Pipeline — This Repository)

The prototype built and evaluated here implements the core matching intelligence loop:

- **Agent 1 — Care Need Articulator** `[BUILT]` — Converts family free-text to a structured care profile JSON with deal-breakers, must-haves, schedule, budget, required skills, inferred qualifications, and a confidence score.
- **Agent 2 — Helper Matching and Ranking Agent** `[BUILT]` — Applies budget gating, hard/soft deal-breaker logic, and a multi-dimensional scoring rubric (Schedule Fit, Care Experience, Skills Match, Qualifications Match) to rank helpers from a pool.
- **Agent 3 — Explanation Agent** `[BUILT]` — Generates personalized, grounded, empathetic match explanations for the top-ranked helpers — the text a family actually reads when deciding whether to reach out.

The core product insight is that **in a care marketplace, the explanation is the product**. A family does not book a helper because they scored 91/100 on a trust index. They book because they read "Samuel has five years of experience supporting younger adults with disabilities and is close to your brother in age" and felt something shift from anxiety to confidence. HAVEN's job is to create that shift — at scale, for every family, every time.

---

## 🎯 Business Objectives

1. **Improve first-match quality** — reduce the number of trial-and-error bookings a family needs before finding a long-term helper; first-match failures are the primary driver of early family churn.
2. **Increase trust conversion** — close the gap between families who browse helpers and families who complete a first booking, where trust and explanation quality are the primary barriers.
3. **Maximize re-booking rate** — the platform's gold-standard metric. A family who re-books the same helper is a retained family, a retained helper, and a validation that the match was genuinely good.
4. **Increase supply-side booking rate** `[PLANNED]` — ensure qualified helpers receive consistent bookings by surfacing their genuine capabilities to the families who need them most, reducing helper dropout from underutilization.
5. **Serve as an architectural foundation** — demonstrate that care matching can be resolved through structured reasoning over deal-breakers and multi-dimensional scoring, not just keyword search — and that this approach is eval-measurable, auditable, and improvable.

---

## 📊 KPIs

| GOAL | METRIC | MEASUREMENT QUESTION |
|---|---|---|
| Match Quality | Rank-1 acceptance rate (family contacts top-1 recommended helper) | Are HAVEN's top recommendations good enough to act on without scrolling to alternatives? |
| Trust Conversion | Browse-to-booking conversion rate (families who view a HAVEN recommendation → complete a booking) | Is the trust explanation closing the decision gap? |
| Re-booking Rate | % of families who re-book the same helper within 30 days of first session | Is the first match genuinely good, not just acceptable? |
| Explanation Quality | LLM-as-Judge scores (Specificity, Relevance, Factual Grounding, Empathy) on Agent 3 outputs | Is the explanation the product — or is it noise? |
| Need Profile Quality | % of required dimensions populated in Agent 1 output; confidence score distribution | Is the Care Need Articulator producing profiles rich enough to power meaningful matching? |
| Supply Utilization `[PLANNED]` | % of active helpers receiving ≥ 1 booking per week | Is profile intelligence improving supply-side visibility enough to reduce helper dropout? |
| Trust Score Calibration `[PLANNED]` | Correlation between Agent 3 trust score and 90-day re-booking rate | Is the trust score predicting actual care relationship quality? |

---

## 🏆 Success Criteria

### Prototype (Eval-Measurable — This Repository)

- **Rank-1 match accuracy ≥ 65%** on the 17-family golden evaluation set — the pipeline recommends the correct helper first across a representative distribution of difficulty levels (clear, medium, hard, no-match).
- **Agent 3 Specificity ≥ 4.5/5** — explanations cite concrete, verifiable facts from the helper's profile rather than generic language.
- **Agent 3 Factual Grounding ≥ 98% pass rate** — every claim in an explanation is traceable to the helper's profile data. Zero hallucinated credentials.
- **Agent 3 Empathy Tone ≥ 4.5/5** — explanations are warm and human, appropriate for a family making a high-stakes care decision.
- **Agent 1 confidence in expected range ≥ 75%** of families on the golden set — the articulator's self-assessed confidence correlates with actual profile completeness.

### Production (Outcome-Measurable — Post-Launch)

- **First-match acceptance rate ≥ 60%** — more than half of families book the top-1 HAVEN recommendation without browsing alternatives.
- **Browse-to-booking conversion ≥ +25%** over baseline (families shown HAVEN recommendations vs. standard search results).
- **Re-booking rate ≥ 55%** within 30 days of first session.
- **Zero hallucinated helper capabilities** in any match recommendation served to families — a helper must never be recommended for a care type they have not demonstrated. Hard gate, zero tolerance.
- **Zero sensitive information mishandling events** — Agent 1 must never store, log, or transmit a raw medical diagnosis stated by a family.

---

## 🚶 User Journeys

### Scenario 1 — Family Side: Need Articulation → Match → Explanation

A family in San Jose has been caring for their 78-year-old mother at home. She has early-stage memory challenges and needs help with meals, light housekeeping, and companionship four hours a day, five days a week. The daughter types into the platform: *"I need someone to help my mom who is 78 and a little forgetful. She's very proud and doesn't want to feel like she needs a nurse. She's Filipino and would love someone who speaks Tagalog. She's still pretty independent but needs some support."*

**Agent 1** identifies the core dimensions, recognizes schedule and budget are unpopulated, and returns a structured care profile with confidence 0.91, including: deal-breakers (`Helper must not have a clinical demeanor`), must-haves (cultural sensitivity, Tagalog preferred), schedule (weekday mornings), budget range, and inferred qualifications (dementia care training, meal preparation experience).

**Agent 2** applies the deal-breaker filter — excluding any helper flagged as clinical in manner — applies the budget gate, scores the remaining pool across Schedule Fit, Care Experience, Skills Match, and Qualifications Match, and returns the top 3 helpers with composite scores.

**Agent 3** produces: *"Maria brings nine years of experience supporting elderly clients navigating memory changes — your mom's independence and dignity will be central to how she works, not an afterthought. She speaks Tagalog fluently, is available every weekday morning, and five of her last six families chose to continue working with her long-term."*

The family reads this and feels something shift — from search anxiety to a specific reason to reach out.

### Scenario 2 — No-Match Scenario: Honest Fallback

A family is looking for a helper experienced in multiple sclerosis care for their 34-year-old brother. The helper pool has no one with MS-specific experience. **Agent 2** recognizes this as a no-match situation and does not fabricate a confident recommendation. **Agent 3** surfaces: *"We don't currently have helpers with specific MS experience, but Samuel is 29 years old and has five years working with adults with physical disabilities — he brings relevant experience and is close to your brother in age, which your family said mattered."*

Honest fallback is better than a confident wrong match.

### Scenario 3 — Hard Deal-Breaker Enforcement

A family specifies that they require a non-smoker — their mother has severe asthma. Three helpers in the pool are identified as smokers in their profiles. **Agent 2** excludes all three at the deal-breaker stage, before scoring begins. They never appear in the recommendation, regardless of how high they score on other dimensions. Deal-breakers are not weights — they are gates.

### Scenario 4 — Companion Care, Non-Clinical Family

A family's 84-year-old father needs companionship, light meal prep, and someone to drive him to appointments. He is in good health and explicitly does not need medical support. **Agent 1** flags clinical care as a soft deal-breaker — the family wants warmth, not credentials. **Agent 2** deprioritizes qualifications in scoring for this care type and ranks highest the helper who most clearly demonstrates companionship experience and a warm, relationship-oriented approach — not the most clinically credentialed one.

### User Flow

```
FAMILY SIDE — PATH A (Matching Pipeline)
──────────────────────────────────────────────────────────────
[Family Enters Care Need — Free Text]
         │
         ▼
[Agent 1: Care Need Articulator]           [BUILT]
  Extracts structured care profile JSON
  → deal_breakers (hard / soft, affirmative phrasing)
  → must_haves, schedule, budget, required_skills
  → explicit_qualifications, inferred_qualifications
  → confidence score (0–1)
         │
  confidence ≥ 0.75?
  ┌───────┴────────┐
  Yes              No
  │                │
  ▼                ▼
[Agent 2]    [Surface low-confidence
              fallback — prompt for
              more context]
  │
  ▼
[Agent 2: Helper Matching and Ranking]    [BUILT]
  Step 1: Budget Gate
    → Exclude helpers above budget threshold (upper bound × 1.2)
  Step 2: Deal-Breaker Filter
    → Hard deal-breakers: hard exclusion from pool
    → Soft deal-breakers: logged as warnings, not exclusions
  Step 2.5: No-Match Detection
    → If no helpers pass deal-breakers: flag no-match
    → Flexible-schedule helpers scored at 2.0 for schedule fit
    → Confirmed daytime-only helpers scored 0.0–0.5 vs. night/weekend needs
  Step 3: Scoring (max composite 10.0)
    → Schedule Fit:         0–3.0 (proportion of required hours covered)
    → Care Experience:      0–3.0 (condition-specific depth)
    → Skills Match:         0–2.0 (explicit skill coverage)
    → Qualifications Match: 0–2.0 (relevant certifications)
  Step 4: Sort, then assign ranks
    → Write sorted list, then assign rank 1/2/3
  Output: top_3 ranked helpers with composite scores + reasoning
         │
         ▼
[Agent 3: Explanation Agent]              [BUILT]
  Takes: care profile + top 1–3 helpers
  Generates: 1–2 paragraph personalized explanation per helper
  Rules: every claim traceable to helper profile,
         warm non-clinical tone, mirrors family's own language
         │
         ▼
[Family Sees: Top Helpers + Explanations]
  → Contacts helper → Books → First Session
  → Re-booking signal → Platform ground truth


HELPER SIDE — PATH B (Profile Intelligence)    [PLANNED]
──────────────────────────────────────────────────────────────
[Trigger: Helper joins / updates profile / weekly batch]
         │
         ▼
[Agent 2: Helper Profile Intelligence]
  Scores profile quality: completeness, specificity,
  market fit (local search demand), trust signals
         │
  Score < 70? → Generate ranked improvement recommendations
         │
[Agent 3: Trust Signal Engine]
  Aggregates: re-booking rate, review sentiment,
  cancellation rate, response time, recency
  Generates: plain-language trust explanation
  Detects: anomalies → escalates to trust team
```

### AI User Experience Principles

- **For families:** The Need Articulator captures context in plain language — no form fields, no clinical vocabulary required. Match explanations read like a recommendation from a trusted friend who knows the helper personally: specific, grounded, and human.
- **Transparency:** Every claim in a match explanation is traceable to the helper's profile. Families can see the source behind any claim.
- **Graceful degradation:** If confidence is too low or no helpers meet the criteria, HAVEN surfaces honest fallback rather than serving a weak recommendation. An honest "we don't have the right match yet" is a better experience than a confident wrong one.
- **No algorithmic black box:** Trust scores are never shown as numbers to families. Only the human-language explanation is surfaced. Scores live in the internal audit layer.
- **Prohibited language in family-facing outputs:** "algorithm," "ranked," "score," "database," "AI," "system." Enforced via prompt constraint and Quality Validator.

---

## 🤖 Multi-Agent System Design

### Design Rationale

HAVEN is a multi-agent system because no single prompt can reliably perform all three tasks — structured extraction from emotional free text, multi-step elimination and scoring logic, and empathetic long-form explanation — without degrading performance on at least one. Separating these into specialized agents with typed JSON hand-offs allows each agent to be evaluated, iterated, and failed gracefully in isolation.

The key architectural insight is the **deal-breaker gate before scoring**. Scoring a helper on care experience and schedule fit is only meaningful if they have already cleared the family's non-negotiables. Conflating elimination and scoring into a single pass causes the model to partially satisfy deal-breakers rather than enforce them absolutely. The pipeline architecture makes this separation explicit and auditable.

### Agent Roster

| AGENT | RESPONSIBILITY | INPUT | OUTPUT | STATUS | MODEL |
|---|---|---|---|---|---|
| **Agent 1 — Care Need Articulator** | Converts family free-text to structured care profile JSON | Family description (free text) | Care profile JSON + confidence score | **Built** | Claude Haiku 4.5 |
| **Agent 2 — Helper Matching and Ranking** | Budget gate → deal-breaker filter → multi-dimensional scoring → ranked top 3 | Care profile JSON + helper pool JSON | Ranked top_3 with scores and reasoning | **Built** | Claude Sonnet 4.6 |
| **Agent 3 — Explanation Agent** | Generates personalized, grounded explanations for recommended helpers | Care profile + top 1–3 helper profiles | Match explanations (1–2 paragraphs per helper) | **Built** | Claude Sonnet 4.6 |
| **Agent 4 — Helper Profile Intelligence** `[PLANNED]` | Scores helper profile quality vs. local search demand; generates actionable improvement recommendations | Helper profile + booking history + local search query data | Profile quality score + ranked recommendations | Planned | GPT-4o mini (scoring) + GPT-4o (recommendations) |
| **Agent 5 — Trust Signal Engine** `[PLANNED]` | Aggregates behavioral signals into explainable trust score; flags anomalies | Booking history + review sentiment + cancellations + response times | Trust score + plain-language explanation + anomaly flags | Planned | GPT-4o mini |
| **Quality Validator** | LLM-as-Judge for all generated outputs: grounding, tone, sensitivity handling | Generated output + source data | Pass/fail + dimension scores + failure reason | **Built** (GPT-4.1 judge on Agent 3) | GPT-4.1 |

### Agent Communication Protocol

All inter-agent communication uses **typed JSON payloads** — no free-form text between agents. Every payload carries: `family_id`, `care_profile`, `confidence`, `flags` (model's self-reported concerns about ambiguity or missing data). Agent 2 output includes `excluded_by_deal_breaker` (list of excluded helpers with reason) and `step_reasoning` (budget gate, deal-breaker summary, schedule viability, scoring notes) for full auditability.

### Prompt Versioning

Every prompt change is logged in `evals/prompts/prompt_log.md` with: version, date, specific changes, rationale for each change, and before/after eval performance. This log is a core project artifact — it shows the thinking behind every prompt decision and constitutes the prompt iteration history from v1 through the final validated versions (Agent 1 v7, Agent 2 v9, Agent 3 v1).

---

## 🧰 Functional Requirements

| SECTION | USER STORY | EXPECTED BEHAVIOR |
|---|---|---|
| **Need Articulation** | As a family, I want to describe my care need in plain language without filling out a form | Agent 1 accepts free text and returns a structured care profile. No required fields at entry. |
| **Deal-Breaker Enforcement** | As a family, I want my non-negotiables treated as absolute exclusions, not as preferences | Hard deal-breakers eliminate helpers from the pool before scoring begins. A helper who triggers a hard deal-breaker never appears in the recommendation, regardless of their scores on other dimensions. |
| **Deal-Breaker Phrasing** | The system must correctly interpret deal-breakers to avoid false exclusions | Agent 1 phrases all deal-breakers affirmatively ("Helper must be a non-smoker") to prevent Agent 2 from misreading a negative statement. This was a documented failure mode in v5 (negative phrasing caused Agent 2 to exclude the correct helper). |
| **Budget Gate** | As a family, I want to only see helpers within my budget | Helpers whose hourly rate exceeds the family's budget upper bound × 1.2 are excluded at Step 1, before deal-breaker or scoring logic runs. |
| **Multi-Dimensional Scoring** | As a family, I want the recommendation to reflect my most important requirements, not just availability | Agent 2 scores each eligible helper across four dimensions with explicit weights (Schedule Fit 0–3.0, Care Experience 0–3.0, Skills Match 0–2.0, Qualifications Match 0–2.0). Composite score max = 10.0. |
| **Schedule Scoring Fairness** | Schedule coverage must be calculated proportionally, accounting for both start and end time | Neither start-time gaps nor end-time gaps are treated asymmetrically. Coverage = proportion of the family's required window that the helper covers, calculated as overlap / family_window_duration. |
| **No-Match Handling** | When no helper in the pool has experience in the family's specific care need, surface the best-available alternatives honestly | Agent 2 detects no-match scenarios. Agent 3 names the gap explicitly and explains why the recommended helpers are the best available — it does not fabricate a confident recommendation. |
| **Flexible Schedule Override** | When a family requires non-standard hours (nights, weekends) and no helper has confirmed those hours, helpers with flexible schedules should not be penalized equally to confirmed daytime-only helpers | Step 2.5 in Agent 2: flexible-schedule helpers score 2.0 for Schedule Fit in no-match schedule scenarios; confirmed daytime-only helpers score 0.0–0.5. |
| **Explanation Grounding** | Every claim in a match explanation must be traceable to the helper's profile data | Agent 3 is instructed: "Do not say anything about a helper that you cannot verify from their profile. If you cannot find grounding for a claim, omit it." LLM-as-Judge verifies grounding on every explanation. |
| **Explanation Tone** | Match explanations must be warm and appropriate for a family making a high-stakes care decision | Agent 3 mirrors the family's own language, acknowledges emotional stakes where present, and never uses clinical, transactional, or algorithmic language. |
| **Fewer-Than-Three Helpers** | The explanation pipeline must handle cases where fewer than 3 helpers survive deal-breaker filtering | Agent 3 returns one recommendation per helper provided. It does not invent helpers to fill three slots. |
| **Helper Profile Quality** `[PLANNED]` | As a helper, I want to know specifically what to improve in my profile to get more bookings | Agent 4 generates up to 3 ranked, specific, actionable improvement recommendations per helper per week. Each recommendation includes what to change, why it matters, and a suggested rewrite. |
| **Trust Score** `[PLANNED]` | As a family, I want to understand why a helper is trustworthy without seeing a number | Agent 5 generates a plain-language trust explanation (≤ 3 sentences) from behavioral signals. Explanation references specific verifiable facts. Never uses the word "algorithm," "score," or "ranked." |
| **Anomaly Detection** `[PLANNED]` | The system must flag helpers with anomalous trust signal patterns before surfacing them to families | Agent 5 flags: review velocity anomalies (> 5 reviews in 72 hours with < 3 bookings), no-show events in last 30 days, sudden availability reduction > 80%. Flagged helpers held at "Unverified" status. |
| **Privacy — Profile Deletion** | Families can delete their care profile at any time | Profile deleted within 24 hours of request. Behavioral data anonymized within 30 days. Agent 1 conversation transcripts deleted immediately after care profile extraction — not retained. |

---

## 📐 Model Requirements

Model selection was made against four evaluation criteria: task complexity, output sensitivity (care domain), latency requirements, and cost at scale. Models were evaluated across Anthropic, OpenAI, and Google families to ensure unbiased selection.

| AGENT | MODEL | RATIONALE | ALTERNATIVE |
|---|---|---|---|
| **Agent 1 — Care Need Articulator** | **Claude Haiku 4.5** | Structured extraction from free text — well-specified task that does not require frontier-level reasoning. Haiku 4.5's speed (< 1s per call) and cost efficiency are critical here: Agent 1 fires on every family onboarding, making it the highest-volume LLM call in the pipeline. Performance is equivalent to larger models on structured extraction with a well-designed prompt. | GPT-4o mini for equivalent cost profile; Claude Sonnet 4.6 if multi-turn conversation is added |
| **Agent 2 — Helper Matching and Ranking** | **Claude Sonnet 4.6** | This is the highest-reasoning task in the pipeline: sequential elimination logic (budget gate → deal-breaker filter → no-match detection), multi-dimensional scoring with explicit rubric, and sort-then-assign ranking. Requires frontier-level instruction following to correctly execute all steps in the correct order. Haiku 4.5 was evaluated and failed on complex deal-breaker reasoning — it collapsed elimination and scoring steps. | GPT-4o for comparable reasoning; Gemini 1.5 Pro viable but less consistent on structured JSON output |
| **Agent 3 — Explanation Agent** | **Claude Sonnet 4.6** | Long-form empathetic writing grounded in specific profile data — requires both grounding discipline (don't hallucinate) and tonal judgment (care-appropriate warmth). Claude Sonnet 4.6 consistently outperforms GPT-4o mini on tone calibration in sensitive contexts while maintaining factual discipline. | GPT-4o for comparable performance; avoid GPT-4o mini (tone tends toward transactional) |
| **Quality Validator / Judge** | **GPT-4.1** | LLM-as-Judge must use a different model family from the generator to prevent self-serving bias — non-negotiable. Since Agents 2 and 3 use Claude, GPT-4.1 provides independent evaluation. GPT-4.1 was selected over GPT-4o for superior structured reasoning on the four-dimensional judge rubric (Specificity, Relevance, Factual Grounding, Empathy Tone). | Claude Opus if Agents 2/3 were GPT-based |
| **Agent 4 — Profile Intelligence** `[PLANNED]` | GPT-4o mini (scoring) + GPT-4o (recommendations) | Profile scoring is structured classification — GPT-4o mini sufficient and cost-efficient at weekly batch scale. Recommendation generation requires nuanced, specific writing — GPT-4o ensures recommendations are genuinely actionable. | Agent scoring: Claude Haiku 4.5; recommendations: Claude Sonnet 4.6 |
| **Agent 5 — Trust Signal Engine** `[PLANNED]` | GPT-4o mini (signal aggregation + explanation) | Signal aggregation is rule-based math with LLM interpretation of review sentiment — GPT-4o mini well-suited and cost-efficient. Trust explanations are short (3 sentences) and templated. GPT-4o reserved for anomaly reasoning edge cases. | Claude Haiku 4.5 for equivalent performance |

### Latency Requirements

| PATH | TARGET |
|---|---|
| Agent 1 (per call) | < 2 seconds |
| Agent 2 (per matching request) | < 4 seconds |
| Agent 3 (per explanation set) | < 3 seconds |
| Agent 1 → 2 → 3 (full pipeline) | < 8 seconds end-to-end |
| Agent 4 + 5 (batch mode) `[PLANNED]` | Background — no user-facing latency constraint |

### Context Window Requirements

- **Agent 1:** 4K sufficient — single family description, no conversation history in prototype.
- **Agent 2:** 16K — must hold the full care profile + all helper profiles simultaneously to apply deal-breaker logic correctly. Scoring requires cross-helper comparison in a single pass.
- **Agent 3:** 8K — care profile + top 3 helper profiles.

---

## 🧮 Data Requirements

| REQUIREMENT | DETAILS | STATUS |
|---|---|---|
| **Family Profiles** | Free-text care descriptions → structured care profile JSON (care_type, schedule, budget, deal_breakers, must_haves, required_skills, qualifications, confidence). | **Built** — 17-family golden dataset (`data/families.json` + `data/golden_set.json`) |
| **Helper Profiles** | Per helper: bio, care types, certifications, years of experience, languages, schedule, hourly rate, personality signals. | **Built** — 20-helper synthetic pool (`data/helpers.json`) |
| **Golden Dataset — Matching** | 17 (family profile + helper pool → ideal ranked match) annotated triplets. Each annotated with: expected_top_3 (helper IDs + rank + rationale), acceptable_rank_1 (helper IDs that are defensible alternatives for rank 1), difficulty (clear / medium / hard / no_match), eval_notes, failure_flags. | **Built** — `data/golden_set.json` |
| **Eval Outputs** | Per pipeline run: Agent 1 outputs, Agent 2 outputs (with step_reasoning and excluded_by_deal_breaker), Agent 3 outputs. | **Built** — `evals/outputs/pipeline/` |
| **Judge Outputs** | Per Agent 3 explanation: GPT-4.1 evaluation across four dimensions with per-claim reasoning. | **Built** — `evals/outputs/judge/judge_outputs.json` |
| **Booking History** `[PLANNED]` | Per helper: booking ID, care type, duration, completion status (completed / cancelled / no-show), who initiated cancellation, advance notice. Used by Agent 5 for trust score computation. | Planned — requires Herewith internal API access |
| **Re-booking Data** `[PLANNED]` | Per family: did they re-book the same helper within 30 / 90 days? This is the primary ground truth signal for match quality evaluation and Agent 5 calibration. | Planned — 30-day delayed ground truth |
| **Local Search Query Data** `[PLANNED]` | Aggregated, anonymized search queries by geography — what care types, languages, and keywords families in each city are searching for. Used by Agent 4 for market fit scoring. | Planned — requires Herewith analytics access |
| **Privacy Constraint** | Care profiles must never contain raw medical diagnoses — only care need descriptors. Agent 1 transcripts deleted after extraction. Family IDs hashed in behavioral data stores. API calls to LLM providers anonymized (no real names or contact information). | Enforced in prompt design |

### Synthetic Dataset Design Principles

The 17-family golden dataset was designed to cover:
- **Difficulty distribution:** 7 clear, 4 medium, 3 hard, 3 no-match
- **Care type coverage:** Dementia / Alzheimer's, MS, post-surgical, companion care, pediatric, bi-cultural, end-of-life adjacent
- **Deal-breaker types:** Smoker, cultural background, age range, clinical manner, pet allergy, language requirement, companion care preference
- **Schedule complexity:** Standard weekday, non-standard (nights / weekends), flexible, no-match schedule scenarios
- **Budget scenarios:** Strict budget constraints, open budget, budget gates that eliminate most of the pool

---

## 💬 Prompt Requirements

All prompts are versioned and documented in `evals/prompts/prompt_log.md`. The final validated versions are:

- **Agent 1:** v7 (April 28, 2026)
- **Agent 2:** v9 (April 28, 2026)
- **Agent 3:** v1 (April 24, 2026 — no iteration required)

### Agent 1 — Care Need Articulator (v7)

1. Role definition must frame the agent as an expert care coordinator, not a chatbot or form processor.
2. Output schema must use `deal_breakers` as a typed object array: `{"text": "...", "type": "hard or soft"}`. The `text` field must be phrased affirmatively — what the helper **must be or must have**, not what they must not be. Negative phrasing ("Helper should not be a smoker") is prohibited because it creates double-negative ambiguity in Agent 2's exclusion logic.
3. Hard deal-breaker classification rules (all three must apply to classify as hard):
   - The family uses mandatory language ("must," "absolutely cannot," "non-negotiable") OR
   - The requirement is a safety baseline (e.g., no smoker in a respiratory care context), AND
   - There is past behavioral evidence that this requirement caused a care relationship to fail
4. Companion care soft deal-breaker rule: "If the care type is companion care or social/emotional support and the family mentions no clinical needs, classify clinical-care-only helpers as soft deal-breakers, not hard."
5. The prompt must include budget field with explicit parsing instruction: extract both the lower and upper bound if stated; if only one bound is stated, note which.

### Agent 2 — Helper Matching and Ranking (v9)

6. Step 0 (Budget Gate): Exclude helpers whose hourly rate exceeds `family_budget_upper_bound × 1.2`. Provide a worked example in the prompt so the model correctly applies the multiplier threshold — without an example, the model tends to exclude helpers exactly at the upper bound or apply no threshold at all.
7. Step 1 (Deal-Breaker Enforcement): Hard deal-breakers are absolute gates. Affirmatively phrased deal-breakers read as positive requirements — exclude any helper who demonstrably fails to meet them. Do not penalize helpers where confirmation is unclear; only exclude on confirmed violation.
8. Step 2 (Scoring Rubric):
   - Schedule Fit (0–3.0): proportion of the family's required window covered by the helper, calculated accounting for both start and end time. Neither start-time gaps nor end-time gaps are asymmetrically penalized.
   - Care Experience (0–3.0): depth and specificity of experience relevant to the family's primary condition.
   - Skills Match (0–2.0): explicit skill coverage against the family's required_skills list.
   - Qualifications Match (0–2.0): relevant certifications and formal training.
9. Step 2.5 (No-Match / Flexible Schedule Override): If the family requires non-standard hours and no helper has confirmed those hours, score flexible-schedule helpers at 2.0 for Schedule Fit; score confirmed daytime-only helpers at 0.0–0.5.
10. Step 4 (Sort-Then-Assign): Write the complete sorted list of helpers by composite score, then assign rank 1 / 2 / 3. This prevents rank-assignment errors caused by assigning ranks before the full ordering is established.

### Agent 3 — Explanation Agent (v1)

11. Role definition: "You are writing for a family making a high-stakes care decision for someone they love. Every word should feel like it was written by a thoughtful person, not generated by a system."
12. Grounding rule: "Do not say anything about a helper that you cannot verify from their profile. If you cannot find grounding for a claim, omit it — do not generalize, do not infer, do not approximate."
13. Tone rules: Mirror the family's own language from the care description. Acknowledge emotional stakes where present. Prohibited words: "algorithm," "scored," "ranked," "database," "system," "AI," "matches your criteria."
14. Variable length rule: Write one recommendation per helper provided. If only 1 or 2 helpers survive deal-breaker filtering, return 1 or 2 recommendations — do not invent helpers to fill 3 slots.

### Quality Validator Prompt (GPT-4.1 Judge — v1)

15. Four evaluation dimensions, each requiring reasoning-before-scoring:
    - **Specificity (1–5):** Are claims concrete and verifiable, or generic? A 5 requires every meaningful claim to cite a specific fact.
    - **Relevance (1–5):** Does the explanation address this specific family's stated needs? A 5 mirrors the family's own language and speaks to their stated concerns.
    - **Factual Grounding (Pass/Fail):** Every claim must be traceable to the helper's profile. Partial hallucination is not a 3 — it is a Fail. `failed_claims` must always be returned.
    - **Empathy Tone (1–5):** Is the tone appropriate for a care decision — warm and human, not clinical or transactional?
16. Process rule: "Complete reasoning before scoring. Never output a score without the reasoning that precedes it."

---

## 🧪 Eval Spec — Testing and Measurement

### Eval Design Principles

HAVEN's eval framework is designed around two constraints specific to care marketplace AI:

1. **Delayed ground truth:** The ultimate measure of match quality is 30-day re-booking — only observable 30+ days after the match is made. The eval framework therefore operates on both **proxy metrics** (immediately measurable via LLM-as-Judge and golden set) and **outcome metrics** (delayed but definitive, post-launch).

2. **False negatives are more dangerous than false positives:** A hallucinated care credential is not an accuracy error — it is a safety failure. The eval framework is calibrated conservatively: it is better to block a good output than to pass a bad one.

### Eval Spec — Agent 1 (Care Need Articulator)

| METRIC | THRESHOLD | EVALUATOR TYPE | RATIONALE |
|---|---|---|---|
| Profile Completeness | ≥ 85% of required dimensions populated on golden set | Code-Based: Field population check | Incomplete profiles produce weak matches — completeness is the floor of Agent 1's value |
| Confidence Calibration | ≥ 75% of Agent 1 confidence scores fall within annotated expected range on golden set | Code-Based: Range check vs. golden_set expected_agent1_confidence | Self-assessed confidence should correlate with actual profile richness — miscalibrated confidence breaks the Orchestrator's routing logic |
| Deal-Breaker Classification | Hard vs. soft classification matches annotator judgment on golden set | Human Eval: PM review of golden set | Misclassifying a hard deal-breaker as soft causes Agent 2 to fail to exclude an incompatible helper |
| Medical Language Avoidance | 0% of Agent 1 outputs contain a raw medical diagnosis | Code-Based: Regex check for diagnosis terminology | HIPAA-adjacent risk. Hard gate. |

### Eval Spec — Agent 2 (Helper Matching and Ranking)

| METRIC | THRESHOLD | EVALUATOR TYPE | RATIONALE |
|---|---|---|---|
| Rank-1 Match Accuracy | ≥ 65% on 17-family golden set | Code-Based: Compare actual rank-1 helper_id against expected and acceptable_rank_1 | If the best match isn't ranked first, the family may never reach them |
| Deal-Breaker Enforcement | 100% of hard deal-breaker violations correctly excluded | Code-Based + Human Review | A helper who violates a hard deal-breaker and appears in results is a P0 failure |
| Hallucination Rate | 0% — no helper recommended for a care type they have not demonstrated | LLM-as-Judge: Capability verification | Zero tolerance. A helper recommended for dementia care with no dementia experience is a harm, not an error. |
| Budget Gate Accuracy | 100% of helpers over threshold excluded | Code-Based: Rate check vs. threshold | Budget exclusion is a rule, not a preference — any violation is a defect |

### Eval Spec — Agent 3 (Explanation Agent)

| METRIC | THRESHOLD | EVALUATOR TYPE | RATIONALE |
|---|---|---|---|
| Specificity | ≥ 4.5/5 avg across golden set explanations | LLM-as-Judge (GPT-4.1) | Generic language produces explanations that could describe any helper — no trust-building value |
| Relevance | ≥ 4.5/5 avg | LLM-as-Judge (GPT-4.1) | An explanation that doesn't address the family's specific stated needs is a missed trust opportunity |
| Factual Grounding | ≥ 98% pass rate (≤ 1 failed claim per 50 explanations) | LLM-as-Judge (GPT-4.1) | A fabricated care credential in a care context is a P0 failure, not a quality issue |
| Empathy Tone | ≥ 4.5/5 avg | LLM-as-Judge (GPT-4.1) | Clinical or transactional tone in a care explanation reduces trust and conversion |

### Post-Launch Performance Tracking `[PLANNED]`

- **Weekly Match Quality Audit:** A random sample of 20 matches made in the past week is reviewed by a human care coordinator. Each rated on: care type alignment, explanation accuracy, "would you make this recommendation yourself?"
- **Monthly Trust Score Calibration Check:** Full Pearson correlation analysis of trust scores vs. 90-day re-booking rates for all helpers on the platform ≥ 90 days. If correlation drops below 0.65, Agent 5 signal weights are reviewed.
- **Failure Taxonomy:** Every booking that ends within 2 sessions is tagged by root cause: (a) care type mismatch, (b) personality mismatch, (c) schedule mismatch, (d) trust signal inaccurate, (e) other. Taxonomy drives next prompt iteration.
- **Quality Validator Calibration:** Judge pass/fail decisions validated against human reviewer judgments on a 50-item calibration set. Target: ≥ 88% agreement. Below 85% on any output type → judge prompt locked for review.

---

## 📈 Eval Results — Prototype

*These are the actual results from running the three-agent pipeline against the 17-family golden dataset with the final validated prompts (Agent 1 v7 + Agent 2 v9 + Agent 3 v1).*

### Agent 1 — Confidence Calibration

| RESULT | VALUE |
|---|---|
| Families evaluated | 17/17 |
| Confidence in annotated expected range | 13/17 (76%) |
| Notable gap | F007 (medium) and F009 (hard) — Agent 1 overconfident relative to ground truth difficulty |

### Agent 2 — Rank-1 Match Accuracy

| RESULT | VALUE |
|---|---|
| Families evaluated | 17/17 |
| Rank-1 correct (exact or acceptable) | **11/17 (65%)** |
| By difficulty: Clear | 5/7 (71%) |
| By difficulty: Medium | 3/4 (75%) |
| By difficulty: Hard | 2/3 (67%) |
| By difficulty: No-Match | 1/3 (33%) |

**Passing families:** F001, F003, F004, F006, F008, F009, F010, F012, F013, F014, F017

**Accepted failures (documented):**

| FAMILY | FAILURE REASON | ACCEPTED BECAUSE |
|---|---|---|
| F002 | Agent 2 ranks H003 over H002 (close tie in schedule + care experience) | Defensible ranking; both helpers are highly suitable |
| F005 | Agent 2 ranks H004 over H001 (tie in composite score) | Both are correct matches; tie-break not resolvable by rubric |
| F007 | Agent 2 ranks clinical helper over companion-care helper | Care type ambiguity: COPD + light chores creates genuine interpretation split |
| F011 | Agent 2 unable to correctly identify no-match scenario | Complex schedule + no-match intersection; documented as known architectural limit |
| F015 | Agent 2 ranks H019 over H015 (close score on hard case) | Hard case with limited helper pool — both reasonable |
| F016 | Agent 2 ranks H004 over H007 (companion care priority) | Scoring nuance in companion vs. personal care weighting |

### Agent 3 — Explanation Quality (GPT-4.1 Judge)

| DIMENSION | SCORE | NOTES |
|---|---|---|
| Specificity | **4.94/5** | 48/51 explanations scored 5/5 — Agent 3 consistently cites concrete, verifiable facts |
| Relevance | **4.59/5** | 31/51 scored 5/5, 19/51 scored 4/5 — minor gaps in addressing all stated concerns |
| Factual Grounding | **50/51 pass (98%)** | 1 failure: F016/H004 — fabricated social proof statement ("Families who've been through this often say..."). Flagged and documented. |
| Empathy Tone | **5.00/5** | Perfect across all 51 explanations — consistent warmth appropriate to care context |

### Prompt Iteration Summary

| VERSION | KEY CHANGE | RANK-1 ACCURACY |
|---|---|---|
| Agent 2 v6 | Relevance-strict qualifications scoring (baseline) | 6/17 (35%) |
| Agent 2 v7 | No-match detection + hard/soft deal-breaker typing | 8/17 (47%) |
| Agent 1 v6 + Agent 2 v8 | Cultural/demographic deal-breakers + schedule weight change | 7/17 (41%) — regression |
| **Agent 1 v7 + Agent 2 v9** | Past-behavior hard deal-breaker + companion care soft deal-breaker + proportion-based schedule scoring + budget gate example + flexible schedule override + sort-then-assign ranking | **11/17 (65%)** |

The regression in v8 was caused by a schedule weight change (Schedule Fit increased from 2.0 to 3.0) that caused helpers with equal care experience but marginally better schedule coverage to outrank helpers with stronger care experience. v9 reverted the weight change and fixed the root cause issues instead.

---

## ⚠️ Eval Gates — Risks and Mitigations

| RISK (FAILURE MODE) | SEVERITY | FREQUENCY ESTIMATE | MITIGATION |
|---|---|---|---|
| **Hallucinated Helper Credential** — Agent 2 or 3 recommends a helper for dementia care with no such experience | Critical: P0 | Low (with grounding directives) | Hard Gate: LLM-as-Judge capability grounding check on every explanation before it is served. Any unverifiable capability claim = output blocked. Fallback: remove ungrounded claim and regenerate. Zero tolerance. |
| **Medical Diagnosis in Care Profile** — Agent 1 stores or transmits a raw medical diagnosis stated by a family | Critical: P0 | Low-Medium (families volunteer diagnoses naturally) | Hard Gate: Regex + LLM check on Agent 1 output. Detected diagnosis → replaced with care need descriptor. Conversation transcript deleted post-extraction. |
| **Fabricated Trust Score for New Helper** `[PLANNED]` | Critical: P0 | Low (with booking count gate) | Hard Gate: Booking count check at Orchestrator level before Agent 5 is invoked. < 3 bookings = "Unverified" status returned unconditionally. Agent 5 not called. |
| **Affirmative Deal-Breaker Misinterpretation** — Agent 2 excludes the correct helper by misreading an affirmative deal-breaker | High: P1 | Low (mitigated in v7 by affirmative phrasing) | Agent 1 v7+ required to use affirmative phrasing for all deal-breakers. Validated in spot-checks post-deployment. Previously occurred in v5: "Not from South Asian background" caused Agent 2 to exclude South Asian helpers rather than non-South-Asian helpers. |
| **Sensitivity Mishandling by Agent 1** — Agent 1 responds to a grief-adjacent or end-of-life description with clinical tone | High: P1 | Medium (end-of-life care requests) | Agent 1 prompt includes sensitivity directive. Quality Validator tone check on all Agent 1 outputs. Clinical language in sensitive context → regenerate with warmth directive. |
| **Agent 3 Fabricates Social Proof** — Explanation invents testimonial-style language not traceable to a real data point | High: P1 | Low-Medium | LLM-as-Judge factual grounding check on every explanation. Observed once in prototype (F016/H004 — "Families who've been through this often say..."). Documented and flagged. |
| **Deal-Breaker Phrasing Causes False Exclusion** | High: P1 | Low (mitigated in v7) | Agent 1 affirmative phrasing rule. Agent 2 prompt: only exclude on confirmed violation, not on ambiguity. |
| **Over-Confident Match on Sparse Profile** — Agent 2 produces a top-3 when care profile confidence < 0.75 | Medium: P2 | Medium (families often don't complete articulation) | Orchestrator enforces confidence gate: Agent 2 not invoked if confidence < 0.75. Family prompted for context enrichment. |
| **Schedule Scoring Asymmetry** — End-time gaps penalized differently from start-time gaps, distorting ranking | Medium: P2 | Low (fixed in v9) | Proportion-based scoring rule enforces symmetric treatment. Explicit instruction: "neither start nor end time should be discounted." |
| **Agent 3 Parse Error on Small Helper Pool** — Fewer than 3 helpers survive deal-breaker filtering; Agent 3 template expects 3 slots | Medium: P2 | Low-Medium (no-match scenarios) | Agent 3 v1 includes variable-length rule: return one explanation per helper provided. Do not invent helpers. Validated on F017 (1 surviving helper). |

---

## 💰 AI Costs and Latency

### Prototype Operational Costs (Current)

At prototype scale (17 golden families, 20 helpers, single pipeline run):

| COMPONENT | AGENT | MODEL | CALL TYPE | APPROX COST |
|---|---|---|---|---|
| Care profile extraction | Agent 1 | Claude Haiku 4.5 | 17 families × ~1,500 tokens | < $0.02 |
| Helper matching + ranking | Agent 2 | Claude Sonnet 4.6 | 17 families × ~6,000 tokens (profile + full helper pool) | ~$0.30 |
| Explanation generation | Agent 3 | Claude Sonnet 4.6 | 17 families × ~3,000 tokens | ~$0.15 |
| LLM-as-Judge | GPT-4.1 | GPT-4.1 | 51 explanations × ~2,000 tokens | ~$0.20 |
| **Total per eval run** | — | — | — | **~$0.67** |

### Production Cost Estimate (At Scale — 1,000 Active Families, 500 Helpers)

| COMPONENT | AGENT | MODEL | VOLUME | COST/DAY |
|---|---|---|---|---|
| Care need articulation | Agent 1 | Claude Haiku 4.5 | 50 new/updated profiles × ~1,500 tokens | ~$0.06 |
| Helper matching + explanation | Agent 2 + 3 | Claude Sonnet 4.6 | 150 matching requests × ~8,000 tokens | ~$1.80 |
| LLM-as-Judge (quality gate) | Validator | GPT-4.1 | 150 × ~2,000 tokens | ~$0.45 |
| Profile re-scoring (weekly batch / 7) `[PLANNED]` | Agent 4 | GPT-4o mini + GPT-4o | ~70 helpers/day | ~$0.35 |
| Trust score refresh `[PLANNED]` | Agent 5 | GPT-4o mini | ~100 event triggers/day | ~$0.15 |
| **Total/day** | — | — | — | **~$2.81** |
| **Per family per day** | — | — | — | **~$0.003** |

At portfolio prototype scale: < $1/day.

### Latency Budget

| PATH | TARGET | NOTES |
|---|---|---|
| Agent 1 (per call) | < 2 seconds | Haiku 4.5 typically < 800ms |
| Agent 2 (per matching request) | < 4 seconds | Sonnet 4.6 with full helper pool — most latency-critical call |
| Agent 3 (per explanation set) | < 3 seconds | Sonnet 4.6, ~500 tokens output per explanation |
| **Full pipeline: family description → top-3 explanations** | **< 8 seconds** | Sequential; parallelizing Agent 1 → 2 possible with prefetch |
| Agents 4 + 5 (batch) `[PLANNED]` | No user-facing constraint | Background weekly batch |

---

## 🔗 Assumptions and Dependencies

**Prototype (validated):**
- Synthetic helper pool of 20 helpers is representative of the range of care experience, schedules, rates, and personality types found in a real marketplace helper pool.
- The 17-family golden dataset is annotated with sufficient difficulty distribution to stress-test the matching pipeline across clear, medium, hard, and no-match scenarios.
- GPT-4.1 judge scores are calibrated against the four-dimension rubric — calibration drift is possible across model updates and should be re-validated quarterly.

**Production (planned):**
- Herewith's booking history, review data, and re-booking data are accessible via internal API. This is the foundational behavioral signal for Agents 4 and 5 — HAVEN cannot compute trust scores without it.
- A minimum of 50 helpers with at least 5 completed bookings each are available for Agent 5 trust score calibration.
- Care coordinator input is available for golden dataset expansion (Agents 1 and 4 labeling requires domain expertise, not just product judgment).
- Local search query data available in anonymized, aggregated form for Agent 4 market fit scoring. If unavailable, Agent 4 falls back to national-level care demand patterns.
- The care taxonomy used by Agent 1 (care types, personality signal categories) is reviewed by a care domain expert before production launch — taxonomy errors propagate to all downstream agents.

---

## 🔒 Compliance / Privacy / Legal

| AREA | CONSIDERATION | MITIGATION |
|---|---|---|
| **HIPAA-Adjacent Risk** | Care need conversations include health-related information about care recipients (conditions, cognitive status, mobility limitations). Herewith is not a covered entity under HIPAA, but care data is sensitive and requires equivalent protection. | Agent 1 captures care need descriptors, not medical diagnoses. Conversation transcripts deleted after profile extraction. Care profiles use coded descriptors ("memory care support"), not diagnosis names (e.g., "Alzheimer's"). Legal review required before any clinically integrated use case expansion. |
| **Vulnerable User Protection** | Care recipients (elderly, disabled, children) are represented by proxy (the family) but are the actual subjects of the data and decisions. | Hard gate on hallucinated credentials — the most direct harm vector for vulnerable users. Sensitivity check on all family-facing content. Human escalation path for any anomaly involving a vulnerable care recipient. |
| **Background Check Data** | Herewith holds background check results for helpers. This is regulated data and must not be used as a raw input to Agent 5. | Agent 5 uses only behavioral platform data. Background check status is a binary platform eligibility gate — if failed, helper is not on the platform. Not a trust signal input. |
| **State-Level Care Licensing** | Some care types (medication administration, wound care, post-surgical support) require licensed professionals in most US states. Recommending an unlicensed helper for a licensed care type is a legal risk. | Care taxonomy includes a `licensing_required` flag per care type. Agent 2 filters out helpers lacking required certifications for licensing-required care types before ranking — this is a hard structured filter, not an LLM judgment. |
| **GDPR / CCPA** | Families and helpers are data subjects. Care profiles and behavioral data constitute personal data. | Right to deletion: care profile deleted within 24 hours of request; behavioral data anonymized within 30 days. Data minimization: Agent 1 transcripts deleted post-extraction. Agent 5 behavioral signals aggregated — raw booking records not retained in the HAVEN application layer. |
| **LLM API Data Sharing** | Care need content sent to Anthropic / OpenAI APIs constitutes third-party sharing of sensitive personal data. | Care profiles anonymized before API calls: no real names, addresses, or contact information. Family ID replaced with session hash. Disclosed at onboarding: "Your care preferences (not personal details) are processed by AI to find your best match." |
| **Algorithmic Bias** `[PLANNED]` | If historical booking data reflects societal biases (e.g., certain helper demographics receiving lower re-booking rates for reasons unrelated to care quality), Agent 5 trust scores could encode and amplify those biases. | Bias audit: trust score distribution analyzed across helper demographics before launch. If distribution is significantly skewed by demographic signals, signal weights are reviewed. Agent 4 recommendations audited for differential treatment across helper demographics. |

---

## 📣 GTM / Rollout Plan

### Phase 0 — Synthetic Prototype `[COMPLETE]`

Build the three-agent core matching pipeline (Agent 1 → Agent 2 → Agent 3) on a synthetic dataset of 17 families and 20 helpers. Validate end-to-end pipeline execution. Design and run the LLM-as-Judge eval framework. Establish baseline performance: **11/17 (65%) rank-1 accuracy, Agent 3 explanations averaging 4.94/5 specificity and 5.0/5 empathy tone.**

### Phase 1 — Golden Dataset Expansion + Prompt Hardening

Expand the golden dataset to 50 families across a wider distribution of care types and edge cases. Hire or engage a care coordinator for human annotation of 20 high-difficulty cases. Target prompt iteration to close the 6 remaining failures — prioritizing F011 (no-match schedule detection) and F016 (companion vs. personal care disambiguation). Add a conversational turn to Agent 1 to improve confidence calibration for medium/hard cases.

### Phase 2 — Agents 4 and 5 Build (Profile Intelligence + Trust Signal Engine)

Build the helper profile intelligence and trust signal systems. This phase requires access to real Herewith behavioral data — booking history, re-booking rates, review text, response times. Build the feedback loop: every re-booking event updates the trust score and enriches the golden dataset as a positive ground truth. Run calibration evaluation for Agent 5: target Pearson correlation ≥ 0.70 between trust score and 90-day re-booking rate on a held-out helper set.

### Phase 3 — Pilot Validation with Real Data

With appropriate privacy consent, run a structured pilot with 20–30 real Herewith families and 50–100 real helper profiles. Compare HAVEN match acceptance rates against Herewith's current baseline matching. Collect first-session feedback. Track 30-day re-booking signal. This is the phase where delayed ground truth evaluation begins — full results available at Phase 4.

### Phase 4 — Production Readiness

Wire the full four-agent pipeline to the Herewith matching flow. Implement the Orchestrator (confidence gate, cache, retry logic, fallback handling). Run the Quality Validator on all generated outputs before they are served. Deploy behind a feature flag to 5% of new family onboardings. Measure: first-match acceptance rate, browse-to-booking conversion, re-booking rate at 30 days.

### Phase 5 — Portfolio and Publication

- **GitHub:** Publish the full prototype with architecture documentation, PRD, eval framework, golden dataset structure, prompt log, and sample outputs with judge scores.
- **LinkedIn:** *"Why I built a three-agent AI system for care matching — and what it taught me about designing evals for a safety-critical domain."*
- **Demo:** Lovable frontend (free-text family description input → real-time AI matching → top 3 helper cards with Agent 3 explanations). Backend via n8n webhook with the locked final prompts (Agent 1 v7 → Agent 2 v9 → Agent 3 v1).

---

## 🏗️ Architecture Notes

### Key Design Decisions and Tradeoffs

| DECISION | ALTERNATIVE CONSIDERED | RATIONALE |
|---|---|---|
| Haiku 4.5 for Agent 1 | Sonnet 4.6 throughout | Agent 1 is a structured extraction task — Haiku 4.5 performs equivalently at ~6x lower cost and ~3x lower latency. Sonnet 4.6 budget reserved for reasoning-intensive matching. |
| Sequential pipeline (not parallel) | Parallel agent execution | Agent 2 requires Agent 1's structured output. Agent 3 requires Agent 2's ranked result. Dependencies make parallel execution impossible without staging. |
| Hard gate before scoring | Weighted soft constraints | Conflating deal-breakers with scoring weights causes the model to partially satisfy non-negotiables rather than enforcing them absolutely. The gate architecture makes the pipeline auditable — you can see exactly why a helper was excluded. |
| Affirmative deal-breaker phrasing | Negative phrasing ("Helper should not be X") | Negative phrasing creates double-negative ambiguity in Agent 2's exclusion logic. Documented failure in v5: "Not from South Asian background" caused Agent 2 to exclude South Asian helpers. Affirmative phrasing ("Helper must have South Asian cultural background") is unambiguous. |
| Sort-then-assign ranking | Assign ranks during scoring | Assigning ranks during the scoring pass causes errors when composite scores are close — the model makes local comparisons rather than global ones. Writing the full sorted list first, then assigning ranks, consistently produces correct orderings. |
| Proportion-based schedule scoring | Absolute hour-count scoring | Proportion-based scoring is scale-invariant — it treats a 3-hour coverage gap in a 4-hour window the same as a 6-hour gap in an 8-hour window. Absolute scoring penalizes longer required windows regardless of coverage quality. |
| GPT-4.1 as judge model | Claude as both generator and judge | Non-negotiable: LLM-as-Judge must use a different model family from the generator to prevent self-serving bias. Since Agents 2 and 3 use Claude Sonnet 4.6, GPT-4.1 provides genuinely independent evaluation. |

---

*Document Version: 2.0 | Prototype Eval Date: April 28, 2026 | Prompt Versions: Agent 1 v7, Agent 2 v9, Agent 3 v1*
*Related files: `evals/prompts/prompt_log.md` · `data/golden_set.json` · `evals/outputs/judge/judge_outputs.json` · `NEXT_STEPS.md`*
