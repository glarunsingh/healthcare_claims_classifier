# Plan: Healthcare Claim Injury Classification System

> **Phase 2 — PLAN**
> Translates the specification into a technical architecture. Defines the tech stack, component breakdown, data flow, API contracts, and integration points.
>
> **Governing spec:** [`.spec/specs/claim-classification.md`](../specs/claim-classification.md)

---

## 1. Architecture Overview

The system follows a **modular pipeline architecture** with four discrete layers. Each layer is independently deployable and testable.

```
┌──────────────────────────────────────────────────────────────────────────┐
│                            API GATEWAY                                   │
│  REST API  •  Auth (JWT / API Key)  •  Rate Limiting  •  Input Validation│
└─────────────────────────────┬────────────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
     ┌────────────┐  ┌───────────────┐  ┌───────────────┐
     │ Ingestion  │  │  Review API   │  │  Audit API    │
     │  Service   │  │ (Human Queue) │  │  (Read-only)  │
     └─────┬──────┘  └───────┬───────┘  └───────┬───────┘
           │                 │                   │
           ▼                 │                   │
     ┌────────────┐          │                   │
     │  Document  │          │                   │
     │ Processing │          │                   │
     │  Service   │          │                   │
     └─────┬──────┘          │                   │
           │                 │                   │
           ▼                 │                   │
     ┌────────────┐          │                   │
     │ Classifi-  │          │                   │
     │  cation    │◄─────────┘                   │
     │  Engine    │                              │
     └─────┬──────┘                              │
           │                                     │
           ▼                                     │
     ┌────────────────────────────────────────────┘
     │              DATA LAYER                    │
     │  PostgreSQL (claims, results, events)      │
     │  Object Storage (raw documents)            │
     └────────────────────────────────────────────┘
```

---

## 2. Tech Stack

| Layer | Technology | Rationale |
|---|---|---|
| Language | Python 3.11+ | Mature ML/NLP ecosystem, LLM SDKs, team familiarity |
| Web Framework | FastAPI | Async-first, auto-generated OpenAPI docs, Pydantic validation |
| LLM Integration | LangChain / direct SDK | Prompt management, structured output parsing, model-swappable |
| LLM Provider | OpenAI GPT-4o (default, swappable) | Strong reasoning, structured JSON output mode; model choice is configurable per OQ-02 |
| Database | PostgreSQL 15+ | JSONB for reasoning traces, row-level security, proven at scale |
| Object Storage | S3-compatible (AWS S3 / MinIO) | Raw document storage, encrypted at rest |
| Queue / Background | Celery + Redis (or RQ) | Async classification jobs, retry, dead-letter |
| Auth | JWT (RS256) via existing client IdP | Role-based access control for reviewers, auditors, admins |
| Containerization | Docker + Docker Compose | Reproducible dev/staging environments |
| CI/CD | GitHub Actions | Lint, test, build, deploy pipeline |
| Testing | pytest + pytest-asyncio | Unit, integration, and contract tests |
| Linting / Formatting | Ruff + mypy | Fast linting and type checking |

---

## 3. Component Breakdown

### 3.1 API Gateway (FastAPI)

**Responsibilities:**
- Route incoming requests to internal services
- Authenticate via JWT (RS256); extract `reviewer_id` from token claims
- Rate-limit per client (configurable)
- Validate request payloads via Pydantic schemas
- Return consistent error responses (RFC 7807 Problem Details)

**Key Endpoints:**

| Method | Path | Description | Auth Role |
|---|---|---|---|
| `POST` | `/api/v1/claims` | Submit a claim bundle for classification | `reviewer`, `admin` |
| `GET` | `/api/v1/claims/{claim_id}` | Get classification result for a claim | `reviewer`, `admin` |
| `GET` | `/api/v1/claims/{claim_id}/status` | Get processing status | `reviewer`, `admin` |
| `POST` | `/api/v1/claims/{claim_id}/review` | Submit reviewer override / approval | `reviewer`, `admin` |
| `GET` | `/api/v1/claims/{claim_id}/audit` | Get full audit trail | `auditor`, `admin` |
| `GET` | `/api/v1/queue/human-review` | List claims in HUMAN_REVIEW queue | `reviewer`, `admin` |
| `GET` | `/api/v1/queue/auto-approved` | List claims in AUTO_APPROVED queue | `reviewer`, `admin` |
| `GET` | `/api/v1/health` | Health check / readiness probe | public |

### 3.2 Ingestion Service

