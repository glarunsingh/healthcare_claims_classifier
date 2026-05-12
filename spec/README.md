# Healthcare Claim Injury Classification — Spec-Driven Design

> **POC** — Classify healthcare injury claims as Work-Related, Non-Work-Related, or Undetermined using an AI/LLM, reading from and writing to Excel.

---

## Quick Summary

**Input:** Excel file with claim ID, date, and supporting document text columns  
**Output:** Same Excel with classification, confidence score, reasoning, and excerpts appended  
**How:** Python script calls GPT-4o for each claim row, parses structured JSON response  

---

## SDD Documents (Read in Order)

| Phase | Document | Purpose |
|---|---|---|
| 1. SPECIFY | [specs/claim-classification.md](specs/claim-classification.md) | What we're building — requirements, examples, acceptance criteria |
| 2. PLAN | [plans/plan.md](plans/plan.md) | How we'll build it — tech stack, architecture, data flow |
| 3. TASKS | [tasks/tasks.md](tasks/tasks.md) | Step-by-step implementation tasks with pseudocode |
| 4. IMPLEMENT | [rules/rules.md](rules/rules.md) | Coding rules, validation checklist, future roadmap |

---

## Source of Truth

The original client requirements are in [userstory.md](../userstory.md) (root of repo).

---

## Assumptions

All assumptions are listed in the SPECIFY doc. Key ones:

- Excel input only (no PDFs, no images, no APIs)
- Document text is already extracted and pasted into Excel columns
- GPT-4o via OpenAI SDK (can switch to Azure OpenAI)
- Sequential processing (no parallelism needed for POC)
- This is a **POC for client demonstration**, not production code
