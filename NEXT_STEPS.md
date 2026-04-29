# HAVEN — What's Left To Do
*Check this file whenever you pick up the project.*
*Strike through items as they are completed.*

---

## Critical Path — GitHub Upload (Resume-Ready)

- [x] **1. Run pipeline** — Run Agent 1 → 2 → 3 on all 17 families. Collect outputs in `evals/outputs/pipeline/`.
- [x] **2. Run evals** — Feed every Agent 3 explanation through the GPT-4.1 judge. Store results in `evals/outputs/judge/`.
- [x] **3. Analyse failures** — Fill the Iteration Summary Table and Failure Taxonomy in `evals/prompts/prompt_log.md`.
- [x] **4. Prompt iteration** — Agent 1 v7 + Agent 2 v9. Final score: 11/17 (65%) rank-1 accuracy. Closed April 28, 2026.
- [x] **5. PRD** — Written as `PRD_HAVEN.md` in the haven root. v2.0 — full production vision + prototype results. April 28, 2026.
- [x] **6. README** — Written as `README.md` in haven root. April 28, 2026.
- [x] **7. Upload to GitHub** — https://github.com/sashar6093/Haven_AI_CareMarketplace. April 28, 2026. Start applying.

---

## Post-Upload Additions (Keep Adding as You Go)

- [ ] **8. Monitoring and guardrails design** — Document production observability strategy in `Project_Tradeoffs_and_Decisions.md`.
- [ ] **9. n8n workflow** — Workspace retrieved (April 26). Build the pipeline in n8n with locked prompts (Agent 1 v5 → Agent 2 v7/v8 → Agent 3 v1).
- [ ] **10. Demo website** — Lovable UI (free-text family description input, real-time AI matching, 3 helper result cards with Agent 3 explanations). Backend: n8n webhook. Deploy to public URL — interviewers can test the product themselves. Build last, after n8n is complete.

---

## Nice To Have

- [ ] **Human annotation calibration** — Manually score 5–6 Agent 3 explanations across the 4 judge dimensions. Compare against GPT-4.1 judge scores to validate calibration.

---

*Last updated: April 28, 2026*
