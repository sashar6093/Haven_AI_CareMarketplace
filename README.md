# HAVEN — AI Care Marketplace Intelligence

> A three-agent AI pipeline that transforms a family's free-text care description into a ranked, explained helper recommendation — with a fully documented eval framework and real performance results.

**Built by:** Samveda Sharma | **Completed:** April 2026 | **Stack:** Python · Claude Haiku 4.5 · Claude Sonnet 4.6 · GPT-4.1 · Jupyter

---

## What This Project Demonstrates

HAVEN is a portfolio project built to show AI product management competency across the full build-and-eval loop:

| Skill | How It's Demonstrated |
|---|---|
| **AI product thinking** | Identified the care marketplace matching problem as an information asymmetry problem — not a search problem. Designed a pipeline architecture that enforces deal-breakers as hard gates before scoring begins. |
| **Multi-agent system design** | Three specialized agents with typed JSON hand-offs, each with a distinct reasoning task and failure mode. Documented why each agent boundary was drawn where it was. |
| **Prompt engineering** | 9 iterations on Agent 2, 7 on Agent 1, documented in a versioned prompt log with before/after scores and root cause analysis for every change — including a regression (v8 → v7 performance) that was diagnosed and fixed. |
| **LLM-as-Judge evaluation** | GPT-4.1 judge evaluating Agent 3 explanations across four dimensions (Specificity, Relevance, Factual Grounding, Empathy Tone) on 51 explanations. Separate model family from generator to prevent self-serving bias. |
| **Golden dataset design** | 17-family annotated evaluation set covering four difficulty levels (clear, medium, hard, no-match), with `expected_top_3`, `acceptable_rank_1`, `failure_flags`, and `eval_notes` per family. |
| **Failure analysis** | Every prompt change is justified by a specific failure. Every accepted failure is documented with why it was accepted rather than fixed. |
| **Model selection rationale** | Unbiased evaluation across Anthropic, OpenAI, and Google models — different models chosen per agent based on task type, not vendor preference. |

---

## The Problem

Care marketplace matching fails families in two ways:

1. **Families can't articulate their needs in structured form.** A mother's care description — "she's very proud and doesn't want to feel like a patient" — contains the most important matching signal, but no filter captures it.

2. **Platforms treat matching as keyword search.** Location + availability + rate doesn't surface the right helper. It surfaces the most available helper.

The result: families go through multiple helpers before finding someone who works. Each failed match is family churn, helper dropout, and a vulnerable person living through disruption.

**HAVEN's core insight:** In a care marketplace, the explanation is the product. A family doesn't book a helper because they scored 91/100. They book because they read *"Samuel has five years supporting younger adults with physical disabilities and is close to your brother in age"* and something shifts.

---

## How It Works

```
Family free-text description
         │
         ▼
┌─────────────────────────────┐
│  Agent 1 — Care Need        │  Claude Haiku 4.5
│  Articulator                │
│                             │
│  Extracts: deal_breakers    │
│  (hard/soft, affirmative),  │
│  must_haves, schedule,      │
│  budget, required_skills,   │
│  qualifications, confidence │
└─────────────┬───────────────┘
              │ Structured care profile JSON
              ▼
┌─────────────────────────────┐
│  Agent 2 — Helper Matching  │  Claude Sonnet 4.6
│  and Ranking                │
│                             │
│  Step 1: Budget gate        │
│  Step 2: Deal-breaker filter│  Hard → excluded
│  Step 2.5: No-match detect  │  Soft → warning
│  Step 3: Score each helper  │
│    Schedule Fit     0–3.0   │
│    Care Experience  0–3.0   │
│    Skills Match     0–2.0   │
│    Qualifications   0–2.0   │
│  Step 4: Sort → assign rank │
└─────────────┬───────────────┘
              │ top_3 with scores + reasoning
              ▼
┌─────────────────────────────┐
│  Agent 3 — Explanation      │  Claude Sonnet 4.6
│  Agent                      │
│                             │
│  Per helper: 1–2 paragraph  │
│  explanation grounded in    │
│  profile data, mirroring    │
│  the family's own language  │
└─────────────┬───────────────┘
              │
              ▼
     Top helpers + explanations
     served to the family
```

### Why Three Separate Agents?

No single prompt reliably performs all three tasks — structured extraction from emotional text, multi-step elimination and scoring logic, and empathetic long-form writing — without degrading on at least one. Separating them into specialized agents with typed JSON hand-offs means each agent can be evaluated, iterated, and failed gracefully in isolation.

The hard gate before scoring is the key architectural decision: eliminating incompatible helpers before scoring begins prevents the model from partially satisfying deal-breakers rather than enforcing them absolutely.

---

## Eval Results

### Agent 2 — Rank-1 Match Accuracy

