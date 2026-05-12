# Phase 1 — SPECIFY: Healthcare Claim Injury Classification

> What are we building, why, and what does success look like?

---

## User Story

**As a** claim reviewer,  
**I want** an AI system that reads claim documents from an Excel sheet and classifies each injury as work-related, non-work-related, or undetermined,  
**so that** I can process claims faster with consistent, explainable decisions.

---

## How It Works (Simple)

```
┌─────────────────┐      ┌──────────────────┐      ┌───────────────────┐
│  Excel Input     │ ──►  │  AI Classifier   │ ──►  │  Excel Output     │
│                  │      │  (LLM-powered)   │      │                   │
│ • claim_id       │      │                  │      │ • claim_id        │
│ • claim_date     │      │  Reads all text  │      │ • classification  │
│ • doctor_notes   │      │  columns, applies│      │ • secondary_tag   │
│ • incident_report│      │  decision logic  │      │ • confidence      │
│ • transcript     │      │                  │      │ • reasoning       │
│ • employer_stmt  │      │                  │      │ • key_excerpts    │
│ • ...            │      └──────────────────┘      └───────────────────┘
└─────────────────┘
```

---

## Assumptions (POC Scope)

| # | Assumption |
|---|---|
| A1 | Input is a single Excel file (`.xlsx`), one row per claim |
| A2 | Each row has a `claim_id`, `claim_date`, and one or more text columns containing document content (doctor notes, incident reports, transcripts, etc.) |
| A3 | All document text is in English |
| A4 | The AI classifier is an LLM (e.g., GPT-4o or Claude) called via API |
| A5 | Output is a new Excel file with the original data + classification columns appended |
| A6 | This is a POC — no database, no web UI, no authentication. Just a Python script |
| A7 | The client will provide an LLM API key to run this |

---

## Classification Categories

| Classification | When to use |
|---|---|
| `WORK_RELATED` | Evidence clearly shows the injury happened during work hours, on work premises, or while doing work duties |
| `NON_WORK_RELATED` | Evidence clearly shows the injury happened outside of work |
| `UNDETERMINED` | Evidence is conflicting, insufficient, or missing |

### Secondary Tag

| Tag | When to use |
|---|---|
| `WORK_AGGRAVATED` | Only when classification is `NON_WORK_RELATED` AND work activities made the condition worse |

---

## Input Excel Structure (Expected)

| Column | Required | Description |
|---|---|---|
| `claim_id` | Yes | Unique claim identifier |
| `claim_date` | Yes | Date the claim was filed |
| `claimant_name` | No | Name of the person filing |
| `doctor_notes` | No | Doctor/physician notes |
| `incident_report` | No | Workplace incident report |
| `transcript` | No | Conversation transcript (patient-doctor, etc.) |
| `employer_statement` | No | Employer's statement about the incident |
| `medical_records` | No | Supporting medical records |

> **Note:** At least one text column must have content for classification to proceed. The script will read ALL text columns and pass them to the LLM.

---

## Output Excel Structure

The output Excel will contain all original columns PLUS these appended columns:

| Column | Type | Description |
|---|---|---|
| `classification` | string | `WORK_RELATED` · `NON_WORK_RELATED` · `UNDETERMINED` |
| `secondary_tag` | string or blank | `WORK_AGGRAVATED` or empty |
| `confidence_score` | float | `0.00` to `1.00` — how confident the AI is |
| `reasoning` | string | Plain-English explanation of why this classification was chosen |
| `key_excerpts` | string | Verbatim quotes from the documents that support the decision |
| `conflicts_or_gaps` | string | Any contradictions found or missing info noted |

---

## Example Scenarios

### Scenario 1 — Non-Work with Aggravation

**Input text (transcript column):**
> "Patient developed neck pain after sleeping in an incorrect position at home. Pain worsened while operating machinery at work."

**Expected output:**

| Field | Value |
|---|---|
| classification | `NON_WORK_RELATED` |
| secondary_tag | `WORK_AGGRAVATED` |
| confidence_score | `0.85` |
| reasoning | Injury originated at home (sleeping position). Work activities (machinery operation) aggravated existing condition. |
| key_excerpts | "neck pain after sleeping in an incorrect position at home", "pain worsened while operating machinery at work" |

### Scenario 2 — Work-Related

**Input text (incident_report column):**
> "Employee fell down the stairs while carrying documents during work hours."

**Expected output:**

| Field | Value |
|---|---|
| classification | `WORK_RELATED` |
| secondary_tag | |
| confidence_score | `0.95` |
| reasoning | Injury occurred on work premises, during work hours, while performing work duties. |
| key_excerpts | "fell down the stairs while carrying documents during work hours" |

### Scenario 3 — Undetermined

**Input text (doctor_notes column):**
> "Patient reports back pain. Unclear when it started."

**Expected output:**

| Field | Value |
|---|---|
| classification | `UNDETERMINED` |
| secondary_tag | |
| confidence_score | `0.30` |
| reasoning | Insufficient evidence to determine whether injury is work-related. No incident details, no timeline. |
| conflicts_or_gaps | "No incident report provided. No employer statement. Origin timeline missing." |

---

## Success Criteria

| # | Criteria |
|---|---|
| SC-1 | The script reads an Excel, classifies all rows, and produces an output Excel |
| SC-2 | All 3 example scenarios above produce correct classifications |
| SC-3 | Every classification includes a confidence score and reasoning |
| SC-4 | The script handles missing/empty text columns gracefully (marks as UNDETERMINED with low confidence) |
| SC-5 | Processing time: ≤ 30 seconds per claim |

---

## Constraints (DO NOT)

| # | Constraint |
|---|---|
| C-1 | Do NOT build a web UI — this is a script-based POC |
| C-2 | Do NOT build a database — read from Excel, write to Excel |
| C-3 | Do NOT make legal determinations — this is a classification recommendation only |
| C-4 | Do NOT hardcode the LLM provider — make it configurable (OpenAI / Azure OpenAI / Anthropic) |
| C-5 | Do NOT include PII in logs |