**Responsibilities:**
- Accept document bundles (multipart upload or JSON with text content)
- Validate: ≥ 1 document, each has `type` and `content`
- Assign `claim_id` (UUIDv4)
- Store raw documents in object storage
- Store document metadata in PostgreSQL
- Enqueue classification job

**Maps to:** FR-01 (AC-01.1, AC-01.2, AC-01.3)

### 3.3 Document Processing Service

**Responsibilities:**
- Retrieve raw documents from object storage
- Extract / normalize text (strip headers, footers, OCR artifacts)
- Chunk long documents by section or paragraph
- Tag each chunk with its document type for downstream weighting

**Maps to:** FR-01, FR-02

### 3.4 Classification Engine

**Responsibilities:**
- Receive processed document chunks
- Execute the 3-step classification pipeline:
  1. **Root-cause extraction** — Prompt the LLM to identify where/when the injury originated and whether work was a factor
  2. **Evidence weighting** — Cross-reference extracted facts across document types; detect conflicts and gaps
  3. **Verdict synthesis** — Produce the structured `ClassificationResult` with enum, tag, confidence, and reasoning trace
- Record model version with every result
- Route result to the appropriate queue (AUTO_APPROVED or HUMAN_REVIEW)

**Maps to:** FR-02, FR-03, FR-04, FR-06

**Prompt Architecture:**

```
System Prompt
├── Role: "You are a healthcare claim injury classification expert."
├── Classification rules (decision tree from spec §8)
├── Output schema (JSON matching ClassificationResult)
└── Guardrails: "Do not guess. If evidence is insufficient, classify UNDETERMINED."

User Prompt
├── Document 1: {type} — {content}
├── Document 2: {type} — {content}
├── ...
└── Instruction: "Classify this injury. Return structured JSON."
```

**Confidence Score Calculation Factors:**
- Number of document types present vs. expected (completeness)
- Agreement across documents (consistency)
- Specificity of causal evidence (how clearly the origin is described)
- Absence of contradictory signals (conflict-free)

### 3.5 Review Service

**Responsibilities:**
- Serve HUMAN_REVIEW and AUTO_APPROVED queues
- Accept reviewer actions: APPROVE or OVERRIDE (with reason)
- Record ReviewEvent to audit log
- Update claim status

**Maps to:** FR-04 (AC-04.1, AC-04.2, AC-04.3)

### 3.6 Audit Service

**Responsibilities:**
- Expose read-only API for audit trail queries
- Return chronological event log for a given `claim_id`
- No mutations allowed through this service

**Maps to:** FR-05 (AC-05.1, AC-05.2, AC-05.3)

---

## 4. Database Schema

```sql
-- Core tables

CREATE TABLE claim_bundles (
    claim_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    submitted_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    submitted_by    VARCHAR(255) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'PENDING'
                    CHECK (status IN ('PENDING','CLASSIFIED','UNDER_REVIEW','CLOSED')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE documents (
    document_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    claim_id        UUID NOT NULL REFERENCES claim_bundles(claim_id),
    type            VARCHAR(30) NOT NULL
                    CHECK (type IN ('CLAIM_FORM','DOCTOR_NOTE','TRANSCRIPT',
                                    'INCIDENT_REPORT','EMPLOYER_STATEMENT',
                                    'MEDICAL_RECORD')),
    content         TEXT NOT NULL,
    source          VARCHAR(100),
    storage_path    VARCHAR(500),          -- S3 key for raw document
    received_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE classification_results (
    result_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    claim_id        UUID NOT NULL REFERENCES claim_bundles(claim_id),
    classification  VARCHAR(20) NOT NULL
                    CHECK (classification IN ('WORK_RELATED','NON_WORK_RELATED',
                                              'UNDETERMINED')),
    secondary_tag   VARCHAR(20)
                    CHECK (secondary_tag IS NULL OR secondary_tag = 'WORK_AGGRAVATED'),
    confidence_score NUMERIC(4,3) NOT NULL CHECK (confidence_score BETWEEN 0 AND 1),
    reasoning       JSONB NOT NULL,        -- ReasoningTrace as JSON
    model_version   VARCHAR(50) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE review_events (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    claim_id        UUID NOT NULL REFERENCES claim_bundles(claim_id),
    actor           VARCHAR(255) NOT NULL,  -- 'AI' or reviewer_id
    action          VARCHAR(20) NOT NULL
                    CHECK (action IN ('CLASSIFIED','ESCALATED','APPROVED','OVERRIDDEN')),
    classification  VARCHAR(20) NOT NULL,
    secondary_tag   VARCHAR(20),
    confidence_score NUMERIC(4,3),
    reasoning_snapshot JSONB,
    override_reason TEXT,
    model_version   VARCHAR(50),
    timestamp       TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Indexes
CREATE INDEX idx_documents_claim ON documents(claim_id);
CREATE INDEX idx_results_claim ON classification_results(claim_id);
CREATE INDEX idx_events_claim ON review_events(claim_id);
CREATE INDEX idx_events_timestamp ON review_events(timestamp);
CREATE INDEX idx_bundles_status ON claim_bundles(status);
```

