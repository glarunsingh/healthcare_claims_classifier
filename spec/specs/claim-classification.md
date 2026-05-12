# Specification: Healthcare Claim Injury Classification System

> **Phase 1 — SPECIFY**
> Defines *what* we are building, *why* it matters, and *what success looks like* — entirely from the user's perspective. No technology choices are made here.

---

## 1. User Story

**As a** healthcare claim reviewer,
**I want** an AI system that reads claim-related documents and classifies an injury as work-related, non-work-related, or undetermined,
**so that** I can process claims faster, with consistent decisions, auditable evidence, and confidence-based escalation for ambiguous cases.

---

## 2. Stakeholders

| Stakeholder | Role | Key Concern |
|---|---|---|
| **Claim Reviewer** | Primary user. Reviews AI output, approves or overrides. | Speed, accuracy, clarity of reasoning |
| **Claims Manager** | Supervises reviewers, monitors queue health. | Throughput, escalation rate, override rate |
| **Compliance Officer** | Audits past decisions for regulatory adherence. | Immutable logs, traceability, data-protection |
| **IT / Engineering** | Builds and operates the system. | Maintainability, security, model versioning |
| **Client (Payer Org)** | Funds the initiative; accountable to regulators. | ROI, risk reduction, legal defensibility |

---

## 3. Success Criteria

| # | Metric | Target |
|---|---|---|
| SC-01 | Time for a reviewer to approve/escalate a classified claim | ≤ 2 minutes |
| SC-02 | False-positive rate (non-work classified as work) | ≤ 5 % on held-out test set |
| SC-03 | False-negative rate (work classified as non-work) | ≤ 5 % on held-out test set |
| SC-04 | Percentage of claims auto-escalated to human review | ≤ 30 % of total volume |
| SC-05 | Every AI decision includes at least one verbatim document excerpt | 100 % |
| SC-06 | Audit trail queryable by claim ID, returning full decision history | 100 % of claims |
| SC-07 | End-to-end classification latency (≤ 10 documents) | ≤ 30 seconds |

---

## 4. Functional Requirements

### FR-01 — Document Ingestion

**As a** claim reviewer,
**I want to** submit a bundle of claim-related documents,
**so that** the system can analyze the full picture of the injury incident.

#### Acceptance Criteria

```
AC-01.1 — Supported document types
GIVEN  a claim bundle is submitted
WHEN   it contains one or more of: claim form, doctor note, conversation
       transcript, incident report, employer statement, medical record
THEN   all documents are accepted and stored against a unique claim ID

AC-01.2 — Empty submission rejected
GIVEN  a claim bundle is submitted
WHEN   zero documents are attached
THEN   the system rejects the submission with error DOCS_REQUIRED
AND    no classification is attempted

AC-01.3 — Partial bundle accepted with reduced confidence
GIVEN  a claim bundle is submitted
WHEN   only a subset of document types is present
THEN   the system proceeds with classification
AND    the confidence score is reduced proportionally to reflect the
       limited evidence set
AND    the reasoning trace lists the missing document types as gaps
```

---

### FR-02 — Injury Origin Analysis

**As a** classification engine,
**I want to** identify the root cause, context, and origin of the reported injury,
**so that** the classification is grounded in evidence, not assumptions.

#### Acceptance Criteria

```
AC-02.1 — Work-related origin
GIVEN  documents describe an injury event
WHEN   the evidence clearly indicates the injury occurred during work hours,
       on work premises, or while performing work duties
THEN   the engine extracts work-context evidence
AND    marks the origin as WORK

AC-02.2 — Non-work-related origin
GIVEN  documents describe an injury event
WHEN   the evidence clearly indicates the injury occurred outside of work
       hours, off work premises, and unrelated to work duties
THEN   the engine extracts non-work-context evidence
AND    marks the origin as NON_WORK

AC-02.3 — Work aggravation of pre-existing / non-work condition
GIVEN  documents indicate an injury originated outside of work
WHEN   subsequent documents show that work activities worsened the condition
THEN   the engine marks primary origin as NON_WORK
AND    attaches the secondary tag WORK_AGGRAVATED
AND    cites the specific aggravation evidence

AC-02.4 — Conflicting or insufficient evidence
GIVEN  documents describe an injury event
WHEN   evidence is contradictory across sources, or insufficient to
       determine origin
THEN   the engine marks origin as UNDETERMINED
AND    lists each specific conflict or gap identified
```

---

### FR-03 — Classification Output

**As a** claim reviewer,
**I want to** receive a structured, machine-readable classification verdict,
**so that** I can quickly understand the AI's decision, the supporting evidence, and any caveats.

#### Acceptance Criteria

