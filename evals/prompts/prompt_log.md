# HAVEN — Prompt Log
## All Agent Prompts · Every Iteration
**Project:** HAVEN — AI Care Marketplace Intelligence
**Author:** Samveda Sharma
**Last Updated:** April 28, 2026

> This file is the living record of every prompt used in the HAVEN pipeline — across all agents, across every iteration. Each entry documents: what the prompt was, why it was written this way, what it produced, and what changed in the next iteration. This log is a core eval artifact — it shows the thinking behind every prompt decision.

---

## How to Read This Log

- Each agent has its own section
- Each iteration is numbered (v1, v2, v3...)
- Every iteration documents: **the prompt → what it produced → what failed → what changed**
- The Judge prompt has its own section at the end

---

## Agent 1 — Care Need Articulator

**Job:** Convert a family's free-text care description into a structured care profile JSON that Agent 2 and Agent 3 can reason over.

**Model:** Claude Haiku 4.5

**Design rationale:** Haiku is sufficient here because this is a structured extraction task — the model does not need deep reasoning, it needs to reliably identify and categorize information already present in the text. Forcing a structured JSON output makes the downstream pipeline deterministic.

---

### v1 — Initial Prompt (April 3, 2026)

**Schema design decision — April 3, 2026:**
Added two qualification fields before the first run based on a product insight: families often do not know to ask for specific certifications even when their care situation implicitly requires them (e.g., a family describing dementia symptoms may not know to request dementia care training). Keeping explicit and inferred qualifications in separate fields allows the eval to measure inference accuracy independently of stated requirements.

**Prompt:**

```
You are an expert care coordinator helping families find the right care helper.

A family has described their care need below. Extract a structured care profile from their description.

Return ONLY a JSON object with these exact fields:
{
  "condition": "the primary condition or care need",
  "care_type": "type of care required",
  "schedule": "when care is needed",
  "household": "who lives in the home",
  "must_haves": ["list", "of", "non-negotiables"],
  "deal_breakers": ["list", "of", "dealbreakers"],
  "comfort_signals": ["things that would make the family feel at ease"],
  "explicit_qualifications": ["certifications or experience the family directly asked for — empty array if none mentioned"],
  "inferred_qualifications": ["certifications or experience not stated but strongly implied by the condition or care type — e.g. dementia care training if condition is dementia, first aid if care recipient lives alone"],
  "confidence": 0.0
}

Set confidence between 0 and 1 based on how clearly the family described their needs.
If the description is vague, set confidence below 0.75.

For inferred_qualifications: reason carefully from the condition and care context.
Do not invent qualifications that are not logically connected to the described situation.
If you cannot confidently infer any, return an empty array.

Family description:
"My mother is 78 and was recently diagnosed with early-stage dementia. She still lives
alone but we are worried about her safety. She needs someone to come on weekday mornings
to help with breakfast, light housekeeping and keep her company. She has a dog named
Biscuit she loves dearly. She is very private and would not be comfortable with someone
too young or too chatty. No smokers please."
```

**Why this prompt was written this way:**
- Role definition ("expert care coordinator") grounds the model's reasoning in the right domain
- "Return ONLY a JSON object" prevents the model from wrapping output in explanation text — which would break Agent 2's ability to parse it
- The output schema fields were designed in the PRD — every field maps to something Agent 2 and Agent 3 need for matching
- The confidence field signals to the orchestrator whether to proceed or flag for review
- `explicit_qualifications` captures what the family directly stated — verifiable against the input text
- `inferred_qualifications` captures what a trained care coordinator would know to add — this is where the model earns its value. Guarded with "do not invent" to prevent hallucination
- The family description is hardcoded for the first test — will be replaced with dynamic input once the node is validated

**Test input:** Hardcoded (mother, 78, early dementia, weekday mornings, no smokers)

**Expected inferred qualifications for this input:** dementia care experience or training, familiarity with elderly care safety protocols, ideally first aid certified (lives alone)

**Output produced:**
```json
{
  "condition": "Early-stage dementia",
  "care_type": "Companion care with activities of daily living assistance",
  "schedule": "Weekday mornings",
  "household": "78-year-old mother living alone with dog (Biscuit)",
  "must_haves": ["Comfortable with dogs", "Non-smoker", "Respectful of privacy", "Calm and quiet demeanor"],
  "deal_breakers": ["Smokers", "Very young caregivers", "Overly chatty or intrusive personality"],
  "comfort_signals": ["Mature, experienced caregiver", "Dog-friendly and willing to interact with pet", "Respectful listener rather than talker", "Reliable and consistent presence"],
  "explicit_qualifications": [],
  "inferred_qualifications": [
    "Dementia care experience or training",
    "Ability to recognize and respond appropriately to cognitive changes",
    "Basic understanding of safety considerations for someone with early-stage dementia",
    "Experience with meal preparation and light housekeeping"
  ],
  "confidence": 0.85
}
```

**What worked well:**
- `explicit_qualifications: []` — correctly empty, the family stated none
- `inferred_qualifications` — strong. Dementia care training and safety protocols correctly flagged
- `comfort_signals` — correctly inferred "respectful listener" from the family's implicit description of their mother's personality
- `confidence: 0.85` — appropriate given the description was fairly clear
- Dog's name captured in household — good detail pickup

**Failure modes observed:**
- `schedule: "Weekday mornings"` is vague — no start or end time. Downstream availability matching cannot use this. Needs a `clarification_needed` flag
- `"Experience with meal preparation and light housekeeping"` incorrectly placed in `inferred_qualifications` — this is a practical skill, not a formal certification. Mixing skills into qualifications dilutes Agent 2's ability to filter for trained helpers
- `comfort_signals` is redundant — on review, everything it captures belongs more precisely in `must_haves`, `required_skills`, or `inferred_qualifications`. The field adds noise without adding signal

---

### v2 — Revised Prompt (April 3, 2026)

**Reason for Revision:**
Three issues were identified from the v1 output:

1. **Vague schedule not flagged** — "Weekday mornings" was extracted as-is with no signal to downstream agents that the time window is unclear. Agent 2 cannot match helper availability without specific hours. Fix: add `clarification_needed` field so Agent 1 explicitly flags which fields need follow-up and what question to ask the family.

2. **Skills mixed into qualifications** — "Experience with meal preparation and light housekeeping" appeared in `inferred_qualifications`, which should only contain formal certifications or training. Practical skills needed for the job (cooking, cleaning, pet care) are a separate concept. Fix: add `required_skills` field to capture practical abilities, and tighten `inferred_qualifications` to certifications only.

3. **`comfort_signals` is redundant** — everything it captured belongs more precisely in `must_haves` (behavioral non-negotiables), `required_skills` (practical abilities), or `inferred_qualifications` (formal training). Keeping it creates overlap and ambiguity in the schema. Fix: remove `comfort_signals` entirely.

**Schema after v2 — what each field captures:**

| Field | Captures |
|---|---|
| `condition` | Primary care need or diagnosis |
| `care_type` | Type of care required (companion, medical, daily living etc.) |
| `schedule` | When care is needed — specific hours if mentioned |
| `household` | Who lives in the home |
| `must_haves` | Personality and behavioral non-negotiables |
| `deal_breakers` | Hard disqualifiers — things that rule a helper out entirely |
| `required_skills` | Practical abilities the job needs, stated or inferred (cooking, cleaning, pet care, mobility assistance etc.) |
| `explicit_qualifications` | Formal certifications or training the family directly asked for |
| `inferred_qualifications` | Formal certifications implied by the condition — not general skills |
| `clarification_needed` | Fields that are vague and need follow-up, with the specific question to ask |
| `confidence` | 0–1 score reflecting how clearly the family described their needs |

**Prompt:**

```
You are an expert care coordinator helping families find the right care helper.

Extract a structured care profile from the family description provided.

Return ONLY a JSON object with these exact fields:
{
  "condition": "the primary condition or care need",
  "care_type": "type of care required",
  "schedule": "when care is needed — include specific hours if mentioned",
  "household": "who lives in the home",
  "must_haves": ["personality and behavioral non-negotiables"],
  "deal_breakers": ["things that would immediately disqualify a helper"],
  "required_skills": ["practical abilities the job needs — e.g. cooking, light housekeeping, pet care, mobility assistance — whether stated or clearly implied"],
  "explicit_qualifications": ["formal certifications or training the family directly asked for — empty array if none mentioned"],
  "inferred_qualifications": ["formal certifications or training implied by the condition — e.g. dementia care certification, first aid training. Do NOT include general skills — only formal training or certifications"],
  "clarification_needed": [
    {
      "field": "the field name that is vague or incomplete",
      "reason": "why this field needs clarification",
      "question": "the specific follow-up question to ask the family"
    }
  ],
  "confidence": 0.0
}

Rules:
- Set confidence between 0 and 1. If the description is vague or key fields are unclear, set below 0.75.
- required_skills: include practical abilities needed for the job — cooking, cleaning, pet care, transfers, etc. These are skills, not certifications.
- inferred_qualifications: only formal training or certifications directly relevant to the stated condition. Never include general caregiving skills here.
- clarification_needed: flag any field where the information is too vague to be useful for matching. If all fields are clear, return an empty array.
```

**Test input:** Same hardcoded family description as v1

**Output produced:** *(to be filled after next run)*

**Failure modes observed:** *(to be filled after next run)*

**What will change in v3:** Restrict `clarification_needed` to only `schedule` and `care_type` fields — the two fields the system genuinely cannot match without. Flagging 4 clarification questions upfront overwhelms families who are already stressed. Only surface what is truly blocking the match.

---

### v3 — Revised Prompt (April 3, 2026)

**Reason for Revision:**
The v2 output flagged 4 clarification questions — schedule, care type, required skills, and household safety. While technically valid observations, asking a family 4 follow-up questions at the start of their care search creates friction and anxiety. This is a care marketplace, not a form. The system should only surface clarifications that are genuinely blocking the match from functioning.

Of the 4 flagged fields, only 2 are truly blocking:
- **Schedule** — without specific hours, helper availability matching is impossible
- **Care type** — without knowing the scope of personal care, we cannot assess whether a helper is qualified

The other two (household safety, housekeeping scope) are useful context but not blocking — the system can make a reasonable match without them and surface these questions later in the family-helper conversation.

Fix: restrict `clarification_needed` in the prompt to only flag `schedule` and `care_type`. All other fields should be extracted as best as possible from the available information.

**Schema — unchanged from v2**

| Field | Captures |
|---|---|
| `condition` | Primary care need or diagnosis |
| `care_type` | Type of care required (companion, medical, daily living etc.) |
| `schedule` | When care is needed — specific hours if mentioned |
| `household` | Who lives in the home |
| `must_haves` | Personality and behavioral non-negotiables |
| `deal_breakers` | Hard disqualifiers — things that rule a helper out entirely |
| `required_skills` | Practical abilities the job needs, stated or inferred |
| `explicit_qualifications` | Formal certifications the family directly asked for |
| `inferred_qualifications` | Formal certifications implied by the condition — not general skills |
| `clarification_needed` | Only schedule and care_type — if either is too vague to match on |
| `confidence` | 0–1 score reflecting how clearly the family described their needs |

**Prompt:**

```
You are an expert care coordinator helping families find the right care helper.

Extract a structured care profile from the family description provided.

Return ONLY a JSON object with these exact fields:
{
  "condition": "the primary condition or care need",
  "care_type": "type of care required",
  "schedule": "when care is needed — include specific hours if mentioned",
  "household": "who lives in the home",
  "must_haves": ["personality and behavioral non-negotiables"],
  "deal_breakers": ["things that would immediately disqualify a helper"],
  "required_skills": ["practical abilities the job needs — e.g. cooking, light housekeeping, pet care, mobility assistance — whether stated or clearly implied"],
  "explicit_qualifications": ["formal certifications or training the family directly asked for — empty array if none mentioned"],
  "inferred_qualifications": ["formal certifications or training implied by the condition — e.g. dementia care certification, first aid training. Do NOT include general skills — only formal training or certifications"],
  "clarification_needed": [
    {
      "field": "schedule or care_type only",
      "reason": "why this field is too vague to match on",
      "question": "the specific follow-up question to ask the family"
    }
  ],
  "confidence": 0.0
}

Rules:
- Set confidence between 0 and 1. If schedule or care_type is unclear, set below 0.75.
- required_skills: practical abilities only — cooking, cleaning, pet care, transfers etc. Not certifications.
- inferred_qualifications: only formal training or certifications directly relevant to the condition. Never general skills.
- clarification_needed: ONLY flag schedule or care_type if they are too vague to match on. Do not flag any other fields. If both are clear enough, return an empty array.
```