---

## 5. API Request / Response Contracts

### 5.1 Submit Claim Bundle

**Request:** `POST /api/v1/claims`

```json
{
  "documents": [
    {
      "type": "INCIDENT_REPORT",
      "content": "Employee fell down stairs while carrying documents...",
      "source": "uploaded"
    },
    {
      "type": "DOCTOR_NOTE",
      "content": "Patient presents with lumbar strain...",
      "source": "ehr_sync"
    }
  ]
}
```

**Response:** `202 Accepted`

```json
{
  "claim_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "status": "PENDING",
  "message": "Claim submitted. Classification in progress.",
  "estimated_completion_seconds": 30
}
```

### 5.2 Get Classification Result

**Response:** `200 OK` on `GET /api/v1/claims/{claim_id}`

```json
{
  "claim_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "status": "CLASSIFIED",
  "classification": "NON_WORK_RELATED",
  "secondary_tag": "WORK_AGGRAVATED",
  "confidence_score": 0.82,
  "reasoning": {
    "key_excerpts": [
      "neck pain after sleeping in an incorrect position at home",
      "pain worsened while operating machinery at work"
    ],
    "logic_applied": "Injury originated at home (sleeping position). Work activities aggravated existing condition (machinery operation). Primary origin is non-work; secondary aggravation from work is noted.",
    "conflicts": [],
    "gaps": ["No employer statement provided"]
  },
  "model_version": "v1.0.0",
  "queue": "AUTO_APPROVED",
  "created_at": "2026-05-12T14:30:00Z"
}
```

### 5.3 Submit Reviewer Override

**Request:** `POST /api/v1/claims/{claim_id}/review`

```json
{
  "action": "OVERRIDDEN",
  "classification": "WORK_RELATED",
  "secondary_tag": null,
  "override_reason": "Employer confirmed injury occurred during supervised work activity."
}
```

### 5.4 Get Audit Trail

**Response:** `200 OK` on `GET /api/v1/claims/{claim_id}/audit`

```json
{
  "claim_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "events": [
    {
      "event_id": "...",
      "actor": "AI",
      "action": "CLASSIFIED",
      "classification": "NON_WORK_RELATED",
      "secondary_tag": "WORK_AGGRAVATED",
      "confidence_score": 0.82,
      "model_version": "v1.0.0",
      "timestamp": "2026-05-12T14:30:00Z"
    },
    {
      "event_id": "...",
      "actor": "reviewer_jane_doe",
      "action": "OVERRIDDEN",
      "classification": "WORK_RELATED",
      "secondary_tag": null,
      "override_reason": "Employer confirmed injury occurred during supervised work activity.",
      "timestamp": "2026-05-12T15:10:00Z"
    }
  ]
}
```

---

## 6. Project Structure

```
healthcare_claims_classifier/
│
├── .spec/                          # SDD artifacts (this folder)
│   ├── specs/
│   │   └── claim-classification.md
│   ├── plans/
│   │   └── claim-classification-plan.md
│   ├── tasks/
│   │   └── claim-classification-tasks.md
│   └── rules/
│       └── sdd-standards.md
│
├── app/                            # Application source code
│   ├── __init__.py
│   ├── main.py                     # FastAPI app entry point
│   ├── config.py                   # Settings (env vars, thresholds)
│   │
│   ├── api/                        # API layer
│   │   ├── __init__.py
│   │   ├── routes/
│   │   │   ├── claims.py           # Claim submission + retrieval
│   │   │   ├── review.py           # Review queue + overrides
│   │   │   └── audit.py            # Audit trail queries
│   │   ├── schemas/                # Pydantic request/response models
│   │   │   ├── claim.py
│   │   │   ├── review.py
│   │   │   └── audit.py
│   │   └── dependencies.py        # Auth, DB session deps
│   │
│   ├── services/                   # Business logic
│   │   ├── __init__.py
│   │   ├── ingestion.py            # Document ingestion service
│   │   ├── document_processing.py  # Text extraction & normalization
│   │   ├── classification.py       # Classification engine (LLM calls)
│   │   ├── review.py               # Review queue management
│   │   └── audit.py                # Audit log writer / reader
│   │
│   ├── models/                     # SQLAlchemy ORM models
│   │   ├── __init__.py
│   │   ├── claim_bundle.py
│   │   ├── document.py
│   │   ├── classification_result.py
│   │   └── review_event.py
│   │
│   ├── prompts/                    # LLM prompt templates
│   │   ├── system_prompt.py
│   │   └── classification_prompt.py
│   │
│   └── core/                       # Cross-cutting concerns
│       ├── __init__.py
│       ├── database.py             # DB connection, session factory
│       ├── security.py             # JWT validation, RBAC
│       └── exceptions.py           # Custom exception classes
│
├── migrations/                     # Alembic DB migrations
│   ├── env.py
│   └── versions/
│
├── tests/                          # Test suite
│   ├── conftest.py
│   ├── unit/
│   │   ├── test_classification.py
│   │   ├── test_ingestion.py
│   │   └── test_review.py
│   ├── integration/
│   │   ├── test_api_claims.py
│   │   ├── test_api_review.py
│   │   └── test_api_audit.py
│   └── fixtures/                   # Test data
│       ├── scenario_work_related.json
│       ├── scenario_non_work_related.json
│       └── scenario_undetermined.json
│
├── docker-compose.yml
├── Dockerfile
├── pyproject.toml
├── .env.example
├── .gitignore
├── README.md
└── userstory.md                    # Original user story (preserved)
```