```
AC-03.1 — Classification enum
GIVEN  analysis is complete
WHEN   a verdict is produced
THEN   the classification field contains exactly one of:
       WORK_RELATED | NON_WORK_RELATED | UNDETERMINED

AC-03.2 — Secondary tag
GIVEN  the classification is NON_WORK_RELATED
WHEN   work activities are found to have aggravated the injury
THEN   the response includes secondary_tag = WORK_AGGRAVATED

AC-03.3 — Confidence score
GIVEN  a classification is produced
THEN   it includes a confidence_score between 0.00 and 1.00
AND    the score reflects volume, consistency, and specificity of evidence

AC-03.4 — Reasoning trace
GIVEN  a classification is produced
THEN   the response includes a reasoning object containing:
       • key_excerpts   — verbatim text fragments from input documents
       • logic_applied  — narrative of the decision path
       • conflicts      — contradictory signals, if any
       • gaps           — missing evidence, if any
```

---

### FR-04 — Confidence-Based Escalation

**As a** claim reviewer,
**I want to** be automatically notified when the AI is uncertain,
**so that** I focus my manual review time on genuinely ambiguous cases.

#### Acceptance Criteria

```
AC-04.1 — Auto-escalation
GIVEN  a classification is produced
WHEN   confidence_score < configurable_threshold (default 0.70)
       OR classification == UNDETERMINED
THEN   the claim is placed in the HUMAN_REVIEW queue
AND    the reviewer is notified with the classification, confidence, and
       identified gaps

AC-04.2 — High-confidence pass-through
GIVEN  a classification is produced
WHEN   confidence_score >= configurable_threshold
       AND classification != UNDETERMINED
THEN   the claim is placed in the AUTO_APPROVED queue for spot-check review

AC-04.3 — Reviewer override
GIVEN  a claim is in any review queue
WHEN   a reviewer changes the classification
THEN   the override is recorded with: reviewer_id, timestamp, new
       classification, override_reason
AND    the original AI classification is preserved for audit
```

---

### FR-05 — Auditability & Compliance

**As a** compliance officer,
**I want to** query the full decision history for any claim,
**so that** I can reconstruct exactly what happened and why, at any point in the future.

#### Acceptance Criteria

```
AC-05.1 — Immutable event log
GIVEN  any classification event occurs (AI verdict, escalation, human
       approval, human override)
WHEN   the event is persisted
THEN   it is stored as an immutable, append-only record containing:
       claim_id, timestamp, actor (AI | reviewer_id), action, classification,
       confidence_score, reasoning_trace, model_version

AC-05.2 — Audit query
GIVEN  a claim ID
WHEN   an audit query is issued
THEN   the system returns the full chronological event log for that claim
       including all intermediate and override events

AC-05.3 — Retention
GIVEN  any audit record
THEN   it is retained for a minimum of 7 years
AND    cannot be modified or deleted during the retention period
```

---

### FR-06 — Model Versioning

**As an** engineering team,
**I want** every classification result to record the model version used,
**so that** we can trace any result back to the exact model that produced it.

#### Acceptance Criteria

```
AC-06.1 — Version stamp
GIVEN  a classification is produced
THEN   the result includes model_version (e.g., "v1.2.0")
AND    the version matches the deployed model artifact exactly
```

---

## 5. Non-Functional Requirements

| ID | Category | Requirement | Acceptance Criterion |
|---|---|---|---|
| NFR-01 | Performance | End-to-end latency | ≤ 30 s for a bundle of ≤ 10 documents |
| NFR-02 | Accuracy | False-positive rate | ≤ 5 % on held-out test set |
| NFR-03 | Accuracy | False-negative rate | ≤ 5 % on held-out test set |
| NFR-04 | Explainability | Evidence citation | Every classification includes ≥ 1 verbatim excerpt |
| NFR-05 | Security | Encryption | Documents encrypted at rest (AES-256) and in transit (TLS 1.2+) |
| NFR-06 | Security | Access control | Role-based access; only authorized reviewers access claim data |
| NFR-07 | Scalability | Concurrent load | 1,000 concurrent claim submissions without degradation |
| NFR-08 | Auditability | Log retention | Write-once, query-able, retained ≥ 7 years |
| NFR-09 | Privacy | PII handling | Compliant with HIPAA / applicable data-protection regulations |
| NFR-10 | Availability | Uptime | 99.5 % monthly uptime SLA |

---

## 6. Data Models

### 6.1 ClaimBundle

| Field | Type | Description |
|---|---|---|
| `claim_id` | UUID | Unique identifier, assigned at ingestion |
| `submitted_at` | timestamp | Time of submission |
| `submitted_by` | string | Reviewer ID or system actor |
| `documents` | Document[] | One or more documents in the bundle |
| `status` | enum | `PENDING` · `CLASSIFIED` · `UNDER_REVIEW` · `CLOSED` |

### 6.2 Document