**Test input:** Same hardcoded family description as v1 and v2

**Output produced:**
```json
{
  "condition": "Early-stage dementia",
  "care_type": "In-home companion care with light housekeeping and meal support",
  "schedule": "Weekday mornings (specific hours not specified)",
  "household": "78-year-old mother living alone with dog (Biscuit)",
  "must_haves": ["Calm and patient demeanor", "Respectful of privacy and boundaries", "Mature in appearance and manner (not young)", "Quiet and not overly talkative", "Genuinely comfortable with dogs"],
  "deal_breakers": ["Smokers", "Overly chatty or intrusive personality", "Very young in appearance or manner"],
  "required_skills": ["Meal preparation (breakfast)", "Light housekeeping", "Dog care and handling", "Dementia-aware companionship"],
  "explicit_qualifications": [],
  "inferred_qualifications": ["Dementia care certification or training"],
  "clarification_needed": [
    {
      "field": "schedule",
      "reason": "Weekday mornings is clear, but specific hours are needed for accurate matching",
      "question": "What time should the caregiver arrive and depart each morning? (e.g., 8am-11am)"
    }
  ],
  "confidence": 0.82
}
```

**What improved from v2:**
- `clarification_needed` correctly flags only schedule — all non-blocking fields ignored as instructed
- `inferred_qualifications` tightened to dementia care certification only — first aid/CPR dropped, correct since it wasn't strongly implied
- `required_skills` clean — practical skills only, no certifications mixed in
- `confidence: 0.82` — appropriately higher than v2's 0.68, no longer penalised for non-blocking vagueness

**Failure modes observed:** None blocking. Agent 1 is working well enough to proceed to Agent 2.

**Decision:** Lock v3 as the working Agent 1 prompt. Further iteration will happen after the full pipeline is validated — over-engineering a single agent before Agent 2 and Agent 3 exist is the wrong priority.

**Status: SUPERSEDED by v4**

---

### v4 — Revised Prompt (April 3, 2026)

**Reason for Revision:**
Two missing fields identified that are critical for matching and cannot be inferred or skipped:

1. **Budget missing from schema** — a family with a $20/hr budget cannot be matched with a helper charging $35/hr regardless of care fit. Without budget, the matching logic is incomplete. Budget is typically a range, not a fixed number ("$18–$22/hr"). Added as a new field. Since families rarely state budget unprompted, added to `clarification_needed` if not mentioned — budget is as blocking as schedule for matching.

2. **`clarification_needed` scope tightened** — previously flagged schedule and care_type. Care type has been dropped because it can always be inferred from the care needs the family describes. `clarification_needed` is now strictly limited to **schedule** and **budget** — the only two fields the system genuinely cannot match without.

**Schema after v4:**

| Field | Captures |
|---|---|
| `condition` | Primary care need or diagnosis |
| `care_type` | Type of care required — inferred from description |
| `schedule` | When care is needed — specific hours if mentioned |
| `budget` | Family's hourly budget range if stated — null if not mentioned |
| `household` | Who lives in the home |
| `must_haves` | Personality and behavioral non-negotiables |
| `deal_breakers` | Hard disqualifiers |
| `required_skills` | Practical abilities the job needs |
| `explicit_qualifications` | Certifications the family directly asked for |
| `inferred_qualifications` | Certifications implied by the condition |
| `clarification_needed` | Schedule (if vague) and/or budget (if not stated) only |
| `confidence` | 0–1 score reflecting how clearly the family described their needs |

**Prompt:**

```
You are an expert care coordinator helping families find the right care helper.

Extract a structured care profile from the family description provided.

Return ONLY a JSON object with these exact fields:
{
  "condition": "the primary condition or care need",
  "care_type": "type of care required — infer from the description if not explicitly stated",
  "schedule": "when care is needed — include specific hours if mentioned",
  "budget": "hourly budget range if stated (e.g. '$18-$22/hr') — null if not mentioned",
  "household": "who lives in the home",
  "must_haves": ["personality and behavioral non-negotiables"],
  "deal_breakers": ["things that would immediately disqualify a helper"],
  "required_skills": ["practical abilities the job needs — e.g. cooking, light housekeeping, pet care, mobility assistance — whether stated or clearly implied"],
  "explicit_qualifications": ["formal certifications or training the family directly asked for — empty array if none mentioned"],
  "inferred_qualifications": ["formal certifications or training implied by the condition — e.g. dementia care certification, first aid training. Do NOT include general skills — only formal training or certifications"],
  "clarification_needed": [
    {
      "field": "schedule or budget only",
      "reason": "why this field is missing or too vague to match on",
      "question": "the specific follow-up question to ask the family"
    }
  ],
  "confidence": 0.0
}

Rules:
- Set confidence between 0 and 1. If schedule or budget is missing, set below 0.75.
- care_type: always infer from the description — never leave blank.
- budget: extract as a range if stated (e.g. "$18–$22/hr"). If not mentioned, set to null and flag in clarification_needed.
- required_skills: practical abilities only — cooking, cleaning, pet care, transfers etc. Not certifications.
- inferred_qualifications: only formal training or certifications directly relevant to the condition. Never general skills.
- clarification_needed: ONLY flag schedule (if vague or missing) or budget (if not stated). Do not flag any other fields.
```

**Test input:** Same hardcoded family description — budget not stated, so clarification_needed should flag both schedule and budget

**Expected output change from v3:** `budget: null` added, `clarification_needed` should now flag both schedule AND budget

**Output produced:**
```json
{
  "condition": "early-stage dementia",
  "care_type": "companion care with light assistance",
  "schedule": "weekday mornings — specific hours not stated",
  "budget": null,
  "household": "78-year-old mother living alone with dog (Biscuit)",
  "must_haves": ["patient and calm demeanor", "respectful of privacy and personal boundaries", "mature in appearance and communication style", "non-smoker", "comfortable with dogs"],
  "deal_breakers": ["smokers", "overly chatty or intrusive personality", "very young in appearance or demeanor"],
  "required_skills": ["meal preparation (breakfast)", "light housekeeping", "pet care (dog companionship)", "conversation and social engagement", "safety awareness"],
  "explicit_qualifications": [],
  "inferred_qualifications": ["dementia care certification or training", "first aid/CPR certification"],
  "clarification_needed": [
    {
      "field": "schedule",
      "reason": "Morning visits are mentioned but specific hours are not stated, making scheduling difficult",
      "question": "What time should the care helper arrive, and how many hours per morning visit does your mother need?"
    },
    {
      "field": "budget",
      "reason": "No hourly rate or budget range was mentioned",
      "question": "What is your budget range per hour for weekday morning care?"
    }
  ],
  "confidence": 0.68
}
```

**What worked:**
- `budget: null` correctly captured — not stated by family
- `clarification_needed` flags exactly schedule and budget — nothing else
- `inferred_qualifications` certifications only — dementia care + first aid/CPR
- `required_skills` clean with no certifications mixed in
- `confidence: 0.68` appropriate — two blocking fields missing
- Clarification questions specific and actionable

**Failure modes observed:** None. Output is correct across all dimensions.

**Status: SUPERSEDED by v6**

---

### v5 — Revised Prompt (April 25, 2026)

**Reason for Revision:**
Agent 2 v7 prompt iteration introduced a no-match detection step (STEP 2.5) that can restore helpers excluded by "soft" deal-breakers when no helper in the surviving pool covers the family's required schedule. To support this, Agent 2 needs to know which of Agent 1's deal-breakers are hard (non-negotiable) vs soft (inferred or preference-based). This classification must happen at extraction time — Agent 1 is closer to the source text and can make this call with more precision than Agent 2.

**What changed:**

1. `deal_breakers` field changed from a plain string array to an array of objects: `{"text": "...", "type": "hard | soft"}`

2. New classification rule added:
   - `"hard"`: family used explicit mandatory language ("required", "must", "need", "mandatory", "non-negotiable") OR it is a safety baseline (abuse history, violent behaviour, criminal record)
   - `"soft"`: family used preference language ("preferred", "good to have", "ideally", "would like") OR Agent 1 inferred it from the condition/context without the family stating it explicitly

**Design rationale:**
The hard/soft distinction must derive from the family's own language, not from a clinical judgment about the condition. This keeps Agent 1 as a text extraction agent — it reads what the family said and classifies accordingly. It does not make product decisions about whether a condition is "serious enough" to require hard experience gates. That judgment is encoded in the family's words.

**Schema after v5:**

| Field | Captures |
|---|---|
| `condition` | Primary care need or diagnosis |
| `care_type` | Type of care required — inferred from description |
| `schedule` | When care is needed — specific hours if mentioned |
| `budget` | Family's hourly budget range if stated — null if not mentioned |
| `household` | Who lives in the home |
| `must_haves` | Personality and behavioral non-negotiables |
| `deal_breakers` | Array of `{text, type}` objects — hard or soft classification |
| `required_skills` | Practical abilities the job needs |
| `explicit_qualifications` | Certifications the family directly asked for |
| `inferred_qualifications` | Certifications implied by the condition |
| `clarification_needed` | Schedule (if vague) and/or budget (if not stated) only |
| `confidence` | 0–1 score reflecting how clearly the family described their needs |

**Prompt:**

```
You are an expert care coordinator helping families find the right care helper.

Extract a structured care profile from the family description provided.

Return ONLY a JSON object with these exact fields:
{
  "condition": "the primary condition or care need",
  "care_type": "type of care required — infer from the description if not explicitly stated",
  "schedule": "when care is needed — include specific hours if mentioned",
  "budget": "hourly budget range if stated (e.g. '$18-$22/hr') — null if not mentioned",
  "household": "who lives in the home",
  "must_haves": ["personality and behavioral non-negotiables"],
  "deal_breakers": [
    {"text": "the disqualifying condition", "type": "hard or soft"}
  ],
  "required_skills": ["practical abilities the job needs — e.g. cooking, light housekeeping, pet care, mobility assistance — whether stated or clearly implied"],
  "explicit_qualifications": ["formal certifications or training the family directly asked for — empty array if none mentioned"],
  "inferred_qualifications": ["formal certifications or training implied by the condition — e.g. dementia care certification, first aid training. Do NOT include general skills — only formal training or certifications"],
  "clarification_needed": [
    {
      "field": "schedule or budget only",
      "reason": "why this field is missing or too vague to match on",
      "question": "the specific follow-up question to ask the family"
    }
  ],
  "confidence": 0.0
}

Rules:
- Set confidence between 0 and 1. If schedule or budget is missing, set below 0.75.
- care_type: always infer from the description — never leave blank.
- budget: extract as a range if stated (e.g. "$18–$22/hr"). If not mentioned, set to null and flag in clarification_needed.
- required_skills: practical abilities only — cooking, cleaning, pet care, transfers etc. Not certifications.
- inferred_qualifications: only formal training or certifications directly relevant to the condition. Never general skills.
- clarification_needed: ONLY flag schedule (if vague or missing) or budget (if not stated). Do not flag any other fields.
- deal_breakers type classification:
  - "hard": the family used explicit mandatory language ("required", "must", "need", "mandatory", "non-negotiable") OR it is a safety baseline (abuse history, violent behaviour, criminal record)
  - "soft": the family used preference language ("preferred", "good to have", "ideally", "would like") OR you inferred it from the condition or context without the family stating it explicitly
```

