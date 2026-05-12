# Spec — Healthcare Claim Injury Classification System

> **Spec-Driven Development artifact.**
> This document is the living source of truth for the classification system. All design decisions, task breakdowns, and AI-generated code must trace back to requirements defined here.

---

## 1. Overview

**What we are building**
An AI-powered classification pipeline that ingests healthcare claim documents and determines whether an injury is *work-related*, *non-work-related*, or *undetermined*, with supporting confidence scores and reasoning traces.

**Why it matters**
Manual claim review is slow, inconsistent, and hard to audit. This system reduces that effort while keeping a human reviewer in the loop for ambiguous cases.

**Success looks like**
- A claim reviewer can open a dashboard, see the AI classification and the supporting evidence, and either approve or escalate it in under 2 minutes.
- The system flags its own uncertainty instead of silently guessing.
- Every decision is auditable — reviewers can see exactly which document excerpts drove a classification.

---

## 2. Actors

| Actor | Description |
|---|---|
| **Claim Reviewer** | Human who validates or overrides AI classifications |
| **Classification Engine** | AI subsystem that analyzes documents and produces a verdict |
| **Audit System** | Downstream consumer of all classification events and their rationale |

---

## 3. Functional Requirements

Each requirement is expressed as a user story with GIVEN / WHEN / THEN acceptance criteria.

---

### REQ-01 — Document Ingestion

**As a** claim reviewer,
**I want to** submit a bundle of claim-related documents (claim forms, doctor notes, transcripts, incident reports, employer statements, medical records)
**so that** the system can analyze the full picture of the incident.

#### Acceptance Criteria

**AC-01.1** — Supported document types
```
GIVEN a claim bundle is submitted
WHEN it contains one or more of: claim form, doctor note, conversation transcript,
     incident report, employer statement, medical record
THEN all submitted documents are accepted and stored against the claim ID
```

**AC-01.2** — Missing documents
```
GIVEN a claim bundle is submitted
WHEN zero documents are attached
THEN the system rejects the submission with error code DOCS_REQUIRED
AND no classification is attempted
```

**AC-01.3** — Partial bundle
```
GIVEN a claim bundle is submitted
WHEN only some document types are present
THEN the system proceeds with classification
AND the confidence score reflects the reduced evidence set
```

---

### REQ-02 — Injury Origin Analysis

**As a** classification engine,
**I want to** identify the root cause and origin context of the reported injury
**so that** the classification is grounded in evidence rather than assumptions.

#### Acceptance Criteria

**AC-02.1** — Workplace origin detected
```
GIVEN documents describe an injury event
WHEN the incident clearly occurred during work hours, on work premises,
     or while performing work duties
THEN the engine extracts the work-context evidence and marks origin as WORK
```

**AC-02.2** — Non-workplace origin detected
```
GIVEN documents describe an injury event
WHEN the incident clearly occurred outside work hours, off work premises,
     and unrelated to work duties
THEN the engine extracts the non-work-context evidence and marks origin as NON_WORK
```

**AC-02.3** — Work aggravation of pre-existing condition
```
GIVEN documents indicate an injury that originated outside work
WHEN subsequent documents indicate work activities worsened the condition
THEN the engine marks primary origin as NON_WORK
AND attaches secondary tag WORK_AGGRAVATED with supporting evidence
```

**AC-02.4** — Conflicting or insufficient evidence
```
GIVEN documents describe an injury event
WHEN evidence is contradictory across sources, or insufficient to determine origin
THEN the engine marks origin as UNDETERMINED
AND lists the specific conflicts or gaps identified
```

---

### REQ-03 — Classification Output

**As a** claim reviewer,
**I want to** receive a structured classification verdict
**so that** I can quickly understand the AI's decision and the evidence behind it.

#### Acceptance Criteria

**AC-03.1** — Classification enum
```
GIVEN the analysis is complete
WHEN a verdict is produced
THEN the classification field contains exactly one of:
     WORK_RELATED | NON_WORK_RELATED | UNDETERMINED
```

**AC-03.2** — Secondary tag
```
GIVEN the classification is NON_WORK_RELATED
WHEN work activities are found to have aggravated the injury
THEN the response includes secondary_tag: WORK_AGGRAVATED
```