| Field | Type | Description |
|---|---|---|
| `document_id` | UUID | Unique identifier |
| `claim_id` | UUID | FK to ClaimBundle |
| `type` | enum | `CLAIM_FORM` · `DOCTOR_NOTE` · `TRANSCRIPT` · `INCIDENT_REPORT` · `EMPLOYER_STATEMENT` · `MEDICAL_RECORD` |
| `content` | text | Raw text content of the document |
| `source` | string | Origin (e.g., "uploaded", "fax_import", "ehr_sync") |
| `received_at` | timestamp | When the document was received |

### 6.3 ClassificationResult

| Field | Type | Description |
|---|---|---|
| `result_id` | UUID | Unique identifier |
| `claim_id` | UUID | FK to ClaimBundle |
| `classification` | enum | `WORK_RELATED` · `NON_WORK_RELATED` · `UNDETERMINED` |
| `secondary_tag` | enum \| null | `WORK_AGGRAVATED` or null |
| `confidence_score` | float | 0.00 – 1.00 |
| `reasoning` | ReasoningTrace | Structured explanation object |
| `model_version` | string | Version of the classification model |
| `created_at` | timestamp | When the classification was produced |

### 6.4 ReasoningTrace

| Field | Type | Description |
|---|---|---|
| `key_excerpts` | string[] | Verbatim text fragments from input documents |
| `logic_applied` | string | Narrative explanation of the decision path |
| `conflicts` | string[] | Contradictory signals across documents, if any |
| `gaps` | string[] | Missing evidence or document types, if any |

### 6.5 ReviewEvent (Audit Log Entry)

| Field | Type | Description |
|---|---|---|
| `event_id` | UUID | Unique identifier |
| `claim_id` | UUID | FK to ClaimBundle |
| `actor` | string | `"AI"` or reviewer_id |
| `action` | enum | `CLASSIFIED` · `ESCALATED` · `APPROVED` · `OVERRIDDEN` |
| `classification` | enum | The classification at this event |
| `secondary_tag` | enum \| null | Tag at this event |
| `confidence_score` | float \| null | Confidence at this event |
| `reasoning_snapshot` | ReasoningTrace \| null | Reasoning at this event |
| `override_reason` | string \| null | Reviewer's reason for override, if applicable |
| `model_version` | string \| null | Model version at this event |
| `timestamp` | timestamp | Immutable event time |

---

## 7. System Flow

```
                         ┌──────────────────────┐
                         │   Claim Reviewer /    │
                         │   External System     │
                         └──────────┬───────────┘
                                    │ submit bundle
                                    ▼
                  ┌──────────────────────────────────┐
                  │        1. INGESTION LAYER         │
                  │  • Validate documents (≥ 1 doc)   │
                  │  • Assign claim_id (UUID)          │
                  │  • Store documents                 │
                  │  • Set status = PENDING             │
                  └───────────────┬──────────────────┘
                                  │
                                  ▼
                  ┌──────────────────────────────────┐
                  │     2. DOCUMENT PROCESSING        │
                  │  • Extract raw text (OCR if needed)│
                  │  • Normalize and clean             │
                  │  • Tag by document type             │
                  └───────────────┬──────────────────┘
                                  │
                                  ▼
                  ┌──────────────────────────────────┐
                  │     3. CLASSIFICATION ENGINE       │
                  │                                    │
                  │  Step A — Root-cause extraction     │
                  │    • Where did injury originate?    │
                  │    • Was work a contributing or     │
                  │      aggravating factor?            │
                  │                                    │
                  │  Step B — Evidence weighting        │
                  │    • Cross-reference documents      │
                  │    • Detect conflicts & gaps        │
                  │                                    │
                  │  Step C — Verdict synthesis         │
                  │    • Assign classification enum     │
                  │    • Attach secondary tag if needed │
                  │    • Compute confidence score       │
                  │    • Build reasoning trace          │
                  │    • Record model version           │
                  └───────────────┬──────────────────┘
                                  │
                      ┌───────────┴───────────┐
                      │                       │
             confidence ≥ threshold    confidence < threshold
             AND ≠ UNDETERMINED         OR UNDETERMINED
                      │                       │
                      ▼                       ▼
              ┌──────────────┐        ┌───────────────┐
              │ AUTO_APPROVED│        │ HUMAN_REVIEW  │
              │    queue     │        │    queue       │
              └──────┬───────┘        └───────┬───────┘
                     │                        │
                     │   reviewer approves    │
                     │   or overrides         │
                     └────────────┬───────────┘
                                  │
                                  ▼
                  ┌──────────────────────────────────┐
                  │          4. AUDIT LOG              │
                  │  • Every event is immutable        │
                  │  • Queryable by claim_id           │
                  │  • Retained ≥ 7 years              │
                  └──────────────────────────────────┘
```

---

## 8. Classification Decision Tree