---

## 7. Security Architecture

| Concern | Approach | Maps to |
|---|---|---|
| Authentication | JWT tokens (RS256) from client IdP; validated on every request | NFR-06 |
| Authorization | Role-based: `reviewer`, `auditor`, `admin` extracted from JWT claims | NFR-06 |
| Encryption at rest | Documents in S3 with SSE-S3 or SSE-KMS; DB with TDE or volume encryption | NFR-05 |
| Encryption in transit | TLS 1.2+ enforced on all endpoints | NFR-05 |
| PII handling | Reasoning traces include excerpts but no stand-alone PII fields; logs redact PII | NFR-09, C-05 |
| Rate limiting | Per-client token bucket at API gateway | NFR-07 |
| Audit immutability | `review_events` table is append-only; no UPDATE/DELETE permissions for application role | FR-05 |

---

## 8. Error Handling Strategy

| Error Scenario | HTTP Status | Error Code | Behavior |
|---|---|---|---|
| Empty document bundle | 422 | `DOCS_REQUIRED` | Reject; no classification attempted |
| Invalid document type | 422 | `INVALID_DOC_TYPE` | Reject with list of valid types |
| Claim not found | 404 | `CLAIM_NOT_FOUND` | — |
| Classification still pending | 200 | — | Return `status: PENDING` |
| LLM call fails (transient) | — | — | Retry up to 3 times with exponential backoff |
| LLM call fails (permanent) | 500 | `CLASSIFICATION_FAILED` | Mark claim as `ERROR`; notify ops |
| Unauthorized | 401 | `UNAUTHORIZED` | — |
| Forbidden (wrong role) | 403 | `FORBIDDEN` | — |

---

## 9. Configuration & Environment Variables

| Variable | Description | Default |
|---|---|---|
| `DATABASE_URL` | PostgreSQL connection string | — (required) |
| `OBJECT_STORAGE_BUCKET` | S3 bucket name for documents | — (required) |
| `LLM_PROVIDER` | `openai` / `azure_openai` / `anthropic` | `openai` |
| `LLM_MODEL` | Model identifier | `gpt-4o` |
| `LLM_API_KEY` | API key for the LLM provider | — (required) |
| `ESCALATION_THRESHOLD` | Confidence below which claims are escalated | `0.70` |
| `JWT_PUBLIC_KEY_URL` | JWKS endpoint for JWT verification | — (required) |
| `MAX_RETRIES_LLM` | LLM call retry count | `3` |
| `LOG_LEVEL` | Application log level | `INFO` |

---

## 10. Testing Strategy

| Level | Scope | Tool | Coverage Target |
|---|---|---|---|
| Unit | Services, prompt logic, confidence calculation | pytest | ≥ 90 % |
| Integration | API endpoints, DB interactions, queue flow | pytest + httpx | ≥ 80 % |
| Contract | API response schemas match spec data models | Pydantic + schemathesis | 100 % of endpoints |
| Scenario | 3 reference scenarios from spec §9 pass end-to-end | pytest | 100 % |
| Performance | Latency ≤ 30 s for 10-document bundle | locust or k6 | NFR-01 |

---

## Plan Version History

| Version | Date | Changes |
|---|---|---|
| 1.0.0 | 2026-05-12 | Initial technical plan |