**AC-03.3** — Confidence score
```
GIVEN a classification is produced
THEN it includes a confidence_score between 0.00 and 1.00
AND the score reflects the volume, consistency, and specificity of supporting evidence
```

**AC-03.4** — Reasoning trace
```
GIVEN a classification is produced
THEN the response includes a reasoning field listing:
     - the key document excerpts used
     - the logic applied to reach the verdict
     - any contradictory signals and how they were resolved
```

---

### REQ-04 — Confidence-Based Escalation

**As a** claim reviewer,
**I want to** be automatically notified when the AI is uncertain
**so that** I spend my review time on genuinely ambiguous cases.

#### Acceptance Criteria

**AC-04.1** — Auto-escalation threshold
```
GIVEN a classification is produced
WHEN confidence_score < 0.70 OR classification == UNDETERMINED
THEN the claim is placed in the HUMAN_REVIEW queue
AND the reviewer is notified with the classification and the identified gaps
```

**AC-04.2** — High-confidence pass-through
```
GIVEN a classification is produced
WHEN confidence_score >= 0.70 AND classification != UNDETERMINED
THEN the claim is placed in the AUTO_APPROVED queue for spot-check review
```

**AC-04.3** — Reviewer override
```
GIVEN a claim is in any review queue
WHEN a reviewer changes the classification
THEN the override is recorded with the reviewer ID and timestamp
AND the original AI classification is preserved for audit
```

---

### REQ-05 — Auditability

**As a** compliance officer,
**I want to** query the full decision history for any claim
**so that** I can reconstruct exactly what happened and why.

#### Acceptance Criteria

**AC-05.1** — Immutable event log
```
GIVEN any classification event occurs (AI verdict, override, escalation)
WHEN the event is written
THEN it is stored as an immutable record with: claim_id, timestamp, actor,
     action, classification, confidence_score, reasoning_trace
```

**AC-05.2** — Audit query
```
GIVEN a claim ID
WHEN an audit query is issued
THEN the system returns the full chronological event log for that claim
```

---

## 4. Non-Functional Requirements

| ID | Requirement | Acceptance Criterion |
|---|---|---|
| NFR-01 | Latency | Classification result returned within 30 seconds of submission for a bundle of ≤ 10 documents |
| NFR-02 | Accuracy | False-positive rate (non-work misclassified as work) ≤ 5 % on held-out test set |
| NFR-03 | Accuracy | False-negative rate (work misclassified as non-work) ≤ 5 % on held-out test set |
| NFR-04 | Explainability | Every classification includes at least one verbatim document excerpt as evidence |
| NFR-05 | Security | Documents are encrypted at rest and in transit; access is role-restricted |
| NFR-06 | Scalability | System handles 1,000 concurrent claim submissions without degradation |
| NFR-07 | Auditability | Audit log is write-once, query-able, and retained for 7 years |

---

## 5. Data Models

### ClaimBundle
```
claim_id        : UUID          -- unique identifier assigned at ingestion
submitted_at    : timestamp
documents       : Document[]    -- one or more documents
```

### Document
```
document_id     : UUID
claim_id        : UUID
type            : CLAIM_FORM | DOCTOR_NOTE | TRANSCRIPT
                  | INCIDENT_REPORT | EMPLOYER_STATEMENT | MEDICAL_RECORD
content         : text
source          : string        -- e.g. "uploaded by reviewer", "fax import"
```

### ClassificationResult
```
result_id           : UUID
claim_id            : UUID
classification      : WORK_RELATED | NON_WORK_RELATED | UNDETERMINED
secondary_tag       : WORK_AGGRAVATED | null
confidence_score    : float (0.00 – 1.00)
reasoning           : ReasoningTrace
created_at          : timestamp
model_version       : string
```

### ReasoningTrace
```
key_excerpts        : string[]  -- verbatim text fragments from documents
logic_applied       : string    -- narrative explanation of the decision path
conflicts_identified: string[]  -- contradictory signals, if any
gaps_identified     : string[]  -- missing evidence, if any
```

