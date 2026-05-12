# Phase 2 — PLAN: Technical Architecture

> How we'll build it — tech choices, component design, and data flow.

---

## Architecture (POC — Keep It Simple)

```
┌─────────────────────────────────────────────────────┐
│                    main.py                           │
│                                                     │
│  1. Load Excel (pandas)                              │
│  2. For each row:                                    │
│     a. Extract all text columns into a document bundle│
│     b. Build LLM prompt (system + user message)      │
│     c. Call LLM API                                  │
│     d. Parse structured JSON response                │
│     e. Append classification columns to row          │
│  3. Save output Excel                                │
│                                                     │
└─────────────────────────────────────────────────────┘
```

This is a **single Python script** with helper modules. No web server, no database, no queue.

---

## Tech Stack

| Component | Choice | Why |
|---|---|---|
| Language | Python 3.11+ | Best LLM ecosystem, pandas for Excel |
| Excel I/O | `pandas` + `openpyxl` | Read/write `.xlsx` natively |
| LLM Client | `openai` SDK (works with Azure OpenAI too) | Structured JSON output mode |
| Config | `.env` file + `python-dotenv` | Simple, no framework needed |
| Output parsing | `pydantic` | Validate LLM output matches expected schema |

### Dependencies (requirements.txt)

```
pandas>=2.0
openpyxl>=3.1
openai>=1.30
pydantic>=2.0
python-dotenv>=1.0
```

---

## Project Structure

```
healthcare_claims_classifier/
├── spec/                          # SDD documents (this folder)
│   ├── specs/
│   │   └── claim-classification.md
│   ├── plans/
│   │   └── plan.md
│   ├── tasks/
│   │   └── tasks.md
│   └── rules/
│       └── rules.md
│
├── src/                           # Source code
│   ├── main.py                    # Entry point — run this
│   ├── excel_handler.py           # Read input Excel, write output Excel
│   ├── classifier.py              # LLM prompt + API call + response parsing
│   ├── prompts.py                 # System and user prompt templates
│   ├── schemas.py                 # Pydantic models for output validation
│   └── config.py                  # Load .env settings
│
├── data/                          # Input/output Excel files
│   ├── input/                     # Place input Excel here
│   └── output/                    # Output Excel generated here
│
├── tests/                         # Test files
│   ├── test_classifier.py
│   └── test_excel_handler.py
│
├── .env.example                   # Environment variable template
├── requirements.txt
├── userstory.md
└── README.md
```

---

## Classification Mechanism — How the AI Decides

The classifier uses a **prompt-based approach** with an LLM. Here's the decision logic encoded in the system prompt:

### Decision Tree (Given to the LLM)

```
Step 1: Read ALL document text for this claim.

Step 2: Identify WHEN and WHERE the injury occurred.
        Look for: dates, times, locations, activities being performed.

Step 3: Apply this decision tree:

        Is there evidence the injury occurred DURING WORK?
        (work hours / work premises / work duties)
        │
        ├─ YES, clear evidence
        │   └─► Classify: WORK_RELATED
        │
        ├─ NO — evidence shows injury occurred OUTSIDE WORK
        │   └─ Did work activities WORSEN the condition?
        │       ├─ YES ─► Classify: NON_WORK_RELATED
        │       │         Tag: WORK_AGGRAVATED
        │       └─ NO  ─► Classify: NON_WORK_RELATED
        │
        └─ UNCLEAR / CONFLICTING / NOT ENOUGH INFO
            └─► Classify: UNDETERMINED

Step 4: Assign a confidence score (0.0 to 1.0) based on:
        - How many document types are present (more = higher)
        - Whether documents agree with each other (agreement = higher)
        - How specific the evidence is (specific = higher)

Step 5: Extract verbatim quotes that support your decision.
```

### LLM Output Schema (JSON)

The LLM is instructed to return **structured JSON** matching this exact schema:

```json
{
  "classification": "WORK_RELATED | NON_WORK_RELATED | UNDETERMINED",
  "secondary_tag": "WORK_AGGRAVATED | null",
  "confidence_score": 0.85,
  "reasoning": "Plain English explanation of the decision",
  "key_excerpts": ["verbatim quote 1", "verbatim quote 2"],
  "conflicts_or_gaps": ["conflict or gap 1", "gap 2"]
}
```

---

## Configuration (.env)

```bash
# LLM Settings
LLM_PROVIDER=openai              # openai | azure_openai
LLM_MODEL=gpt-4o                 # model name
LLM_API_KEY=sk-...               # your API key

# Azure OpenAI (only if LLM_PROVIDER=azure_openai)
AZURE_OPENAI_ENDPOINT=https://your-resource.openai.azure.com/
AZURE_OPENAI_API_VERSION=2024-02-15-preview

# File paths
INPUT_EXCEL=data/input/claims.xlsx
OUTPUT_EXCEL=data/output/claims_classified.xlsx

# Classification Settings
CONFIDENCE_THRESHOLD=0.70         # Below this → flag for human review
```

---

## Error Handling (Simple for POC)

| Scenario | Behavior |
|---|---|
| Row has no text in any document column | Classify as `UNDETERMINED`, confidence `0.0`, reasoning: "No document text provided" |
| LLM API call fails | Retry once after 5 seconds. If still fails, mark row as `ERROR` with error message |
| LLM returns invalid JSON | Mark row as `ERROR`, log the raw response |
| Excel file not found | Exit with clear error message |
| Missing API key | Exit with clear error message at startup |
