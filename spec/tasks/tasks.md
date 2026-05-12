# Phase 3 — TASKS: Implementation Breakdown

> Discrete, testable units of work. Each task can be built and tested independently.

---

## Task Dependency Graph

```
TASK-1 (Project Setup)
   │
   ├──► TASK-2 (Excel Reader)
   │       │
   │       └──► TASK-4 (Main Pipeline) ──► TASK-6 (Testing & Scenarios)
   │
   ├──► TASK-3 (LLM Classifier)
   │       │
   │       └──► TASK-4 (Main Pipeline)
   │
   └──► TASK-5 (Excel Writer)
           │
           └──► TASK-4 (Main Pipeline)
```

---

## TASK-1 — Project Setup

**Goal:** Scaffold the project, install dependencies, set up config loading.

### Deliverables
- `requirements.txt` with: pandas, openpyxl, openai, pydantic, python-dotenv
- `src/config.py` — loads `.env` and validates required settings
- `.env.example` — template with all variables
- `data/input/` and `data/output/` directories
- `.gitignore` — ignore `.env`, `data/output/`, `__pycache__`

### Pseudocode — `config.py`

```python
import os
from dotenv import load_dotenv

load_dotenv()

class Config:
    LLM_PROVIDER   = os.getenv("LLM_PROVIDER", "openai")
    LLM_MODEL      = os.getenv("LLM_MODEL", "gpt-4o")
    LLM_API_KEY    = os.getenv("LLM_API_KEY")
    INPUT_EXCEL    = os.getenv("INPUT_EXCEL", "data/input/claims.xlsx")
    OUTPUT_EXCEL   = os.getenv("OUTPUT_EXCEL", "data/output/claims_classified.xlsx")
    CONFIDENCE_THRESHOLD = float(os.getenv("CONFIDENCE_THRESHOLD", "0.70"))

    # Azure-specific (optional)
    AZURE_ENDPOINT = os.getenv("AZURE_OPENAI_ENDPOINT")
    AZURE_API_VER  = os.getenv("AZURE_OPENAI_API_VERSION", "2024-02-15-preview")

def validate_config():
    """Exit early if required settings are missing."""
    if not Config.LLM_API_KEY:
        raise SystemExit("ERROR: LLM_API_KEY not set. Add it to your .env file.")
```

### Test
```
GIVEN  .env has LLM_API_KEY set
WHEN   validate_config() is called
THEN   no error is raised

GIVEN  .env is missing LLM_API_KEY
WHEN   validate_config() is called
THEN   SystemExit is raised with clear message
```

---

## TASK-2 — Excel Reader

**Goal:** Read the input Excel file and return a list of claim dictionaries.

### Deliverables
- `src/excel_handler.py` — function `read_claims(filepath) -> list[dict]`

### Pseudocode — `read_claims()`

```python
import pandas as pd

# These are the text columns we look for in the Excel
TEXT_COLUMNS = [
    "doctor_notes", "incident_report", "transcript",
    "employer_statement", "medical_records"
]

def read_claims(filepath: str) -> list[dict]:
    """
    Read Excel file. Return list of dicts, one per claim row.
    Each dict has: claim_id, claim_date, and a 'documents' dict
    with document_type -> text content.
    """
    df = pd.read_excel(filepath)

    # Validate required columns exist
    if "claim_id" not in df.columns:
        raise ValueError("Excel must have a 'claim_id' column")

    claims = []
    for _, row in df.iterrows():
        # Collect all non-empty text columns into a documents dict
        documents = {}
        for col in TEXT_COLUMNS:
            if col in df.columns and pd.notna(row[col]) and str(row[col]).strip():
                documents[col] = str(row[col]).strip()

        claims.append({
            "claim_id": row["claim_id"],
            "claim_date": row.get("claim_date", ""),
            "documents": documents,        # e.g. {"doctor_notes": "...", "incident_report": "..."}
            "raw_row": row.to_dict()        # keep original row for output
        })

    return claims
```

### Test
```
GIVEN  an Excel with 3 rows and columns: claim_id, claim_date, doctor_notes, incident_report
WHEN   read_claims() is called
THEN   returns a list of 3 dicts, each with claim_id and documents

GIVEN  a row where doctor_notes is empty and incident_report has text
WHEN   that row is processed
THEN   documents = {"incident_report": "..."} (empty columns excluded)

GIVEN  an Excel missing the claim_id column
WHEN   read_claims() is called
THEN   raises ValueError
```

---

## TASK-3 — LLM Classifier

**Goal:** Take a claim's documents, send them to the LLM, and return a structured classification.

### Deliverables
- `src/prompts.py` — system prompt + user prompt builder
- `src/schemas.py` — Pydantic model for the LLM response
- `src/classifier.py` — function `classify_claim(documents: dict) -> ClassificationResult`

### Pseudocode — `schemas.py`