### ReviewEvent
```
event_id            : UUID
claim_id            : UUID
actor               : "AI" | reviewer_id
action              : CLASSIFIED | ESCALATED | APPROVED | OVERRIDDEN
classification      : WORK_RELATED | NON_WORK_RELATED | UNDETERMINED
confidence_score    : float | null
notes               : string | null
timestamp           : timestamp
```

---

## 6. System Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                        INGESTION LAYER                          │
│  Claim bundle submitted → validate → assign claim_id → store   │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                     DOCUMENT PROCESSING                         │
│  Extract text → normalize → chunk by document type             │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                    CLASSIFICATION ENGINE                        │
│                                                                 │
│  1. Root-cause extraction                                       │
│     └─ Where did the injury originate?                          │
│     └─ Was work a contributing or aggravating factor?           │
│                                                                 │
│  2. Evidence weighting                                          │
│     └─ Cross-reference across document types                   │
│     └─ Detect conflicts and gaps                                │
│                                                                 │
│  3. Verdict synthesis                                           │
│     └─ Assign classification enum                               │
│     └─ Attach secondary tag if applicable                       │
│     └─ Compute confidence score                                 │
│     └─ Build reasoning trace                                    │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                    ┌───────────┴───────────┐
                    │                       │
          confidence ≥ 0.70         confidence < 0.70
          AND not UNDETERMINED       OR UNDETERMINED
                    │                       │
                    ▼                       ▼
           AUTO_APPROVED             HUMAN_REVIEW
               queue                    queue
                    │                       │
                    └───────────┬───────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                        AUDIT LOG                                │
│  Every event (AI verdict, escalation, override) written as      │
│  immutable record                                               │
└─────────────────────────────────────────────────────────────────┘
```

---

## 7. Classification Decision Tree

```
Does the evidence clearly show the injury occurred DURING WORK / ON WORK PREMISES?
│
├── YES ──► WORK_RELATED (confidence based on document corroboration)
│
├── NO
│    │
│    └── Does the evidence clearly show the injury occurred OUTSIDE WORK?
│         │
│         ├── YES
│         │    │
│         │    └── Do any documents indicate work AGGRAVATED the condition?
│         │         │
│         │         ├── YES ──► NON_WORK_RELATED + secondary_tag: WORK_AGGRAVATED
│         │         │
│         │         └── NO  ──► NON_WORK_RELATED
│         │
│         └── UNCLEAR / CONFLICTING / INSUFFICIENT EVIDENCE
│                   │
│                   └──► UNDETERMINED → auto-escalate to HUMAN_REVIEW
```

---

## 8. Constraints & Assumptions

| # | Constraint / Assumption |
|---|---|
| C-01 | Documents are provided in English. Multi-language support is out of scope for v1. |
| C-02 | The system does not make legal determinations; it produces a classification recommendation only. |
| C-03 | All documents within a bundle relate to the same claim and the same injury event. |
| C-04 | The AI model used by the classification engine is versioned; the version is recorded with every result. |
| C-05 | Human reviewers have the final authority; AI classifications are advisory. |
| C-06 | PII (names, dates of birth, addresses) is present in documents and must be handled per applicable data-protection regulations. |

---

## 9. Out of Scope (v1)

- Multi-language document support
- Automated legal determination or compensation calculation
- Integration with claims management platforms (API contract will be defined in a separate integration spec)
- Retraining pipeline for the classification model

---

## 10. Open Questions

| # | Question | Owner | Status |
|---|---|---|---|
| OQ-01 | What is the target confidence threshold for escalation — is 0.70 correct, or should it be tunable per claim type? | Product | Open |
| OQ-02 | Which specific LLM / model will power the classification engine? | Engineering | Open |
| OQ-03 | Are there regulatory requirements (e.g., HIPAA, state workers' comp rules) that constrain how classifications are stored or shared? | Legal / Compliance | Open |
| OQ-04 | What is the expected average document length per bundle? This affects latency and chunking strategy. | Engineering | Open |
| OQ-05 | Should WORK_AGGRAVATED ever elevate the classification from NON_WORK_RELATED to WORK_RELATED under certain jurisdictions? | Legal | Open |