```
Is there clear evidence the injury occurred DURING WORK
(work hours / work premises / work duties)?
│
├─ YES
│   └─► WORK_RELATED
│       confidence = f(document corroboration, specificity)
│
├─ NO — Is there clear evidence the injury occurred OUTSIDE WORK?
│   │
│   ├─ YES
│   │   └─ Do any documents show that WORK AGGRAVATED the condition?
│   │       │
│   │       ├─ YES ─► NON_WORK_RELATED + secondary_tag: WORK_AGGRAVATED
│   │       │         confidence = f(aggravation evidence strength)
│   │       │
│   │       └─ NO  ─► NON_WORK_RELATED
│   │                 confidence = f(non-work evidence strength)
│   │
│   └─ UNCLEAR / CONFLICTING / INSUFFICIENT
│       └─► UNDETERMINED → auto-escalate to HUMAN_REVIEW
│           confidence = low (reflects ambiguity)
│
└─ INSUFFICIENT EVIDENCE TO DECIDE
    └─► UNDETERMINED → auto-escalate to HUMAN_REVIEW
```

---

## 9. Reference Scenarios (from User Story)

### Scenario 1 — Non-Work-Related Injury with Work Aggravation

> A conversation between patient and doctor indicates the patient initially developed neck pain after sleeping in an incorrect position at home. The pain later worsened while operating machinery at work.

| Field | Expected Value |
|---|---|
| classification | `NON_WORK_RELATED` |
| secondary_tag | `WORK_AGGRAVATED` |
| key_excerpts | "neck pain after sleeping in an incorrect position at home", "pain worsened while operating machinery at work" |
| logic_applied | Injury originated at home (sleeping position); work aggravated existing condition (machinery operation) |

### Scenario 2 — Work-Related Injury

> An incident report states that the employee fell down the stairs while carrying documents during work hours.

| Field | Expected Value |
|---|---|
| classification | `WORK_RELATED` |
| secondary_tag | null |
| key_excerpts | "fell down the stairs while carrying documents during work hours" |
| logic_applied | Injury occurred on work premises, during work hours, while performing work duties |

### Scenario 3 — Undetermined

> Available documents contain conflicting statements or insufficient information.

| Field | Expected Value |
|---|---|
| classification | `UNDETERMINED` |
| secondary_tag | null |
| conflicts | List of contradictory statements identified |
| gaps | List of missing document types or information |

---

## 10. Explicit Constraints — DO NOT

| # | Constraint |
|---|---|
| C-01 | Do NOT make legal determinations or calculate compensation. The system produces a classification *recommendation* only. |
| C-02 | Do NOT support multi-language documents in v1. English only. |
| C-03 | Do NOT integrate with external claims management platforms in v1 (API contract defined separately). |
| C-04 | Do NOT build a model retraining pipeline in v1. |
| C-05 | Do NOT expose PII in logs or API responses beyond what is necessary for the reasoning trace. |
| C-06 | Do NOT allow deletion or mutation of audit log entries. |
| C-07 | Do NOT override human reviewers' final decisions programmatically. |

---

## 11. Assumptions

| # | Assumption |
|---|---|
| A-01 | All documents within a bundle relate to the same claim and injury event. |
| A-02 | Documents are submitted as plain text or can be converted to plain text (OCR / parsing is a pre-processing step). |
| A-03 | Human reviewers have domain expertise to evaluate AI recommendations. |
| A-04 | The escalation confidence threshold is configurable per deployment and may vary by claim type. |
| A-05 | The client will provide labeled historical claims for model evaluation (held-out test set). |

---

## 12. Open Questions

| # | Question | Owner | Status |
|---|---|---|---|
| OQ-01 | Is 0.70 the correct default escalation threshold, or should it be tunable per claim type / jurisdiction? | Product | Open |
| OQ-02 | Which LLM / model family will power the classification engine (GPT-4o, Claude, fine-tuned BERT, etc.)? | Engineering | Open |
| OQ-03 | Are there specific regulatory requirements (HIPAA, state workers' comp statutes) that constrain storage, sharing, or retention? | Legal / Compliance | Open |
| OQ-04 | What is the expected average document count and length per bundle? Impacts latency and chunking strategy. | Product / Engineering | Open |
| OQ-05 | Should WORK_AGGRAVATED ever escalate the classification to WORK_RELATED under certain jurisdictions? | Legal | Open |
| OQ-06 | Will the system receive structured data (JSON) or unstructured documents (PDF, DOCX, images)? Impacts document processing layer. | Engineering | Open |
| OQ-07 | What authentication / SSO provider will reviewers use? | Client IT | Open |
| OQ-08 | Is there an existing data warehouse or logging platform the audit log should feed into? | Client IT | Open |

---

## Spec Version History

| Version | Date | Changes |
|---|---|---|
| 1.0.0 | 2026-05-12 | Initial specification derived from user story |
