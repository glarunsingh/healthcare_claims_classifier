# Phase 4 — IMPLEMENT: Rules & Execution Guide

> How to build it right — rules for coding agents and developers implementing the tasks.

---

## How to Use This Spec

```
1. Read the SPECIFY doc    → Understand WHAT we're building
2. Read the PLAN doc       → Understand HOW it's designed
3. Follow the TASKS doc    → Build each task in order, with pseudocode as guide
4. Follow THIS doc         → Rules to keep the implementation on track
```

---

## Ground Rules for Implementation

### Scope Rules (DO NOT violate these)

| Rule | Description |
|---|---|
| R-1 | This is a **CLI script**, NOT a web app. No Flask, no FastAPI, no endpoints. |
| R-2 | **No database**. Data lives in Excel files only. |
| R-3 | **No authentication or authorization**. This is a local POC. |
| R-4 | **No Docker** required to run. Just `pip install` and `python src/main.py`. |
| R-5 | **No async/await**. Process claims sequentially in a simple loop. |
| R-6 | Keep it to **6 source files** max (as defined in the PLAN). |
| R-7 | All config via `.env`. No CLI arguments, no YAML config files. |

### Code Quality Rules

| Rule | Description |
|---|---|
| Q-1 | Every function should be under 50 lines. If it's longer, something is wrong. |
| Q-2 | Use type hints on function signatures. |
| Q-3 | Print progress to console (claim X of Y). No logging framework needed. |
| Q-4 | Handle errors per-row — one bad claim should NOT crash the entire run. |
| Q-5 | Validate LLM output with Pydantic. If it doesn't parse, mark as ERROR. |

### LLM Prompt Rules

| Rule | Description |
|---|---|
| P-1 | Use `temperature=0.1` for consistency. |
| P-2 | Use `response_format={"type": "json_object"}` to force JSON output. |
| P-3 | System prompt must include the full classification decision tree. |
| P-4 | User prompt must label each document type clearly (e.g., `--- DOCTOR NOTES ---`). |
| P-5 | Do NOT send the entire Excel — send one claim at a time. |

---

## Validation Checklist

After implementing all 6 tasks, verify these before calling the POC complete:

```
[ ] python src/main.py runs without errors on sample_claims.xlsx
[ ] Output Excel has all 7 new columns (classification through needs_human_review)
[ ] Scenario 1 (CLM-001) → NON_WORK_RELATED + WORK_AGGRAVATED
[ ] Scenario 2 (CLM-002) → WORK_RELATED
[ ] Scenario 3 (CLM-003) → UNDETERMINED + needs_human_review = true
[ ] Missing API key → clear error message, no crash
[ ] Empty text columns → UNDETERMINED (no LLM call)
[ ] Row with bad LLM response → ERROR, rest of file still processes
[ ] .env.example is filled in with all variable names
[ ] requirements.txt matches what's actually imported
```

---

## What Comes AFTER the POC

This is out of scope for now but documented for the client conversation:

| Future Enhancement | Description |
|---|---|
| Batch API calls | Use OpenAI Batch API to reduce cost for large files |
| Web UI | Simple Streamlit or Gradio app for non-technical users |
| Audit trail | Log each classification decision with timestamps |
| Multi-model voting | Run claim through 2+ models, compare results |
| Azure OpenAI | Switch from OpenAI to Azure OpenAI for enterprise compliance |
| Confidence calibration | Fine-tune thresholds based on human review feedback |
| PDF/image support | Extract text from scanned documents (OCR) |