```python
from pydantic import BaseModel, Field
from typing import Optional
from enum import Enum

class Classification(str, Enum):
    WORK_RELATED = "WORK_RELATED"
    NON_WORK_RELATED = "NON_WORK_RELATED"
    UNDETERMINED = "UNDETERMINED"

class SecondaryTag(str, Enum):
    WORK_AGGRAVATED = "WORK_AGGRAVATED"

class ClassificationResult(BaseModel):
    classification: Classification
    secondary_tag: Optional[SecondaryTag] = None
    confidence_score: float = Field(ge=0.0, le=1.0)
    reasoning: str
    key_excerpts: list[str] = []
    conflicts_or_gaps: list[str] = []
```

### Pseudocode — `prompts.py`

```python
SYSTEM_PROMPT = """
You are a healthcare claim injury classification expert.
Your job is to read claim documents and classify the injury.

CLASSIFICATION RULES:
1. WORK_RELATED — injury clearly happened during work hours, on work premises,
   or while performing work duties
2. NON_WORK_RELATED — injury clearly happened outside of work
   - If work AGGRAVATED a non-work injury, add secondary_tag: WORK_AGGRAVATED
3. UNDETERMINED — evidence is conflicting, insufficient, or missing

CONFIDENCE SCORING:
- 0.9-1.0: Strong, consistent evidence from multiple documents
- 0.7-0.9: Good evidence, minor gaps
- 0.5-0.7: Some evidence but notable gaps or minor conflicts
- 0.0-0.5: Very weak evidence, major conflicts, or almost no info

RESPOND WITH ONLY THIS JSON (no markdown, no extra text):
{
  "classification": "WORK_RELATED | NON_WORK_RELATED | UNDETERMINED",
  "secondary_tag": "WORK_AGGRAVATED or null",
  "confidence_score": 0.85,
  "reasoning": "explanation here",
  "key_excerpts": ["verbatim quote from documents"],
  "conflicts_or_gaps": ["any conflicts or missing info"]
}
"""

def build_user_prompt(documents: dict) -> str:
    """
    Build the user message from all available documents.
    documents = {"doctor_notes": "...", "incident_report": "...", ...}
    """
    parts = ["Classify the following healthcare claim injury:\n"]
    for doc_type, content in documents.items():
        label = doc_type.upper().replace("_", " ")
        parts.append(f"--- {label} ---\n{content}\n")
    return "\n".join(parts)
```

### Pseudocode — `classifier.py`

```python
import json
from openai import OpenAI
from src.config import Config
from src.prompts import SYSTEM_PROMPT, build_user_prompt
from src.schemas import ClassificationResult

def get_llm_client():
    """Create OpenAI or Azure OpenAI client based on config."""
    if Config.LLM_PROVIDER == "azure_openai":
        from openai import AzureOpenAI
        return AzureOpenAI(
            api_key=Config.LLM_API_KEY,
            azure_endpoint=Config.AZURE_ENDPOINT,
            api_version=Config.AZURE_API_VER,
        )
    return OpenAI(api_key=Config.LLM_API_KEY)

def classify_claim(documents: dict) -> ClassificationResult:
    """
    Send documents to LLM and return structured classification.
    If documents is empty, return UNDETERMINED immediately.
    """
    # Handle empty documents (no text to classify)
    if not documents:
        return ClassificationResult(
            classification="UNDETERMINED",
            confidence_score=0.0,
            reasoning="No document text provided for this claim.",
            conflicts_or_gaps=["All document columns are empty"],
        )

    client = get_llm_client()
    user_prompt = build_user_prompt(documents)

    # Call LLM
    response = client.chat.completions.create(
        model=Config.LLM_MODEL,
        messages=[
            {"role": "system", "content": SYSTEM_PROMPT},
            {"role": "user", "content": user_prompt},
        ],
        temperature=0.1,           # Low temp for consistent classification
        response_format={"type": "json_object"},  # Force JSON output
    )

    # Parse response
    raw_json = response.choices[0].message.content
    result = ClassificationResult.model_validate_json(raw_json)
    return result
```

### Test
```
GIVEN  documents = {"incident_report": "Employee fell down stairs during work hours"}
WHEN   classify_claim() is called
THEN   returns ClassificationResult with classification = WORK_RELATED

GIVEN  documents = {} (empty)
WHEN   classify_claim() is called
THEN   returns UNDETERMINED with confidence 0.0 (no LLM call made)

GIVEN  LLM returns invalid JSON
WHEN   response is parsed
THEN   error is caught and logged, row marked as ERROR
```

---

## TASK-4 — Main Pipeline

**Goal:** Wire everything together: read Excel → classify each row → write output Excel.

### Deliverables
- `src/main.py` — the entry point script

### Pseudocode — `main.py`