| Metric | Result |
|---|---|
| Families evaluated | 17/17 |
| Rank-1 correct | **11/17 (65%)** |
| — Clear cases | 5/7 (71%) |
| — Medium cases | 3/4 (75%) |
| — Hard cases | 2/3 (67%) |
| — No-match cases | 1/3 (33%) |

### Agent 3 — Explanation Quality (GPT-4.1 Judge, 51 explanations)

| Dimension | Score |
|---|---|
| Specificity | **4.94 / 5.0** |
| Relevance | **4.59 / 5.0** |
| Factual Grounding | **50/51 pass (98%)** |
| Empathy Tone | **5.00 / 5.0** |

The one grounding failure (F016/H004): Agent 3 fabricated a social proof statement — "Families who have gone through this often say..." — with no source in the helper's profile. Flagged, documented, and included in the failure taxonomy.

### Prompt Iteration History

| Prompt Version | Key Change | Rank-1 Accuracy |
|---|---|---|
| Agent 2 v6 | Baseline | 6/17 (35%) |
| Agent 2 v7 | No-match detection + hard/soft deal-breaker typing | 8/17 (47%) |
| Agent 1 v6 + Agent 2 v8 | Cultural deal-breakers + schedule weight change | 7/17 (41%) — **regression** |
| **Agent 1 v7 + Agent 2 v9** | Affirmative phrasing + proportion-based scoring + flexible schedule override + sort-then-assign | **11/17 (65%)** |

The v8 regression is documented in the prompt log. The schedule weight increase (2.0 → 3.0) caused helpers with marginally better schedule coverage to outrank helpers with stronger care experience. v9 reverted the weight and fixed the root causes instead.

### If I Were Scaling This

Relevance scored lowest of the four judge dimensions (4.59/5) — the gap between specificity and relevance reveals that Agent 3 was consistently citing correct facts from the helper's profile but not always connecting them back to the specific language and concerns the family had used. A helper's "9 years of dementia care experience" is specific; *"9 years supporting families navigating exactly the kind of memory changes you described"* is relevant. That distinction is where the next iteration lives.

Two paths to close it at scale: expand the golden set to 50+ families and fine-tune on high-relevance explanation pairs; or add a lightweight verification pass — a secondary agent that checks each explanation against the family's original care description before it is served, flagging any explanation that doesn't mirror at least two of the family's own stated concerns.

---

## Key Design Decisions

### Deal-breakers as hard gates, not weights

The alternative was to encode deal-breakers as high-weight scoring dimensions. The problem: a weighted approach allows a helper to partially satisfy a non-negotiable and still rank highly. A family who specified "no smokers" (asthma) should never see a smoking helper ranked second because they had excellent dementia experience. The gate architecture makes this enforcement explicit and auditable — you can see exactly why every helper was excluded.

### Affirmative deal-breaker phrasing

A documented failure from v5: Agent 1 wrote `"Not from South Asian cultural background"` as a deal-breaker. Agent 2 read this as "helpers who ARE South Asian trigger this" and excluded the correct helper. The fix — requiring Agent 1 to always phrase deal-breakers as what the helper must be (`"Helper must have South Asian cultural background"`) — eliminated the double-negative ambiguity completely.

### Sort-then-assign ranking

Assigning ranks during the scoring pass causes Agent 2 to make local comparisons rather than global ones, producing ranking errors when composite scores are close. Requiring the model to write the full sorted list first, then assign rank 1/2/3, consistently produces correct orderings.

### Haiku 4.5 for Agent 1, Sonnet 4.6 for Agents 2 and 3

Agent 1 is structured extraction from a well-specified schema — Haiku 4.5 performs equivalently to Sonnet 4.6 on this task at 6x lower cost and 3x lower latency. The Sonnet budget is reserved for Agent 2's multi-step elimination reasoning (where Haiku 4.5 was evaluated and failed on complex deal-breaker sequences) and Agent 3's grounded empathetic writing.

### GPT-4.1 as judge, not Claude

LLM-as-Judge must use a different model family from the generator to prevent self-serving bias. Since Agents 2 and 3 use Claude Sonnet 4.6, GPT-4.1 provides genuinely independent evaluation. This is not a preference — it is a non-negotiable design requirement.

---

## Accepted Failures

Six families fail rank-1 matching. Each is accepted for a documented reason — not because fixing them was too hard, but because the right fix would require architectural changes beyond the scope of prompt iteration:

| Family | Agent 2 Result | Reason Accepted |
|---|---|---|
| F002 | H003 ranked over H002 | Close tie — both helpers are highly suitable |
| F005 | H004 ranked over H001 | Identical composite scores; tie-break not resolvable by rubric |
| F007 | Clinical helper ranked over companion-care helper | Genuine ambiguity: COPD + light chores creates a valid interpretation split |
| F011 | No-match scenario not fully detected | Complex schedule + no-match intersection; documented architectural limit |
| F015 | H019 ranked over H015 | Hard case with limited pool; both are reasonable |
| F016 | H004 ranked over H007 | Companion vs. personal care scoring nuance beyond rubric resolution |

---

## Repository Structure

```
haven/
├── README.md                          ← This file
├── PRD_HAVEN.md                       ← Full product requirements document (v2.0)
├── Project_Tradeoffs_and_Decisions.md ← Every non-obvious design decision with rationale
├── NEXT_STEPS.md                      ← Project checklist and status
│
├── data/
│   ├── families.json                  ← 17 synthetic family care descriptions
│   ├── helpers.json                   ← 20 synthetic helper profiles
│   └── golden_set.json                ← Annotated evaluation set (expected rankings,
│                                         acceptable alternatives, difficulty, eval notes)
│
└── evals/
    ├── 01_pipeline.ipynb              ← Agent 1 → 2 → 3 pipeline (run this first)
    ├── 02_eval_analysis.ipynb         ← Match accuracy + LLM-as-Judge eval (run second)
    │
    ├── prompts/
    │   └── prompt_log.md              ← Every prompt version with rationale,
    │                                     before/after scores, and failure analysis
    │
    └── outputs/
        ├── pipeline/
        │   ├── agent_1_outputs.json   ← Structured care profiles for all 17 families
        │   ├── agent_2_outputs.json   ← Ranked top-3 with scores and step reasoning
        │   └── agent_3_outputs.json   ← Match explanations for all 17 families
        └── judge/
            ├── match_accuracy.json    ← Rank-1 accuracy results (11/17)
            ├── judge_outputs.json     ← GPT-4.1 scores for all 51 explanations
            └── failure_taxonomy.json  ← Categorized failures (A–E taxonomy)
```

---

## Running the Project

### Prerequisites

```bash
# Clone the repo and navigate to the haven folder
cd haven

# Activate the virtual environment
source venv/bin/activate   # macOS/Linux
# or: venv\Scripts\activate  # Windows

# Set API keys
export ANTHROPIC_API_KEY="your-key-here"
export OPENAI_API_KEY="your-key-here"

# Launch Jupyter
jupyter notebook
```

### Step 1 — Run the Pipeline

Open `evals/01_pipeline.ipynb` and run all cells. This runs Agent 1 → Agent 2 → Agent 3 on all 17 families. Outputs are written to `evals/outputs/pipeline/`.

To inspect a single family, set `INSPECT_FAMILY = "F001"` (or any family ID) in the inspection cell and run it to see the full pipeline trace with Agent 1 care profile, Agent 2 step reasoning, and Agent 3 explanations.

### Step 2 — Run Evals

Open `evals/02_eval_analysis.ipynb` and run all cells. This:
- Compares Agent 2 rankings against the golden set
- Runs GPT-4.1 judge on all 51 Agent 3 explanations
- Outputs match accuracy, judge scores, and failure taxonomy

**Note:** The LLM-as-Judge section (Section 5) makes ~50 API calls to GPT-4.1. This costs approximately $0.20–0.30 per full run.

---

## The Broader HAVEN Vision

This repository implements the core matching intelligence pipeline. The full HAVEN product vision is a four-agent system:

| Agent | Status | Role |
|---|---|---|
| Agent 1 — Care Need Articulator | Built | Family free-text → structured care profile |
| Agent 2 — Helper Matching and Ranking | Built | Budget gate → deal-breaker filter → scoring → ranked top 3 |
| Agent 3 — Explanation Agent | Built | Grounded, empathetic match explanations |
| Agent 4 — Helper Profile Intelligence | Planned | Profile quality scoring + market-fit improvement recommendations |
| Agent 5 — Trust Signal Engine | Planned | Behavioral signal aggregation → explainable trust score + anomaly detection |

The full production PRD — including the four-agent architecture, all five eval specs, AI cost modeling, compliance/privacy considerations, and GTM rollout plan — is in `PRD_HAVEN.md`.

---

## About This Project

HAVEN was built as part of an AI Product Manager portfolio. The goal was not to build a perfect system — it was to build, evaluate, iterate, document failures honestly, and make defensible decisions about when to stop iterating and ship.

The 65% rank-1 accuracy is not the headline. The headline is that every failure is diagnosed, every prompt change has a before/after score, every design decision has an explicit rationale, and the system's behavior is auditable from the `step_reasoning` field in every Agent 2 output.

That is what production-grade AI product management looks like.

---

*Built by [Samveda Sharma](https://www.linkedin.com/in/samveda-sharma-23121544/) · April 2026*