**Expected output change from v4:** `deal_breakers` field now returns objects instead of strings. For F011: "No experience or comfort with dementia care" should classify as `soft` (inferred from condition, family never stated experience as mandatory). "Unable to be present during overnight hours" should classify as `hard` (overnight availability is the family's explicit primary requirement).

**Output produced:** *(to be filled after pipeline re-run)*

**Failure modes observed:** *(to be filled after pipeline re-run)*

**Status: SUPERSEDED by v6**

---

### v6 — Revised Prompt (April 26, 2026)

**Reason for Revision:**
Two issues identified from v7 eval run analysis:

1. **F017 — Cultural/demographic requirement silently dropped**: Agent 1 completely failed to extract the family's explicit requirement for a South Asian helper. The requirement appeared nowhere in must_haves or deal_breakers. Root cause: the Rules section had no instruction covering cultural, demographic, language, or gender requirements. Fix: added explicit rule — capture these as must_haves (if preference language) or hard deal_breakers (if mandatory language).

2. **F009 — Over-inference from vague conditions**: The v5 fix (inferred_qualifications guidance) was in the schema comment but not tightly enough worded in the Rules section. For families describing mild/vague conditions ("sometimes forgets things", "aging in place"), Agent 1 should not infer clinical certifications. Fix: tightened the inferred_qualifications rule to require a clearly and specifically named condition before inferring certifications.

**What changed:**
- `inferred_qualifications` rule: now explicitly states "only infer formal certifications when the condition is clearly and specifically named. If vague or mild, leave inferred_qualifications empty."
- `deal_breakers` rule: expanded to explicitly cover cultural, demographic, language, and gender requirements with the same hard/soft classification logic.
- Schema comment for `inferred_qualifications` field updated to match.

**Status: SUPERSEDED by v7**

---

### v7 — Revised Prompt (April 28, 2026)

**Reason for Revision:**
Two extraction failures identified from v8 pipeline run analysis and golden set audit:

1. **F017 — South Asian preference mis-classified as soft**: The family described past negative experience where helpers from other cultures were let go because the grandmother could not adjust. This is demonstrated past behavior — equivalent to a mandatory constraint — but v6's hard deal-breaker rule only covered explicit mandatory language and safety baselines. Past dismissal of helpers is behavioral evidence that a requirement is hard, regardless of the words used. Fix: added "past demonstrated behavior counts as mandatory" to the hard classification rule.

2. **F007 — Clinical helper ranked #1 for a non-clinical companion care family**: The family's mother has COPD but manages it herself; she explicitly dislikes fuss and wants a calm, low-key companion for light chores. Agent 1 extracted no deal-breakers, so H003 (James Okonkwo, CNA with clinical framing) was never excluded — despite the family's must-haves clearly signaling a non-clinical presence. Fix: added a rule that when care_type is companion care and the family signals a non-fussy, low-key preference, Agent 1 extracts a soft deal-breaker for "overly clinical or medical-professional in manner."

**What changed:**

1. Hard deal-breaker rule extended: `"hard": ... OR the family describes past negative experience where helpers were dismissed or did not work out specifically because of this requirement — past demonstrated behavior counts as mandatory`

2. New rule appended: `When care_type is companion care AND the family explicitly signals they want a non-clinical, low-key, or unfussy presence (e.g. "dislikes fuss", "calm low-key", "does not want medical feel"), add a soft deal-breaker: "helper who presents as overly clinical or medical-professional in manner"`

**Expected impact:**
- F017: South Asian requirement now classified hard → non-South-Asian helpers excluded → H014 (Diana Patel, Gujarati/Hindi speaker) ranks #1
- F007: H003 (James Okonkwo, clinical CNA) excluded by soft deal-breaker → H004 (Rosa Mendez, companion specialist) ranks #1

**Output produced:** *(to be filled after pipeline re-run)*

**Failure modes observed:** *(to be filled after pipeline re-run)*

**Status: ACTIVE ✓**

---

## Agent 2 — Helper Profile Scorer

**Job:** Score every helper in the pool against the care profile from Agent 1. Return a ranked shortlist of the top 5 with composite scores and profile gaps identified.

**Model:** Claude Haiku 4.5

**Design rationale:** Scoring against a rubric is a structured reasoning task. Haiku handles this reliably at lower cost. The rubric is defined explicitly in the prompt so the model's scoring is deterministic and auditable.

---

### v1 — Initial Prompt (April 3, 2026)

**Scoring framework design decisions:**
- **Top 3 returned** — not 5. A family choosing care does not need to evaluate 5 candidates; 3 is the right decision surface
- **Scores out of 10, one decimal point** — a score of 8.5 vs 7.2 is explainable. A score of 85 vs 87 implies precision the model cannot genuinely justify
- **Budget hard gate at 20% flex** — helpers whose hourly rate exceeds the family's upper budget limit by more than 20% are excluded before scoring. The 20% buffer acknowledges that both sides may negotiate slightly. If family budget is null, gate is skipped entirely
- **No budget gate metadata in output** — internal logic, not relevant to downstream agents or the family
- **No other hard gates** — all other dimensions are scored, not binary. A helper without a certification but with 10 years of direct experience should still rank
- **Completeness penalty only when inference fails** — vague but inferable data is not penalised. Penalty only when data is genuinely absent and cannot be inferred
- **hourly_rate included in output** — shown to families in Agent 3's recommendation

**Scoring dimensions:**

| Dimension | Max score | What is scored |
|---|---|---|
| Schedule fit | 2.5 | Do helper's available days and hours overlap with the family's need? |
| Care experience | 3.0 | Does the helper have direct experience with the specific condition? |
| Skills match | 2.5 | How many required_skills from the care profile does the helper cover? |
| Qualifications match | 2.0 | Do they hold any explicit or inferred certifications? |
| Completeness penalty | −1.0 max | Only if a critical field is missing AND cannot be inferred |

**Composite score: out of 10.0**

**Prompt:**

```
You are a care matching specialist. Your job is to score a pool of
care helpers against a family's care profile and return the top 3
best-fit helpers.

You will be given:
1. A care profile JSON (the family's needs)
2. A list of helper profiles JSON

Follow these steps exactly:

STEP 1 — BUDGET GATE
Only apply if care_profile.budget is not null.
  - Parse the upper bound of the family's budget range
  - Calculate the exclusion threshold: upper_bound × 1.20
  - Silently exclude any helper whose hourly_rate exceeds this threshold
  - If care_profile.budget is null, skip this step entirely

STEP 2 — SCORE EACH REMAINING HELPER
Score each helper across these 4 dimensions. Use one decimal point only.

Schedule Fit (0–2.5):
  - 2.5: helper's days and hours fully cover the family's need
  - 1.5–2.4: partial overlap — minor gaps in days or hours
  - 0.5–1.4: limited overlap — significant gaps
  - 0–0.4: no overlap or schedule completely unspecified and cannot be inferred

Care Experience (0–3.0):
  - 2.5–3.0: direct, substantial experience with the exact condition stated
  - 1.5–2.4: some related experience or limited direct condition experience
  - 0.5–1.4: general elderly care only, no specific condition match
  - 0–0.4: no relevant experience evident

Skills Match (0–2.5):
  - Score proportionally based on how many required_skills from the
    care profile the helper demonstrably covers
  - Full coverage = 2.5, each clearly missing skill reduces proportionally

Qualifications Match (0–2.0):
  - 1.6–2.0: holds one or more of the inferred or explicit certifications
  - 0.8–1.5: holds related but not exact certifications
  - 0–0.7: no relevant certifications listed or inferable

Completeness Penalty (0 to −1.0):
  - Apply ONLY if a field critical to matching is missing AND cannot
    be reasonably inferred from the bio or other fields
  - Do not penalise vague language if the meaning can be inferred
  - Deduct 0.5 per genuinely unresolvable missing field, max −1.0

Composite score = sum of 4 dimension scores minus completeness penalty
Maximum possible score = 10.0

STEP 3 — RETURN TOP 3
Rank all scored helpers by composite score descending.
Return only the top 3.

Return ONLY a JSON object in this format:
{
  "top_3": [
    {
      "rank": 1,
      "helper_id": "...",
      "name": "...",
      "hourly_rate": N,
      "scores": {
        "schedule_fit": N.N,
        "care_experience": N.N,
        "skills_match": N.N,
        "qualifications_match": N.N,
        "completeness_penalty": N.N
      },
      "composite_score": N.N,
      "profile_gaps": ["list any fields that were missing or
                        unresolvable — empty array if none"],
      "score_reasoning": "2–3 sentences explaining why this helper
                          scored as they did against this specific
                          care profile"
    }
  ]
}
```

**Note:** `score_reasoning` is internal — used for debugging and eval only. Agent 3 writes the family-facing explanation separately.

**Test input:** Care profile from Agent 1 v4 output + all 20 helper profiles from helpers.json. Budget is null so budget gate will be skipped.

**Output produced:**
```json
{
  "top_3": [
    {
      "rank": 1,
      "helper_id": "H001",
      "name": "Margaret Chen",
      "hourly_rate": null,
      "scores": {
        "schedule_fit": 2.5,
        "care_experience": 3.0,
        "skills_match": 2.5,
        "qualifications_match": 2.0,
        "completeness_penalty": 0.0
      },
      "composite_score": 10.0,
      "profile_gaps": [],
      "score_reasoning": "Margaret is an exceptional match with 8 years of direct dementia-specific experience and all required certifications (CDP, CPR/First Aid). Her weekday schedule (7am–1pm) fully covers the stated morning need, and she demonstrates mastery across all required skills including meal prep, housekeeping, pet care, dementia-aware communication, and safety awareness."
    },
    {
      "rank": 2,
      "helper_id": "H004",
      "name": "Rosa Mendez",
      "hourly_rate": null,
      "scores": {
        "schedule_fit": 2.4,
        "care_experience": 3.0,
        "skills_match": 2.5,
        "qualifications_match": 2.0,
        "completeness_penalty": 0.0
      },
      "composite_score": 9.9,
      "profile_gaps": [],
      "score_reasoning": "Rosa brings 22 years of elder care experience with 10 years focused specifically on dementia clients. She holds advanced certifications (CDP, CADDCT) and explicitly demonstrates the calm, non-intrusive personality the family requires."
    },
    {
      "rank": 3,
      "helper_id": "H007",
      "name": "Donna Mitchell",
      "hourly_rate": null,
      "scores": {
        "schedule_fit": 2.4,
        "care_experience": 2.5,
        "skills_match": 2.3,
        "qualifications_match": 1.6,
        "completeness_penalty": 0.0
      },
      "composite_score": 8.8,
      "profile_gaps": [],
      "score_reasoning": "Donna offers 15 years of companion care with explicit focus on early-to-mid-stage dementia and deep respect for privacy and routine. While her certifications are foundational rather than practitioner-level, her demonstrated personality and stability make her a strong third choice."
    }
  ]
}
```

**What worked well:**
- Correct top 3 — H001, H004, H007 are the right matches for this care profile
- Budget gate correctly skipped — budget was null
- Score reasoning specific and grounded in profile facts, not generic
- Model correctly distinguished Dementia Friends training from practitioner-level CDP — reflected in Donna's lower qualifications score
- Completeness penalty correctly 0.0 for all three complete profiles

**Failure modes observed:**
- `hourly_rate: null` for all three helpers — model did not carry the hourly_rate through from the input data despite it being present in helpers.json. Prompt needs an explicit instruction to populate this field from the input

**What will change in v2:**
- Add explicit instruction: "Carry hourly_rate directly from the helper's profile data — do not leave null if it appears in the input"

**Status: SUPERSEDED by v2**

---

### v2 — Revised Prompt (April 6, 2026)

**Reason for Revision:**
`hourly_rate` returned as null for all helpers despite being present in the input data. The model did not know it was expected to carry this field through from input to output. A single explicit instruction fixes this — no other changes to the prompt needed.

**Change made:**
Added one rule at the end of the prompt:
`- hourly_rate: always populate directly from the helper's input profile. Never return null if the value exists in the input data.`

**Full prompt:** Same as v1 with this rule added to the Rules section at the end.

```
You are a care matching specialist. Your job is to score a pool of care helpers against a family's care profile and return the top 3 best-fit helpers.

You will be given:
1. A care profile JSON (the family's needs)
2. A list of helper profiles JSON

Follow these steps exactly:

STEP 1 — BUDGET GATE
Only apply if care_profile.budget is not null.
  - Parse the upper bound of the family's budget range
  - Calculate the exclusion threshold: upper_bound × 1.20
  - Silently exclude any helper whose hourly_rate exceeds this threshold
  - If care_profile.budget is null, skip this step entirely

STEP 2 — SCORE EACH REMAINING HELPER
Score each helper across these 4 dimensions. Use one decimal point only.

Schedule Fit (0–2.5):
  - 2.5: helper's days and hours fully cover the family's need
  - 1.5–2.4: partial overlap — minor gaps in days or hours
  - 0.5–1.4: limited overlap — significant gaps
  - 0–0.4: no overlap or schedule completely unspecified and cannot be inferred

Care Experience (0–3.0):
  - 2.5–3.0: direct, substantial experience with the exact condition stated
  - 1.5–2.4: some related experience or limited direct condition experience
  - 0.5–1.4: general elderly care only, no specific condition match
  - 0–0.4: no relevant experience evident

Skills Match (0–2.5):
  - Score proportionally based on how many required_skills from the care profile the helper demonstrably covers
  - Full coverage = 2.5, each clearly missing skill reduces proportionally

Qualifications Match (0–2.0):
  - 1.6–2.0: holds one or more of the inferred or explicit certifications
  - 0.8–1.5: holds related but not exact certifications
  - 0–0.7: no relevant certifications listed or inferable

Completeness Penalty (0 to −1.0):
  - Apply ONLY if a field critical to matching is missing AND cannot be reasonably inferred from the bio or other fields
  - Do not penalise vague language if the meaning can be inferred
  - Deduct 0.5 per genuinely unresolvable missing field, max −1.0

Composite score = sum of 4 dimension scores minus completeness penalty
Maximum possible score = 10.0

STEP 3 — RETURN TOP 3
Rank all scored helpers by composite score descending.
Return only the top 3.

Return ONLY a JSON object in this format:
{
  "top_3": [
    {
      "rank": 1,
      "helper_id": "...",
      "name": "...",
      "hourly_rate": N,
      "scores": {
        "schedule_fit": N.N,
        "care_experience": N.N,
        "skills_match": N.N,
        "qualifications_match": N.N,
        "completeness_penalty": N.N
      },
      "composite_score": N.N,
      "profile_gaps": ["list any fields that were missing or unresolvable — empty array if none"],
      "score_reasoning": "2–3 sentences explaining why this helper scored as they did against this specific care profile"
    }
  ]
}

Rules:
- hourly_rate: always populate directly from the helper's input profile. Never return null if the value exists in the input data.
```

**Output produced:**
```json
{
  "top_3": [
    {
      "rank": 1, "helper_id": "H004", "name": "Rosa Mendez", "hourly_rate": 30,
      "scores": {"schedule_fit": 2.5, "care_experience": 3.0, "skills_match": 2.5, "qualifications_match": 2.0},
      "composite_score": 10.0, "profile_gaps": [],
      "score_reasoning": "Rosa is an exceptional match. 22 years experience with 10 years specifically in dementia care, holds CDP and CADDCT certifications, covers all required skills. Quiet, boundary-respecting demeanor and love of animals directly address the family's must-haves."
    },
    {
      "rank": 2, "helper_id": "H001", "name": "Margaret Chen", "hourly_rate": 32,
      "scores": {"schedule_fit": 2.5, "care_experience": 3.0, "skills_match": 2.5, "qualifications_match": 2.0},
      "composite_score": 10.0, "profile_gaps": [],
      "score_reasoning": "Margaret matches Rosa in composite score. 18 years experience with 8 years focused on dementia. Covers all required skills, calm demeanor, non-smoker, dog owner. Ranked second due to slightly higher hourly rate ($32 vs. $30)."
    },
    {
      "rank": 3, "helper_id": "H010", "name": "Amara Osei", "hourly_rate": 26,
      "scores": {"schedule_fit": 2.5, "care_experience": 2.5, "skills_match": 2.5, "qualifications_match": 1.6},
      "composite_score": 9.1, "profile_gaps": [],
      "score_reasoning": "9 years home-based elderly care, 4 years direct dementia experience, Teepa Snow certification, all required skills covered, perfect schedule fit. Lower hourly rate ($26) makes her excellent value as third choice."
    }
  ]
}
```

**What worked:**
- Tiebreaker applied correctly and transparently — Margaret's reasoning explicitly states ranked second due to higher rate
- Completeness penalty removed — Amara scores 9.1 cleanly, skills_match corrected to 2.5
- H006 correctly excluded for dog allergy via model inference — but undocumented in prompt
- Score reasoning specific and grounded throughout

**Failure modes observed:**
- Deal-breaker exclusion (H006 dog allergy) happened via model inference with no explicit instruction — correct behaviour but undocumented and unreliable across different inputs

**What will change in v4:**
- Add explicit deal-breaker evaluation as a second layer between budget gate and scoring
- Use chain of thought reasoning per helper against family's deal_breakers
- Excluded helpers logged with clear reason before scoring begins

**Status: SUPERSEDED by v4**

---

### v4 — Revised Prompt (April 6, 2026)

**Reason for Revision:**
H006 (Samuel Park) was correctly excluded for having a dog allergy — a clear deal-breaker for a family whose care recipient has a dog. However, this exclusion happened via model inference with no explicit instruction in the prompt. This is unreliable — a different model, a different run, or a less obvious deal-breaker might not be caught.

Deal-breakers need to be a formal, documented evaluation step. The architectural question was where to place it:

**Why deal-breaker evaluation is the second layer (after budget gate, before scoring), not the last layer (after scoring):**

| Dimension | First Layer — before scoring | Last Layer — after scoring |
|---|---|---|
| Product logic | Correct — a deal-breaker is binary. No score makes an ineligible helper eligible | Incorrect — scoring effort wasted on helpers who will never be recommended |
| Processing | Faster — ineligible helpers eliminated before the expensive scoring step | Slower — all helpers scored regardless, some discarded after |
| Cost | Lower — fewer helpers scored = fewer tokens consumed. Compounds at scale | Higher — tokens spent scoring helpers who are thrown away |
| Debuggability | Cleaner — eliminated helpers logged with clear reason before scoring begins | Murkier — helper appears with high score then disappears, confusing in logs |
| User experience | No impact — families never see eliminated helpers either way | No impact — same final output |

**Decision: deal-breaker evaluation is the second layer. Deal-breakers are filters, not scores.**

**Updated pipeline:**
```
STEP 1 — BUDGET GATE           (hard number filter)
STEP 2 — DEAL-BREAKER CHECK    (chain of thought reasoning per helper)
STEP 3 — SCORE REMAINING       (only helpers who passed steps 1 and 2)
STEP 4 — RETURN TOP 3
```

**Prompt:**

```
You are a care matching specialist. Your job is to score a pool of care helpers against a family's care profile and return the top 3 best-fit helpers.

You will be given:
1. A care profile JSON (the family's needs)
2. A list of helper profiles JSON

Follow these steps exactly:

STEP 1 — BUDGET GATE
Only apply if care_profile.budget is not null.
  - Parse the upper bound of the family's budget range
  - Calculate the exclusion threshold: upper_bound × 1.20
  - Silently exclude any helper whose hourly_rate exceeds this threshold
  - If care_profile.budget is null, skip this step entirely

STEP 2 — DEAL-BREAKER CHECK
For each helper not excluded in Step 1, reason through the family's deal_breakers one by one.
Use chain of thought reasoning: explicitly consider each deal-breaker against what is known or inferable about the helper.
If any single deal-breaker is confirmed or strongly inferable from the helper's profile, exclude that helper entirely.
Log each excluded helper with the specific deal-breaker that caused exclusion.

Example reasoning:
  - Deal-breaker: "smokers" → check if helper mentions smoking. If not mentioned, assume non-smoker unless bio signals otherwise.
  - Deal-breaker: "overly chatty or intrusive personality" → check bio language for signals of communication style.
  - Deal-breaker: "very young in appearance or demeanor" → check age and how the helper describes themselves.
  - Deal-breaker: pet discomfort or allergy → check if helper mentions allergies or discomfort with animals.

Only exclude a helper if a deal-breaker is clearly confirmed or strongly inferable. Do not exclude on weak signals.

STEP 3 — SCORE EACH REMAINING HELPER
Score each helper across 4 dimensions. Use one decimal point only.
If data for a dimension is missing and cannot be inferred, score that dimension as 0 and note it in profile_gaps.

Schedule Fit (0–2.5):
  - 2.5: helper's days and hours fully cover the family's need
  - 1.5–2.4: partial overlap — minor gaps in days or hours
  - 0.5–1.4: limited overlap — significant gaps
  - 0.0: no overlap or schedule completely unspecified and cannot be inferred

Care Experience (0–3.0):
  - 2.5–3.0: direct, substantial experience with the exact condition stated
  - 1.5–2.4: some related experience or limited direct condition experience
  - 0.5–1.4: general elderly care only, no specific condition match
  - 0.0: no relevant experience evident or cannot be assessed

Skills Match (0–2.5):
  - Score proportionally based on how many required_skills from the care profile the helper demonstrably covers
  - Full coverage = 2.5, each clearly missing skill reduces proportionally
  - 0.0 if skills field is empty and cannot be inferred from bio

Qualifications Match (0–2.0):
  - 1.6–2.0: holds one or more of the inferred or explicit certifications
  - 0.8–1.5: holds related but not exact certifications
  - 0.0: no relevant certifications listed or inferable

Composite score = sum of 4 dimension scores. Maximum = 10.0.
Tie-breaking rule: if two helpers have equal composite scores, rank the one with the lower hourly_rate higher. If hourly rates are also equal, rank the one with more years_of_experience higher. If years of experience are also equal, either ranking is acceptable.

STEP 4 — RETURN TOP 3
Rank all scored helpers by composite score descending, applying tie-breaking rule where needed.
Return only the top 3.

Return ONLY a JSON object in this format:
{
  "excluded_by_deal_breaker": [
    {
      "helper_id": "...",
      "name": "...",
      "deal_breaker_triggered": "exact deal-breaker from care profile",
      "reasoning": "one sentence explaining the inference"
    }
  ],
  "top_3": [
    {
      "rank": 1,
      "helper_id": "...",
      "name": "...",
      "hourly_rate": N,
      "scores": {
        "schedule_fit": N.N,
        "care_experience": N.N,
        "skills_match": N.N,
        "qualifications_match": N.N
      },
      "composite_score": N.N,
      "profile_gaps": ["list any fields that were missing or unresolvable — empty array if none"],
      "score_reasoning": "2–3 sentences explaining why this helper scored as they did against this specific care profile"
    }
  ]
}

Rules:
- hourly_rate: always populate directly from the helper's input profile. Never return null if the value exists in the input data.
- Do not penalise helpers for missing fields that are irrelevant to this specific care profile.
- Only note a field in profile_gaps if it was missing, unresolvable, and caused a dimension to score 0.
- Only exclude a helper on a deal-breaker if it is clearly confirmed or strongly inferable — not on weak or ambiguous signals.
```

**Output produced:** (April 6, 2026)

```json
{
  "excluded_by_deal_breaker": [
    {
      "helper_id": "H006",
      "name": "Samuel Park",
      "deal_breaker_triggered": "comfortable with dogs",
      "reasoning": "Helper explicitly states 'I am allergic to dogs' in bio, making him incompatible with household that includes dog (Biscuit)."
    },
    {
      "helper_id": "H019",
      "name": "Ashley Johnson",
      "deal_breaker_triggered": "very young in appearance or demeanor",
      "reasoning": "Helper is 24 years old with no prior care experience and bio emphasizes 'very energetic and friendly' and 'eager to get into elder care,' signaling youth in demeanor and inexperience inappropriate for dementia companion care."
    }
  ],
  "top_3": [
    {
      "rank": 1,
      "helper_id": "H004",
      "name": "Rosa Mendez",
      "hourly_rate": 30,
      "scores": {
        "schedule_fit": 2.5,
        "care_experience": 3.0,
        "skills_match": 2.5,
        "qualifications_match": 2.0
      },
      "composite_score": 10.0,
      "profile_gaps": [],
      "score_reasoning": "Rosa is an exceptional match for early-stage dementia companion care. She brings 22 years of experience with 10 years specifically as primary caregiver for dementia clients, holds dual dementia certifications (CDP and CADDCT), and demonstrates all five required skills including pet care and cognitive engagement. Her weekday morning schedule (7:30am–12:30pm) fully covers the stated need, and her bio explicitly reflects the calm, non-intrusive, boundary-respecting demeanor the family prioritizes."
    },
    {
      "rank": 2,
      "helper_id": "H001",
      "name": "Margaret Chen",
      "hourly_rate": 32,
      "scores": {
        "schedule_fit": 2.5,
        "care_experience": 3.0,
        "skills_match": 2.5,
        "qualifications_match": 2.0
      },
      "composite_score": 10.0,
      "profile_gaps": [],
      "score_reasoning": "Margaret is equally strong as Rosa with 18 years of experience and 8 years focused on dementia and Alzheimer's care. She holds Certified Dementia Practitioner (CDP) certification and CPR/First Aid, covers all five required skills, and her weekday morning schedule (7:00am–1:00pm) fully aligns with family needs. Her bio emphasizes calm, patient listening and respect for privacy—key family must-haves. Ranked second due to slightly higher hourly rate ($32 vs. $30) per tie-breaking rule."
    },
    {
      "rank": 3,
      "helper_id": "H007",
      "name": "Donna Mitchell",
      "hourly_rate": 24,
      "scores": {
        "schedule_fit": 2.4,
        "care_experience": 2.0,
        "skills_match": 2.5,
        "qualifications_match": 1.6
      },
      "composite_score": 8.5,
      "profile_gaps": [],
      "score_reasoning": "Donna is a solid fit with 15 years of companion care experience and 3 years supporting early-dementia clients. She covers all five required skills including pet care (owns a dog herself), holds CPR/First Aid and HHA certifications, and demonstrates the quiet, calm, boundary-respecting demeanor the family values. Schedule (8:00am–1:00pm weekdays) has minor overlap gap compared to top two. Her lower hourly rate ($24) and longer tenure in companion care make her an excellent value option."
    }
  ]
}
```

**Failure modes observed:** None on this run. All four steps executed correctly:
- Budget gate correctly skipped (budget: null)
- H006 excluded via confirmed bio evidence (explicit dog allergy)
- H019 excluded via strong inference (age 24 + "eager to get into elder care" bio language)
- Tie-breaking applied correctly: Rosa and Margaret both scored 10.0; Rosa ranked #1 due to lower rate ($30 vs $32)
- All hourly_rate fields populated correctly
- score_reasoning is specific and grounded in actual helper bio facts throughout
- excluded_by_deal_breaker retained in output as internal audit log — not passed to Agent 3

**What will change in v5:** Tie-breaking rule language to be made unambiguous — model was applying "rank higher" as rank 2 instead of rank 1.

**Status: LOCKED ✓**

---

### v5 — Revised Prompt (April 11, 2026)

**Reason for Revision:**
The tie-breaking rule in v4 said "rank the one with the lower hourly_rate higher." The model consistently interpreted "higher" as a higher rank number (rank 2) rather than a better position (rank 1). Rosa ($30/hr) and Margaret ($32/hr) both scored 10.0, but Margaret was ranked first across multiple runs — the opposite of the intended behaviour.

A secondary issue was identified: the tie-breaking rule was also ambiguous for ties at positions other than rank 1. If two helpers tied at rank 3, saying "rank #1" would be incorrect. The language was updated to "ranked first among them" — meaning first within the tied group, not first overall.

**Change:** Tie-breaking rule only. All other prompt content unchanged.

**Updated tie-breaking rule:**

```
Tie-breaking rule: if two helpers have equal composite scores, the one with the LOWER hourly_rate must appear earlier in the ranked list. A lower price is better — rank it above a higher price. If hourly rates are also equal, the one with MORE years_of_experience appears earlier. If years of experience are also equal, either ordering is acceptable.
```

**Scoring stability note (not a prompt issue):**
Across multiple runs, Rosa's `schedule_fit` score fluctuated between 2.4 and 2.5. Her schedule starts at 7:30am; the family stated "weekday mornings" without specifying exact start time. The model inconsistently treats this as full coverage (2.5) or a minor gap (2.4). When Rosa scores 2.4, her composite drops to 9.9, she is no longer tied with Margaret, and tie-breaking does not apply — Margaret ranks first on composite score alone. This is LLM non-determinism, not a prompt failure. Setting temperature to 0 would reduce but not eliminate this. Decision: leave temperature unchanged for now and treat scoring variation as an eval finding to document in the failure taxonomy.

**Status: SUPERSEDED by v6**

---

### v6 — Revised Prompt (April 20, 2026)

**Reason for Revision:**
The Qualifications Match dimension was rewarding volume of certifications rather than relevance to the specific care profile. A helper with CDP + CADDCT + CHPNA for a companion care family scored 2.0 — the same as a helper with a single perfectly relevant certification. Worse, the model was finding ways to call unrelated certifications "related" when a helper had many of them, inflating scores for over-qualified helpers in cases where their qualifications added no value.

Two specific failure modes were identified:

1. **Volume bias** — a helper with 5 certifications, none directly relevant to the care profile, could score 0.8–1.5 on the "related but not exact" band simply because the model found loose domain overlap. This is not meaningful signal.

2. **Fixed weights across care types** — the formula weights Qualifications Match at 2.0 max regardless of care type. For companion care families, qualifications are nearly irrelevant. For post-surgical or dementia families, they are critical. A flat weight treats both identically, which distorts rankings. This is a known architectural limitation — a full dynamic-weight solution (where care type adjusts dimension maxima) is the right long-term fix but requires care_type classification logic to be added upstream. Documenting here for v7.

**Change made:** Qualifications Match dimension rewritten to be strictly anchored to the care profile's `explicit_qualifications` and `inferred_qualifications` fields only. Volume rule added explicitly. All other prompt content unchanged from v5.

**Updated Qualifications Match dimension:**

```
Qualifications Match (0–2.0):
Score ONLY against the care profile's explicit_qualifications and
inferred_qualifications fields. Do not reward certifications that
do not appear in those fields, regardless of how many the helper holds.

  - 1.8–2.0: holds one or more certifications that exactly match
             explicit_qualifications or inferred_qualifications
  - 1.0–1.7: holds a certification with partial overlap — same
             domain as a required qualification, but not an exact match
  - 0.0: no certifications matching explicit or inferred
         qualifications — score 0 regardless of total certifications held

Volume rule: 5 relevant certifications scores no higher than 1 relevant
certification. Only relevance to this care profile matters, not count.
```

**Full prompt:**

```
You are a care matching specialist. Your job is to score a pool of care helpers against a family's care profile and return the top 3 best-fit helpers.

You will be given:
1. A care profile JSON (the family's needs)
2. A list of helper profiles JSON

Follow these steps exactly:

STEP 1 — BUDGET GATE
Only apply if care_profile.budget is not null.
  - Parse the upper bound of the family's budget range
  - Calculate the exclusion threshold: upper_bound × 1.20
  - Silently exclude any helper whose hourly_rate exceeds this threshold
  - If care_profile.budget is null, skip this step entirely

STEP 2 — DEAL-BREAKER CHECK
For each helper not excluded in Step 1, reason through the family's deal_breakers one by one.
Use chain of thought reasoning: explicitly consider each deal-breaker against what is known or inferable about the helper.
If any single deal-breaker is confirmed or strongly inferable from the helper's profile, exclude that helper entirely.
Log each excluded helper with the specific deal-breaker that caused exclusion.

Example reasoning:
  - Deal-breaker: "smokers" → check if helper mentions smoking. If not mentioned, assume non-smoker unless bio signals otherwise.
  - Deal-breaker: "overly chatty or intrusive personality" → check bio language for signals of communication style.
  - Deal-breaker: "very young in appearance or demeanor" → check age and how the helper describes themselves.
  - Deal-breaker: pet discomfort or allergy → check if helper mentions allergies or discomfort with animals.

Only exclude a helper if a deal-breaker is clearly confirmed or strongly inferable. Do not exclude on weak signals.

STEP 3 — SCORE EACH REMAINING HELPER
Score each helper across 4 dimensions. Use one decimal point only.
If data for a dimension is missing and cannot be inferred, score that dimension as 0 and note it in profile_gaps.

Schedule Fit (0–2.5):
  - 2.5: helper's days and hours fully cover the family's need
  - 1.5–2.4: partial overlap — minor gaps in days or hours
  - 0.5–1.4: limited overlap — significant gaps
  - 0.0: no overlap or schedule completely unspecified and cannot be inferred

Care Experience (0–3.0):
  - 2.5–3.0: direct, substantial experience with the exact condition stated
  - 1.5–2.4: some related experience or limited direct condition experience
  - 0.5–1.4: general elderly care only, no specific condition match
  - 0.0: no relevant experience evident or cannot be assessed

Skills Match (0–2.5):
  - Score proportionally based on how many required_skills from the care profile the helper demonstrably covers
  - Full coverage = 2.5, each clearly missing skill reduces proportionally
  - 0.0 if skills field is empty and cannot be inferred from bio

Qualifications Match (0–2.0):
Score ONLY against the care profile's explicit_qualifications and
inferred_qualifications fields. Do not reward certifications that
do not appear in those fields, regardless of how many the helper holds.
  - 1.8–2.0: holds one or more certifications that exactly match
             explicit_qualifications or inferred_qualifications
  - 1.0–1.7: holds a certification with partial overlap — same
             domain as a required qualification, but not an exact match
  - 0.0: no certifications matching explicit or inferred
         qualifications — score 0 regardless of total certifications held

Volume rule: 5 relevant certifications scores no higher than 1 relevant
certification. Only relevance to this care profile matters, not count.

Composite score = sum of 4 dimension scores. Maximum = 10.0.
Tie-breaking rule: if two helpers have equal composite scores, the one with the LOWER hourly_rate must appear earlier in the ranked list. A lower price is better — rank it above a higher price. If hourly rates are also equal, the one with MORE years_of_experience appears earlier. If years of experience are also equal, either ordering is acceptable.

STEP 4 — RETURN TOP 3
Rank all scored helpers by composite score descending, applying tie-breaking rule where needed.
Return only the top 3.

Return ONLY a JSON object in this format:
{
  "excluded_by_deal_breaker": [
    {
      "helper_id": "...",
      "name": "...",
      "deal_breaker_triggered": "exact deal-breaker from care profile",
      "reasoning": "one sentence explaining the inference"
    }
  ],
  "top_3": [
    {
      "rank": 1,
      "helper_id": "...",
      "name": "...",
      "hourly_rate": N,
      "scores": {
        "schedule_fit": N.N,
        "care_experience": N.N,
        "skills_match": N.N,
        "qualifications_match": N.N
      },
      "composite_score": N.N,
      "profile_gaps": ["list any fields that were missing or unresolvable — empty array if none"],
      "score_reasoning": "2–3 sentences explaining why this helper scored as they did against this specific care profile"
    }
  ]
}

Rules:
- hourly_rate: always populate directly from the helper's input profile. Never return null if the value exists in the input data.
- Do not penalise helpers for missing fields that are irrelevant to this specific care profile.
- Only note a field in profile_gaps if it was missing, unresolvable, and caused a dimension to score 0.
- Only exclude a helper on a deal-breaker if it is clearly confirmed or strongly inferable — not on weak or ambiguous signals.
```

**What this fixes:**
- A helper with CDP + CADDCT + CHPNA for a companion care family (where inferred_qualifications = []) now scores 0.0 on qualifications, not 1.5+
- A helper with a single HHA cert for a family whose inferred_qualifications includes HHA scores 1.8–2.0 — the same as a helper with five certs who also holds HHA
- Model can no longer inflate scores by finding loose "domain overlap" across irrelevant certifications

**Known limitation carried forward to v7:**
Qualifications Match is still weighted at a flat 2.0 max regardless of care type. For companion care families (where inferred_qualifications = []), the entire dimension collapses to 0 for all helpers — which is correct behaviour, but it also means the composite max effectively drops to 8.0 for those families, making cross-family score comparisons misleading. Dynamic dimension weights based on care_type is the architectural fix. Flagged for v7.

**Output produced:** (April 24, 2026) — full pipeline run on all 17 families

**Match accuracy results:**
- Rank-1 exact match: 6/17 (35%)
- Full set match (all 3 correct): 2/17 (12%)
- Average overlap: 1.8/3 helpers correct per family

**Families with rank-1 miss:** F001, F002, F005, F007, F009, F011, F013, F015, F016, F017

**Notable observations:**
- Many rank-1 failures are rank-order errors within a near-correct set — the right helpers are present but in the wrong order. This points to scoring weight calibration rather than a fundamental matching failure.
- F011 and F012 are no_match cases where Agent 2 returned strong helpers instead of weak ones — the agent has no awareness of the no_match scenario.
- F005 returned 4 helpers instead of 3 — a prompt compliance failure (model added a duplicate H001 entry).
- F013 (progressive MS, 34-year-old) required extended thinking (budget_tokens: 10000) because the complexity of scoring 15 remaining helpers after deal-breaker exclusions exceeded the standard output token budget.

**Known limitation carried forward to v7:**
Qualifications Match is still weighted at a flat 2.0 max regardless of care type. For companion care families (inferred_qualifications = []), the entire dimension collapses to 0.0 for all helpers — effectively reducing the composite max to 8.0 and making cross-family score comparisons misleading. Dynamic dimension weights based on care_type is the architectural fix.

**Status: SUPERSEDED by v8**

---

### v7 — Revised Prompt (April 25, 2026)

**Reason for Revision:**
Failure taxonomy from the April 24 eval run identified 10/17 wrong rank-1 failures (Type E). Root cause analysis across the failing families surfaced three distinct failure modes:

- **Type A — Tie-break failures (e.g. F001):** H004 and H001 both scored 10.0; no tiebreaker resolved the tie consistently. The existing hourly_rate tiebreaker was present in v6 but the failure suggests it was not applied reliably.
- **Type B — Care Experience overweighted vs Schedule Fit (e.g. F002):** Model ranked the helper with higher care experience (3.0) over the helper with better schedule fit (2.0) even when schedule was the harder constraint for that family.
- **Type C — Deal-breaker eliminates golden candidates (F011 — no-match scenario):** H015 (golden rank 1) and H018 (golden rank 3) were excluded by "No experience or comfort with dementia care" — a deal-breaker Agent 1 *inferred* from the condition. The family never stated experience as a requirement. None of the surviving helpers covered the 8pm–8am overnight window.

This iteration addresses **Type C** first as the highest-impact fix. Type A and B (scoring calibration) will follow in v8 once the no-match logic is validated.

**What changed:**

1. **STEP 2.5 — SCHEDULE VIABILITY CHECK added** between Step 2 and Step 3:
   - Detects non-standard schedule requirements (any window 6pm–9am, or weekend-only)
   - If zero surviving helpers cover those hours → no-match scenario
   - In no-match: restore helpers excluded solely by `soft` deal-breakers; keep `hard` deal-breaker exclusions intact
   - Set `"no_match": true` in output; log which soft deal-breakers were suspended in `step_reasoning.schedule_viability`

2. **`no_match` field added** to output JSON (top-level, defaults to `false`)

3. **`deal_breaker_type` field added** to `excluded_by_deal_breaker` array entries — captures whether each exclusion was `hard` or `soft`

4. **`schedule_viability` sub-field added** to `step_reasoning` — records whether non-standard schedule was detected, whether no-match was triggered, and which soft deal-breakers were suspended

**Design rationale — why soft deal-breakers can be suspended:**
The no-match detection only triggers when ALL helpers fail schedule coverage. At that point, the care-experience deal-breaker is moot as a primary filter — the family has no helpers who meet both criteria. The correct fallback is to surface the most schedule-flexible helpers available and be transparent about the gap, rather than returning high-care-experience helpers whose schedules are completely wrong.

Importantly: only `soft` deal-breakers (inferred or preference-based) are suspended. `Hard` deal-breakers (explicitly stated by family as mandatory, or safety baselines) are never waived. This preserves the agent's safety guarantees. For example: if a family with an MS patient explicitly states "must have MS care experience", that remains a hard exclusion even in a no-match scenario.

**Dependency:** Requires Agent 1 v5 — deal_breakers must be typed objects `{text, type}` for Step 2.5 to read the `type` field. Running v7 against Agent 1 v4 output (plain string deal_breakers) will cause Step 2.5 to treat all deal-breakers as hard by default (safe fallback).

**Prompt:**

```
You are a care matching specialist. Your job is to score a pool of care helpers against a family's care profile and return the top 3 best-fit helpers.

You will be given:
1. A care profile JSON (the family's needs)
2. A list of helper profiles JSON

Follow these steps exactly:

STEP 1 — BUDGET GATE
Only apply if care_profile.budget is not null.
  - Parse the upper bound of the family's budget range
  - Calculate the exclusion threshold: upper_bound × 1.20
  - Silently exclude any helper whose hourly_rate exceeds this threshold
  - If care_profile.budget is null, skip this step entirely

STEP 2 — DEAL-BREAKER CHECK
For each helper not excluded in Step 1, check each deal-breaker against what is known or inferable about the helper.
Each deal-breaker in the care profile has a type field: "hard" or "soft".
If any single deal-breaker is confirmed or strongly inferable, exclude that helper entirely.
Log each excluded helper with the specific deal-breaker that caused exclusion and its type.

Only exclude a helper if a deal-breaker is clearly confirmed or strongly inferable. Do not exclude on weak signals.

STEP 2.5 — SCHEDULE VIABILITY CHECK
Run this after Step 2.

Check whether the family's required schedule is non-standard — meaning any window that falls between 6pm and 9am, or is weekend-only.

If yes: count how many helpers surviving Step 2 have any confirmed or inferable availability overlap with those required hours.

If zero surviving helpers cover the required hours — this is a no-match scenario. Do the following:
  1. Restore any helpers that were excluded in Step 2 solely because of "soft" deal-breakers. Keep all "hard" deal-breaker exclusions intact.
  2. Proceed to Step 3 with this restored pool.
  3. In Step 3, prioritise schedule flexibility: helpers who explicitly state "flexible schedule", "open to different hours", or similar score 2.0–2.5 on schedule_fit.
  4. Set "no_match": true in the output JSON. In step_reasoning.schedule_viability, record which soft deal-breakers were suspended and which helpers were restored.

If the required schedule is standard (daytime/weekday) OR at least one surviving helper has schedule overlap — skip this step entirely and proceed to Step 3 normally. Set "no_match": false.

STEP 3 — SCORE EACH REMAINING HELPER
Score each helper across 4 dimensions. Use one decimal point only.
If data for a dimension is missing and cannot be inferred, score that dimension as 0 and note it in profile_gaps.

Schedule Fit (0–2.5):
  - 2.5: helper's days and hours fully cover the family's need
  - 1.5–2.4: partial overlap — minor gaps in days or hours
  - 0.5–1.4: limited overlap — significant gaps
  - 0.0: no overlap or schedule completely unspecified and cannot be inferred

Care Experience (0–3.0):
  - 2.5–3.0: direct, substantial experience with the exact condition stated
  - 1.5–2.4: some related experience or limited direct condition experience
  - 0.5–1.4: general elderly care only, no specific condition match
  - 0.0: no relevant experience evident or cannot be assessed

Skills Match (0–2.5):
  - Score proportionally based on how many required_skills from the care profile the helper demonstrably covers
  - Full coverage = 2.5, each clearly missing skill reduces proportionally
  - 0.0 if skills field is empty and cannot be inferred from bio

Qualifications Match (0–2.0):
Score ONLY against the care profile's explicit_qualifications and inferred_qualifications fields.
  - 1.8–2.0: holds one or more certifications that exactly match explicit_qualifications or inferred_qualifications
  - 1.0–1.7: holds a certification with partial overlap — same domain, not exact match
  - 0.0: no certifications matching explicit or inferred qualifications

Volume rule: 5 relevant certifications scores no higher than 1 relevant certification.

Composite score = sum of 4 dimension scores. Maximum = 10.0.
Tie-breaking rule: if two helpers have equal composite scores, the one with the LOWER hourly_rate must appear earlier. If rates are also equal, the one with MORE years_of_experience appears earlier.

STEP 4 — RETURN TOP 3
Rank all scored helpers by composite score descending, applying tie-breaking rule where needed.
Return only the top 3.

Return ONLY a valid JSON object in this exact format — no text before it, no text after it:
{
  "no_match": false,
  "step_reasoning": {
    "budget_gate": "state whether budget was null (skipped) or the threshold calculated and how many helpers were excluded",
    "deal_breaker_summary": "one sentence listing which helpers were excluded and why, noting each deal-breaker's type (hard/soft)",
    "schedule_viability": "state whether the family schedule is non-standard; if no-match scenario triggered, list which soft deal-breakers were suspended and which helpers were restored",
    "scoring_notes": "2–3 sentences summarising how the top scorers separated from the rest"
  },
  "excluded_by_deal_breaker": [
    {
      "helper_id": "...",
      "name": "...",
      "deal_breaker_triggered": "exact deal-breaker text from care profile",
      "deal_breaker_type": "hard or soft",
      "reasoning": "one sentence explaining the inference"
    }
  ],
  "top_3": [
    {
      "rank": 1,
      "helper_id": "...",
      "name": "...",
      "hourly_rate": 0,
      "scores": {
        "schedule_fit": 0.0,
        "care_experience": 0.0,
        "skills_match": 0.0,
        "qualifications_match": 0.0
      },
      "composite_score": 0.0,
      "profile_gaps": [],
      "score_reasoning": "2–3 sentences explaining why this helper scored as they did"
    }
  ]
}

Rules:
- hourly_rate: always populate directly from the helper's input profile. Never return null if the value exists in the input data.
- Do not penalise helpers for missing fields that are irrelevant to this specific care profile.
- Only note a field in profile_gaps if it was missing, unresolvable, and caused a dimension to score 0.
- Only exclude a helper on a deal-breaker if it is clearly confirmed or strongly inferable — not on weak or ambiguous signals.
```

**Additional fix (April 26):** Added internal reasoning instruction before the JSON format block: "Do all scoring calculations internally. Do not write out per-helper scores or reasoning in text before the JSON. The step_reasoning fields are summaries only — 1–3 sentences each." Root cause: F013 (19 helpers after budget gate) produced exhaustive per-helper markdown reasoning before the JSON, hitting the 8192 token limit mid-output and causing a parse error.

**Target improvement:** Rank-1 accuracy ≥ 9/17 (from 6/17 in v6). Primary target is F011 no-match detection. Secondary benefit expected on any other families with non-standard schedule requirements.

**Output produced:** *(to be filled after pipeline re-run)*

**Failure modes observed:** *(to be filled after pipeline re-run)*

**Status: SUPERSEDED by v8**

---

### v8 — Revised Prompt (April 26, 2026)

**Reason for Revision:**
v7 eval run (April 26): rank-1 accuracy improved to 8/17 (47%) from 6/17 (35%). Four failure patterns remain across 9 wrong families:

- **F009**: Care experience over-scored for vague conditions — dementia specialists awarded 3.0 for "aging in place with cognitive concerns." Agent 1 v6 already stopped inferring qualifications for vague conditions, but Agent 2 still awarded the top care_experience band.
- **F011**: No-match detection triggering correctly (no_match=True) but H004 (Rosa, 3.0 care experience, 0.5 schedule_fit) still outranking H015 (schedule-flexible, restored from soft deal-breaker) purely on composite score.
- **F015**: Ranking error — H016 (composite 2.8) ranked 1st while H017/H020 (composite 3.0) ranked below it. Model broke its own ordering rule.
- **F002, F005, F016**: Schedule fit underweighted — helpers within 0.3 composite points ranked by care experience when schedule fit should differentiate.

**What changed — 4 targeted fixes integrated into existing steps:**

1. **Care Experience 2.5–3.0 band** (STEP 3): Added caveat — do not award top band if the care profile condition is vague/mild and the helper's experience is with a more severe or clinical version of a related condition. Vague conditions (e.g. "sometimes forgets things") should use the 1.5–2.4 band for experienced general elderly helpers.

2. **No-match scoring override** (STEP 2.5): After Step 3 scoring in a no-match scenario, restored helpers (previously excluded by soft deal-breakers) rank above any helper with schedule_fit ≤ 0.5 regardless of composite score. Within each group, rank by composite.

3. **Ranking verification** (before JSON): Explicit instruction added — "Before writing the JSON, verify your top_3 is in descending composite order — rank 1 must have the highest composite."

4. **Schedule-first secondary ranking** (STEP 4): When two helpers are within 0.3 composite points of each other, the one with higher schedule_fit ranks first. Integrated into STEP 4 alongside the existing tie-breaking rule.

**Removed:** Separate "Important: Do all scoring internally" paragraph — merged into STEP 4 as it belongs with the ranking and output instructions, not as a standalone note.

**Target improvement:** Rank-1 accuracy ≥ 12/17 (71%). Expected gains: F009 (+1), F011 (+1), F015 (+1), F001 (+1 via golden set relaxation), F002/F005/F016 (+2–3 via schedule-first secondary ranking).

**Output produced:** 7/17 rank-1 accuracy. Regression from v7 (8/17). Weight change (Schedule Fit 3.0, Skills Match 2.0) caused F004 to flip wrong (H009 care=2.0 now outranks H002 care=1.5 with equal schedule). F010 ranking error: H002 (composite 6.0) ranked 3rd behind H007/H013 (5.6) — verification step not catching it.

**Failure modes observed:**
- F010: Ranking error — H002 composite 6.0 ranked 3rd. Step 4 sort instruction insufficient.
- F002: H003 (ends 12pm) scored sched=2.5 despite 2-hour end gap on 8am–2pm family window. Schedule scoring not penalising partial coverage correctly.
- F005: H004 (4.5hr coverage of 8hr window) scored higher than H001 (5hr coverage). Proportion-based scoring not applied.
- F011: Flexible-schedule helpers (H015, H009, H018) score 0.0 on schedule — same as daytime-only helpers — so high care-experience daytime helpers outrank them.
- F015: Budget gate applied inconsistently — H015 ($16) excluded while H020 ($17) included under $18 threshold.

**Status: SUPERSEDED by v9**

---

### v9 — Revised Prompt (April 28, 2026)

**Reason for Revision:**
Five distinct failures from v8 run, identified through golden set audit:

1. **F002 + F005 — Schedule scoring ignores proportion of window covered**: H003 (ends 12pm, family needs until 2pm) and H004 (covers 4.5 of 8 required hours) both scored 2.5 on schedule_fit — the model detected overlap but didn't penalise how much of the window was missed. Fix: replaced the schedule rubric with proportion-based bands and added a Coverage rule that requires calculating overlap accounting for both start and end times symmetrically.

2. **F015 — Budget gate inconsistency**: Model excluded H015 ($16) and H016 ($17) while including H020 ($17) under a $18 threshold. Fix: added a worked example to Step 1 making the threshold arithmetic explicit so the model applies it consistently.

3. **F011 — Flexible schedule helpers scored 0.0 in no-match scenario**: In a no-match overnight scenario, helpers with "Flexible" schedule should score higher than helpers with confirmed daytime-only schedules. The v8 Step 2.5 item 3 said "prioritise schedule flexibility" but the model was still giving all helpers similar 0.5 scores. Fix: replaced the vague instruction with an explicit override — Flexible = 2.0, confirmed daytime-only = 0.0–0.5 — and stated that Flexible must always score higher.

4. **F010 — Ranking error persists**: H002 (composite 6.0) ranked 3rd behind H007/H013 (5.6). The v8 "verify descending order" instruction was not sufficient. Fix: replaced with an explicit sort-then-assign instruction — model must write the sorted list first, then assign ranks from it top-down.

5. **Golden set update**: F004 and F009 given `acceptable_rank_1` relaxation after audit confirmed both H009/H002 (F004) and H007/H010/H002 (F009) are defensible given rubric precision limits. F016 golden set corrected — H001 ($32) is over budget for this family ($24–$30 max), H007 is correct rank 1.

**What changed in Agent 2 (v8 → v9):**

1. **Schedule Fit rubric** — replaced 4-band rubric with proportion-based rubric:
   - 3.0: full coverage
   - 2.0–2.9: minor gaps (≤30 min, or missing one day)
   - 1.0–1.9: half to three-quarters coverage
   - 0.5: less than half coverage
   - 0.0: no overlap or unspecified
   - Coverage rule added: *"Calculate overlap accounting for both the helper's start time and end time against the family's required window — neither end is discounted."*

2. **Step 1 (Budget Gate)** — added worked example with $13–$15/hr budget showing $16 and $17 pass, $19 does not.

3. **Step 2.5 item 3** — replaced "prioritise schedule flexibility" with explicit override:
   - Flexible days/hours = schedule_fit 2.0
   - Confirmed daytime-only = 0.0–0.5
   - Stated rule: *"Flexible schedule must score higher than confirmed daytime-only for non-standard families."*

4. **Step 4 (Ranking)** — replaced "verify descending order" with sort-then-assign: *"Write out all scored helpers sorted by composite, highest first. Assign rank 1 to the first, rank 2 to the second, rank 3 to the third."* Tie-breaking rule retained unchanged.

**Target improvement:** Rank-1 accuracy ≥ 12/17. Golden set fixes alone project +4 (F016 corrected, F004/F009/F017 relaxed). Prompt fixes target F002, F005, F010, F011, F015 (+4–5).

**Output produced:** *(to be filled after pipeline re-run)*

**Failure modes observed:** *(to be filled after pipeline re-run)*

**Status: ACTIVE ✓**

---

## Agent 3 — Match Quality & Trust Agent

**Job:** Take the top 3 scored helpers from Agent 2 and write a trust-building, family-facing explanation for each. Explanations must be specific (every claim grounded in the helper's profile), personal (mirror the family's own language back to them), and emotionally calibrated (acknowledge the weight of the decision without being dramatic). No scores shown to the family — explanation only.

**Model:** Claude Sonnet 4.6

**Design rationale:** This is the user-facing output — the thing a family actually reads before deciding whether to reach out to a helper. It requires nuanced, empathetic writing that balances factual grounding with emotional intelligence. Haiku is not sufficient here. Sonnet 4.6 is warranted.

**Trust-building framework — three principles Agent 3 must follow:**

1. **Specificity over genericism** — "Rosa has spent 10 years specifically with dementia clients" builds trust. "Rosa is experienced and caring" does not. Every claim must trace to a specific fact in the helper's profile.

2. **Mirror the family's language** — if the family said "she is very private and would not be comfortable with someone too chatty," the explanation should reflect that framing directly: "Rosa describes herself as someone who believes in being present rather than filling the room with conversation."

3. **Acknowledge the emotional stakes** — the family is not hiring a cleaner. They are trusting a stranger with someone they love. The tone must hold that weight without being clinical or transactional.

---

### v1 — Initial Prompt (April 3, 2026)

**Prompt:**

```
You are a senior care coordinator writing match recommendations for a family who is looking for care help.

You will be given:
1. The family's care profile (what they need, who they are, what matters to them)
2. The top 3 helper profiles selected for this family, in ranked order

Your job is to write a short, trust-building explanation for each helper — the explanation the family will read when deciding who to reach out to.

Rules:
- Write in warm, plain English — not clinical, not corporate
- Every claim you make about a helper must be traceable to a specific fact in their profile. Do not invent or embellish.
- Mirror the family's own language where possible. If they described a need in a particular way, reflect that framing back.
- Acknowledge that this is not a routine decision. The family is trusting someone with a person they love.
- Do not show scores, rankings as numbers, or any system language. The family should feel they are reading a thoughtful human recommendation, not an algorithm output.
- Keep each explanation to 3–4 sentences. Specific and concise beats long and generic.

Return ONLY a JSON object in this format:
{
  "recommendations": [
    {
      "rank": 1,
      "helper_id": "...",
      "name": "...",
      "hourly_rate": N,
      "explanation": "3–4 sentence trust-building explanation grounded in specific profile facts and the family's own language"
    },
    {
      "rank": 2,
      "helper_id": "...",
      "name": "...",
      "hourly_rate": N,
      "explanation": "..."
    },
    {
      "rank": 3,
      "helper_id": "...",
      "name": "...",
      "hourly_rate": N,
      "explanation": "..."
    }
  ]
}
```

**Output produced:** (April 8, 2026)

```json
{
  "recommendations": [
    {
      "rank": 1,
      "helper_id": "H004",
      "name": "Rosa Mendez",
      "hourly_rate": 30,
      "explanation": "Rosa has spent 22 years in home care — 10 of them working specifically with people living with dementia — so she understands the rhythms and sensitivities that come with early-stage memory changes in a way that's hard to teach. Families often tell us they value someone who moves quietly and respectfully through a person's home, and Rosa's approach is exactly that: calm, unhurried, and genuinely mindful of privacy and boundaries. She's completely comfortable with animals and will have no trouble making Biscuit part of the morning routine alongside breakfast and light tidying. Her availability runs Monday through Friday, 7:30am to 12:30pm, which lines up perfectly with the weekday mornings you're looking for."
    },
    {
      "rank": 2,
      "helper_id": "H001",
      "name": "Margaret Chen",
      "hourly_rate": 32,
      "explanation": "Margaret brings 18 years of experience to this kind of work, with 8 of those years focused specifically on early to mid-stage dementia and Alzheimer's care — which means she's well-practiced in the gentle, consistent presence that makes such a difference at this stage. She holds four certifications including dementia care and CPR/First Aid, so you can feel confident she's prepared not just for the everyday moments but for anything unexpected. Her manner is described as calm and boundary-respecting, and she genuinely loves animals, so Biscuit would be in good hands. She's available Monday through Friday, 7:00am to 1:00pm, giving you full weekday morning coverage."
    },
    {
      "rank": 3,
      "helper_id": "H010",
      "name": "Amara Osei",
      "hourly_rate": 26,
      "explanation": "Amara has nine years of home-based elderly care experience, including two long-term relationships with clients living with dementia, and she's trained in the Teepa Snow approach — a method known for its focus on dignity and meeting people where they are. She handles all the practical pieces your mother needs — breakfast, light housekeeping, pet care, and thoughtful companionship — and holds both a dementia certification and CPR/First Aid. She's available Monday through Friday, 7:00am to 12:00pm, and at $26 an hour she may offer the most flexibility as you're figuring out the right schedule and budget. This is a genuinely strong option, especially if you'd like a little room to grow into the arrangement."
    }
  ]
}
```

**Failure modes observed:** None on this run. Explanations are specific, grounded in profile facts, warm in tone, and correctly avoid scores or system language. Biscuit (the family's dog) is named directly in Rosa's explanation — strong language mirroring. All three explanations are 4 sentences and complete.

**What will change in v2:** No changes based on this run. v1 is stable. Future revisions driven by eval results — specifically watching for generic language ("experienced and caring") and any claims not traceable to the helper's profile.

**Status: LOCKED ✓**

---

## LLM-as-Judge — Match Explanation Evaluator

**Job:** Independently score Agent 3's match explanations across 4 dimensions against the golden set. Must provide chain-of-thought reasoning before giving each score.

**Model:** GPT-4.1 (OpenAI)

**Design rationale:** The judge deliberately uses a different model family from the generator (Claude Sonnet 4.6) to eliminate self-serving bias — a model scoring its own outputs generously is a known failure mode in LLM-as-Judge pipelines. GPT-4.1 was selected over GPT-4o and Gemini 2.5 Pro for this role because instruction following is the most critical quality for a judge that must execute a strict sequence (quote → reason → score) and return a precise output schema consistently across runs. See Project_Tradeoffs_and_Decisions.md Decision 7 for full reasoning.

**Dimensions scored:**

| Dimension | Type | What It Measures |
|---|---|---|
| Specificity | Scored 1–5 | Does the explanation cite concrete facts from the helper's bio, or is it generic? |
| Relevance | Scored 1–5 | Does it directly address this family's stated needs and deal-breakers? |
| Factual Grounding | Pass / Fail | Every claim must be traceable to the helper bio — nothing invented |
| Empathy Tone | Scored 1–5 | Appropriate for a family making a high-stakes care decision? |

---

### v1 — Initial Prompt (April 21, 2026)

**Design decisions made before writing:**

1. **Chain-of-thought before score is mandatory** — the prompt requires the judge to quote or paraphrase the explanation, reason about it, and only then score. A score without reasoning is unauditable and cannot be used to improve Agent 3. This is enforced in both the instructions and the output schema — the reasoning field appears before the score field so the model cannot shortcut to the score.

2. **Factual Grounding is Pass/Fail, not scored** — a hallucination is a binary failure. Scoring it 1–5 would imply partial hallucination is acceptable. It is not. Any claim that cannot be traced to the helper's profile is an automatic Fail, and the specific failed claim must be identified. This makes the eval actionable.

3. **Evaluated per helper, not per family** — Agent 3 writes 3 explanations per family. Evaluating them as a set obscures which specific explanation failed. Each helper explanation is evaluated independently so failures are traceable to the exact output.

4. **`failed_claims` always returned** — even on a Pass, returned as an empty array. Keeps the output schema consistent for automated parsing.

5. **`overall_assessment` is required** — a structured score alone does not tell Agent 3 what to do differently. The overall assessment synthesises the four dimensions into one actionable observation: what worked and the single most important thing to improve.

**Prompt:**

```
You are an independent evaluator assessing the quality of a care 
match explanation written for a family seeking elderly care support.

You will be given:
1. The family's care profile — their care needs, must-haves, 
   deal-breakers, and how they described their situation
2. The helper's profile — their background, experience, skills, 
   and certifications
3. The match explanation — the text written to help this family 
   decide whether to reach out to this helper

Your job is to evaluate the explanation across 4 dimensions.

You MUST follow this process for every dimension:
  Step 1 — Quote or paraphrase the specific part of the 
            explanation you are evaluating
  Step 2 — Reason about whether it meets the criterion
  Step 3 — Give your score or result

Do not give scores without completing Steps 1 and 2 first.

---

DIMENSION 1 — SPECIFICITY (1–5)
Does the explanation cite concrete, verifiable facts from the 
helper's profile, or does it use language that could apply to 
any helper?

  5 — Every meaningful claim cites a specific fact: years of 
      experience, exact certification name, specific condition 
      or prior role. No generic language present.
  4 — Mostly specific. One or two phrases are generic but do 
      not mislead.
  3 — Mix of specific and generic. Some claims are grounded, 
      others are floating.
  2 — Mostly generic. Specific facts used sparingly. Could 
      describe most helpers.
  1 — Entirely generic. No facts from the helper's profile 
      are cited.

---

DIMENSION 2 — RELEVANCE (1–5)
Does the explanation directly address what this specific family 
described — their care needs, their must-haves, and their 
stated concerns?

  5 — Explanation directly addresses the family's specific 
      needs, mirrors their own language back to them, and 
      speaks to their stated concerns and deal-breakers.
  4 — Addresses most of the family's key needs with minor gaps.
  3 — Addresses some needs but misses at least one significant 
      concern the family raised.
  2 — Generic care match language. Does not engage with this 
      family's specific situation.
  1 — Irrelevant or misaligned with what the family described.

---

DIMENSION 3 — FACTUAL GROUNDING (Pass / Fail)
Every claim in the explanation must be traceable to a specific 
fact in the helper's profile. Nothing invented or embellished.

  Pass — All claims can be verified against the helper's profile.
  Fail — One or more claims cannot be traced to the helper's 
         profile. Any hallucinated or embellished fact = 
         automatic Fail.

For this dimension, list every factual claim in the explanation 
and state whether each is traceable to the helper's profile. 
Check years of experience, certification names, prior roles, 
skills, and any personality or behavioural claims.

---

DIMENSION 4 — EMPATHY TONE (1–5)
Is the tone appropriate for a family making a high-stakes care 
decision — warm and human, without being clinical, 
transactional, or overdramatic?

  5 — Warm, human, and holds the emotional weight of the 
      decision without being dramatic or clinical. Reads like 
      a thoughtful person wrote it.
  4 — Mostly appropriate tone. Minor lapses into clinical or 
      transactional language.
  3 — Neutral — neither cold nor warm. Reads more like a 
      professional summary than a human recommendation.
  2 — Clinical or transactional language dominates. Feels 
      like a product listing.
  1 — Inappropriate tone — either cold and detached, or 
      overdramatic and manipulative.

---

Return ONLY a JSON object in this format:

{
  "family_id": "...",
  "helper_id": "...",
  "evaluation": {
    "specificity": {
      "reasoning": "quote or paraphrase the relevant part of 
                   the explanation, then reason about whether 
                   claims are specific or generic",
      "score": N
    },
    "relevance": {
      "reasoning": "quote or paraphrase the relevant part of 
                   the explanation, then reason about whether 
                   it addresses this family's specific needs 
                   and language",
      "score": N
    },
    "factual_grounding": {
      "reasoning": "list each factual claim in the explanation 
                   and state whether it is traceable to the 
                   helper's profile",
      "result": "pass or fail",
      "failed_claims": ["list any claims that cannot be 
                        verified — empty array if pass"]
    },
    "empathy_tone": {
      "reasoning": "quote or paraphrase the relevant part of 
                   the explanation, then reason about whether 
                   the tone is appropriate",
      "score": N
    }
  },
  "overall_assessment": "2–3 sentences: what the explanation 
                         did well, and the single most important 
                         thing to improve"
}

Rules:
- Always complete reasoning before scoring. Never output a 
  score without the reasoning that precedes it.
- Factual Grounding is binary. Partial hallucination is not 
  a 2 or 3 — it is a Fail.
- failed_claims must always be returned — empty array if Pass, 
  populated list if Fail.
- Do not reward an explanation for mentioning facts not relevant 
  to this family's needs — relevance and specificity are 
  separate dimensions.
- Do not penalise warm or emotional language if it is grounded 
  in fact. Empathy and specificity are not in conflict.
```

**Test input:** Agent 3 v1 output for F001 (Rosa Mendez explanation) + F001 care profile + H004 helper profile

**Expected behaviour:**
- Specificity: 4–5 — Rosa's explanation names 22 years, 10 years dementia focus, CDP/CADDCT, Biscuit by name
- Relevance: 5 — explanation directly mirrors the family's language about privacy, dog, routine
- Factual Grounding: Pass — all claims traceable to H004 profile
- Empathy Tone: 4–5 — warm, human, not clinical

**Output produced:** (April 24, 2026) — 51 explanations evaluated across all 17 families

**Score summary:**

| Dimension | Result |
|---|---|
| Specificity | avg 4.94/5 — distribution: 48×5, 3×4, 0×(1–3) |
| Relevance | avg 4.59/5 — distribution: 31×5, 19×4, 1×3, 0×(1–2) |
| Factual Grounding | 50/51 pass, 1 fail |
| Empathy Tone | avg 5.00/5 — 51×5 |

**Factual grounding failure:**
- F016/H004 (rank 2) — *"Families who have gone through the experience of finding the right helper often say they wished they'd found someone like Rosa sooner."* This is a fabricated social proof testimonial not traceable to Rosa's profile. Agent 3 added persuasive framing that violates the factual grounding rule.

**Failure modes observed:**
- 1 hallucination (F016/H004) — social proof invented
- Relevance scores of 4 (not 5) are concentrated in cases where Agent 2 ranked the wrong helper at #1 — Agent 3 wrote a technically strong explanation for the wrong person, which the judge correctly penalised on relevance
- 1 relevance score of 3 (F015/H020) — explanation was generic for a helper with a thin profile

**What worked:**
- Empathy tone is perfect across all 51 explanations — the trust-building framing is working as designed
- Specificity near-perfect — the "every claim must be traceable" rule is being followed
- No generic language failures (0 Type B) — Agent 3 is grounding claims in profile facts throughout

**Status: ACTIVE ✓ — v1 locked, monitoring for hallucination pattern in future runs**

---

## Iteration Summary Table

*(Updated after each eval run)*

| Agent | Version | Key Change | Rank-1 Accuracy | Set Match | Avg Overlap | Specificity | Relevance | Factual Grounding | Empathy Tone |
|---|---|---|---|---|---|---|---|---|---|
| Agent 2 | v6 | Relevance-strict qualifications scoring | 6/17 (35%) | 2/17 (12%) | 1.8/3 | — | — | — | — |
| Agent 2 | v7 | No-match detection + hard/soft deal-breaker typing | 8/17 (47%) | 2/17 (12%) | 1.8/3 | — | — | — | — |
| Agent 1 v6 + Agent 2 | v8 | Cultural/demographic deal-breaker + schedule/care experience weight change (Sched 3.0, Skills 2.0) | 7/17 (41%) | — | — | — | — | — | — |
| Agent 1 v7 + Agent 2 | v9 | Past-behavior hard deal-breaker + companion care soft deal-breaker + proportion-based schedule scoring + budget gate example + flexible schedule override + sort-then-assign ranking | **11/17 (65%)** | 2/17 (12%) | 1.6/3 | — | — | — | — |
| Agent 3 | v1 | Initial prompt | — | — | — | 4.94/5 | 4.59/5 | 50/51 pass | 5.00/5 |

**Eval run date:** April 24, 2026 (v6) · April 26, 2026 (v7, v8) · April 28, 2026 (v9)
**Pipeline:** Agent 1 v4 → Agent 2 v6 → Agent 3 v1 (baseline) · Agent 1 v5 → Agent 2 v7 → Agent 3 v1 (v7) · Agent 1 v6 → Agent 2 v8 → Agent 3 v1 (v8) · Agent 1 v7 → Agent 2 v9 → Agent 3 v1 (v9)
**Families evaluated:** 17/17
**Explanations judged:** 51 (3 per family)

---

## Failure Taxonomy

*(Updated after each eval run — April 24, 2026)*

| Label | Description | Count | Instances |
|---|---|---|---|
| A — Hallucination | Explanation cites a fact not in the helper bio | 1 | F016/H004 — fabricated social proof testimonial ("Families who have gone through the experience often say...") |
| B — Generic | Language that could apply to any helper | 0 | — |
| C — Relevance miss | Explanation ignores a key need from the care profile | 0 | — |
| D — Tone mismatch | Language is clinical for a care context | 0 | — |
| E — Wrong rank | Best-fit helper not ranked #1 by Agent 2 | 6 | F002, F005, F007, F011, F015, F016 |

**Primary failure pattern:** Type E (wrong rank 1) — final 6 families after v9 iteration. F002 and F005 are close ties; F007, F011, F015, F016 have edge-case scheduling or scoring nuances beyond current rubric resolution. Accepted as iteration endpoint.

**Final score: 11/17 (65%) rank-1 accuracy with Agent 1 v7 + Agent 2 v9 + Agent 3 v1. Prompt iteration closed. Moving to PRD → README → GitHub upload.**

---

*This file updates after every prompt change. Never delete old versions — the history is the artifact.*