```python
from src.config import Config, validate_config
from src.excel_handler import read_claims, write_results
from src.classifier import classify_claim

def main():
    # Step 1: Validate config
    validate_config()
    print(f"Using model: {Config.LLM_MODEL}")
    print(f"Input:  {Config.INPUT_EXCEL}")
    print(f"Output: {Config.OUTPUT_EXCEL}")

    # Step 2: Read claims from Excel
    claims = read_claims(Config.INPUT_EXCEL)
    print(f"Loaded {len(claims)} claims")

    # Step 3: Classify each claim
    results = []
    for i, claim in enumerate(claims):
        print(f"Classifying claim {i+1}/{len(claims)}: {claim['claim_id']}...")

        try:
            result = classify_claim(claim["documents"])
            results.append({
                **claim["raw_row"],
                "classification": result.classification,
                "secondary_tag": result.secondary_tag or "",
                "confidence_score": result.confidence_score,
                "reasoning": result.reasoning,
                "key_excerpts": " | ".join(result.key_excerpts),
                "conflicts_or_gaps": " | ".join(result.conflicts_or_gaps),
                "needs_human_review": result.confidence_score < Config.CONFIDENCE_THRESHOLD,
            })
        except Exception as e:
            results.append({
                **claim["raw_row"],
                "classification": "ERROR",
                "secondary_tag": "",
                "confidence_score": 0.0,
                "reasoning": f"Error: {str(e)}",
                "key_excerpts": "",
                "conflicts_or_gaps": "",
                "needs_human_review": True,
            })

    # Step 4: Write output Excel
    write_results(results, Config.OUTPUT_EXCEL)
    print(f"Done! Output saved to {Config.OUTPUT_EXCEL}")

if __name__ == "__main__":
    main()
```

### Test
```
GIVEN  an input Excel with 3 claims (one per scenario)
WHEN   main() is run
THEN   output Excel is created with 3 rows
AND    each row has classification, confidence, reasoning columns
AND    Scenario 1 → NON_WORK_RELATED + WORK_AGGRAVATED
AND    Scenario 2 → WORK_RELATED
AND    Scenario 3 → UNDETERMINED
```

---

## TASK-5 — Excel Writer

**Goal:** Write the classified results back to a new Excel file.

### Deliverables
- Add `write_results()` function to `src/excel_handler.py`

### Pseudocode — `write_results()`

```python
import pandas as pd
import os

def write_results(results: list[dict], output_path: str):
    """Write classification results to output Excel."""
    # Ensure output directory exists
    os.makedirs(os.path.dirname(output_path), exist_ok=True)

    df = pd.DataFrame(results)

    # Reorder columns: original columns first, then classification columns last
    classification_cols = [
        "classification", "secondary_tag", "confidence_score",
        "reasoning", "key_excerpts", "conflicts_or_gaps", "needs_human_review"
    ]
    other_cols = [c for c in df.columns if c not in classification_cols]
    df = df[other_cols + classification_cols]

    df.to_excel(output_path, index=False)
```

### Test
```
GIVEN  a list of 2 result dicts with all required fields
WHEN   write_results() is called
THEN   an Excel file is created at the output path
AND    it has the correct number of rows and columns
AND    classification columns appear at the end
```

---

## TASK-6 — Testing with Reference Scenarios

**Goal:** Create a sample input Excel with the 3 reference scenarios and validate end-to-end.

### Deliverables
- `data/input/sample_claims.xlsx` — 3 rows matching the 3 scenarios from the spec
- `tests/test_classifier.py` — unit tests for classifier with mocked LLM
- `tests/test_excel_handler.py` — unit tests for read/write
- `tests/test_e2e.py` — end-to-end test (requires API key)

### Sample Input Excel Content

| claim_id | claim_date | doctor_notes | incident_report | transcript | employer_statement |
|---|---|---|---|---|---|
| CLM-001 | 2024-01-15 | | | Patient developed neck pain after sleeping in an incorrect position at home. Pain worsened while operating machinery at work. | |
| CLM-002 | 2024-02-20 | | Employee fell down the stairs while carrying documents during work hours. | | |
| CLM-003 | 2024-03-10 | Patient reports back pain. Unclear when it started. | | | |

### Expected Output

| claim_id | classification | secondary_tag | confidence_score | needs_human_review |
|---|---|---|---|---|
| CLM-001 | NON_WORK_RELATED | WORK_AGGRAVATED | ≥ 0.70 | false |
| CLM-002 | WORK_RELATED | | ≥ 0.70 | false |
| CLM-003 | UNDETERMINED | | < 0.70 | true |

---

## Implementation Order

| Order | Task | Depends On | Effort |
|---|---|---|---|
| 1 | TASK-1: Project Setup | — | 30 min |
| 2 | TASK-2: Excel Reader | TASK-1 | 1 hour |
| 3 | TASK-3: LLM Classifier | TASK-1 | 2 hours |
| 4 | TASK-5: Excel Writer | TASK-1 | 30 min |
| 5 | TASK-4: Main Pipeline | TASK-2, 3, 5 | 1 hour |
| 6 | TASK-6: Testing | TASK-4 | 1 hour |

**Total estimated effort: ~6 hours**
