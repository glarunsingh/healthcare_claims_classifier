# Tasks: Healthcare Claim Injury Classification System

> **Phase 3 — TASKS**
> Breaks the plan into discrete, testable, independently-implementable work units. Each task maps back to specific functional requirements and acceptance criteria from the spec.
>
> **Governing spec:** [`.spec/specs/claim-classification.md`](../specs/claim-classification.md)
> **Governing plan:** [`.spec/plans/claim-classification-plan.md`](../plans/claim-classification-plan.md)

---

## Task Graph — Dependency Overview

```
TASK-01 (Project Bootstrap)
   │
   ├──► TASK-02 (Database Models & Migrations)
   │       │
   │       ├──► TASK-04 (Ingestion Service)
   │       │       │
   │       │       └──► TASK-05 (Document Processing)
   │       │               │
   │       │               └──► TASK-06 (Classification Engine)
   │       │                       │
   │       │                       ├──► TASK-07 (Escalation Routing)
   │       │                       │       │
   │       │                       │       └──► TASK-08 (Review Queue & Overrides)
   │       │                       │
   │       │                       └──► TASK-09 (Audit Log Service)
   │       │
   │       └──► TASK-03 (Pydantic Schemas)
   │               │
   │               └──► TASK-04, TASK-08, TASK-09 (all API routes)
   │
   └──► TASK-10 (Auth & RBAC Middleware)
           │
           └──► all API routes
   
TASK-11 (Integration Tests & Scenarios) — depends on TASK-04 through TASK-09
TASK-12 (Docker & CI/CD) — depends on TASK-01, can run in parallel with TASK-03+
TASK-13 (Configuration & Env Management) — depends on TASK-01
```

---

## TASK-01 — Project Bootstrap

**Priority:** P0 — Foundation
**Depends on:** None
**Estimated complexity:** Small
**Maps to:** Plan §6 (Project Structure)

### Description
Initialize the Python project with FastAPI, configure tooling, and establish the project directory structure.

### Deliverables
- [ ] `pyproject.toml` with dependencies: `fastapi`, `uvicorn`, `sqlalchemy`, `alembic`, `psycopg2-binary`, `pydantic`, `httpx`, `openai`, `boto3`, `python-jose`, `celery`, `redis`
- [ ] Dev dependencies: `pytest`, `pytest-asyncio`, `ruff`, `mypy`, `httpx`
- [ ] Directory scaffold matching Plan §6
- [ ] `app/main.py` with basic FastAPI app instantiation
- [ ] `app/config.py` with Pydantic `Settings` class loading from environment
- [ ] `.env.example` with all required env vars documented
- [ ] `.gitignore` for Python projects

### Acceptance Test
```
GIVEN  the project is cloned and dependencies installed
WHEN   `uvicorn app.main:app --reload` is run
THEN   the server starts on port 8000
AND    GET /api/v1/health returns {"status": "ok"}
```

---

## TASK-02 — Database Models & Migrations

**Priority:** P0 — Foundation
**Depends on:** TASK-01
**Estimated complexity:** Medium
**Maps to:** Spec §6 (Data Models), Plan §4 (Database Schema)

### Description
Create SQLAlchemy ORM models for all four entities and set up Alembic migrations.

### Deliverables
- [ ] `app/models/claim_bundle.py` — ClaimBundle ORM model
- [ ] `app/models/document.py` — Document ORM model
- [ ] `app/models/classification_result.py` — ClassificationResult ORM model with JSONB `reasoning` field
- [ ] `app/models/review_event.py` — ReviewEvent ORM model (append-only semantics enforced at application layer)
- [ ] `app/core/database.py` — Async engine, session factory, `get_db` dependency
- [ ] `migrations/` — Alembic config + initial migration creating all tables and indexes
- [ ] All CHECK constraints from Plan §4 are encoded in the models

### Acceptance Test
```
GIVEN  a running PostgreSQL instance
WHEN   `alembic upgrade head` is run
THEN   all four tables are created with correct columns, types, and constraints
AND    indexes on claim_id, timestamp, and status exist

GIVEN  a claim_bundle row exists
WHEN   a document row is inserted with a non-existent claim_id
THEN   the DB raises a foreign key constraint error
```

---

## TASK-03 — Pydantic Request/Response Schemas

**Priority:** P0 — Foundation
**Depends on:** TASK-01
**Estimated complexity:** Small
**Maps to:** Spec §6 (Data Models), Plan §5 (API Contracts)

### Description
Define Pydantic v2 models for all API request and response payloads. These models are the contract between the API and its consumers.

### Deliverables
- [ ] `app/api/schemas/claim.py`:
  - `DocumentInput` (type: enum, content: str, source: optional str)
  - `ClaimSubmitRequest` (documents: list[DocumentInput], min_length=1)
  - `ClaimSubmitResponse` (claim_id, status, message)
  - `ClassificationResultResponse` (full result with reasoning)
  - `ClaimStatusResponse` (claim_id, status)
- [ ] `app/api/schemas/review.py`:
  - `ReviewActionRequest` (action: enum, classification: enum, override_reason: optional str)
  - `ReviewActionResponse` (event_id, claim_id, status)
- [ ] `app/api/schemas/audit.py`:
  - `AuditEventResponse` (single event)
  - `AuditTrailResponse` (claim_id, events: list)
- [ ] Enum definitions: `DocumentType`, `Classification`, `SecondaryTag`, `ClaimStatus`, `ReviewAction`

### Acceptance Test
```
GIVEN  a ClaimSubmitRequest with zero documents
WHEN   Pydantic validation runs
THEN   a ValidationError is raised with message indicating min 1 document

GIVEN  a ClassificationResultResponse is constructed
WHEN   confidence_score is 1.5
THEN   a ValidationError is raised (must be between 0 and 1)
```

---

## TASK-04 — Ingestion Service & API Route

**Priority:** P1 — Core Flow
**Depends on:** TASK-02, TASK-03
**Estimated complexity:** Medium
**Maps to:** FR-01 (AC-01.1, AC-01.2, AC-01.3)

### Description
Build the ingestion endpoint that accepts document bundles, stores them, and enqueues a classification job.

### Deliverables
- [ ] `app/services/ingestion.py`:
  - `create_claim(documents)` → creates ClaimBundle, stores Documents, returns claim_id
  - Validates ≥ 1 document (AC-01.2)
  - Enqueues async classification task
- [ ] `app/api/routes/claims.py`:
  - `POST /api/v1/claims` — calls ingestion service, returns 202
  - `GET /api/v1/claims/{claim_id}` — returns classification result or status
  - `GET /api/v1/claims/{claim_id}/status` — returns current status
- [ ] Object storage integration: store raw document content in S3 bucket

### Acceptance Criteria Mapping
| AC | Test |
|---|---|
| AC-01.1 | Submit bundle with 3 different document types → all stored, claim_id returned |
| AC-01.2 | Submit bundle with 0 documents → 422 `DOCS_REQUIRED` |
| AC-01.3 | Submit bundle with only 1 document type → classification proceeds, gaps listed |

### Acceptance Test
```
GIVEN  a valid POST to /api/v1/claims with 2 documents
THEN   response is 202 with claim_id and status PENDING
AND    documents are stored in DB and object storage
AND    a classification job is enqueued
```

---

## TASK-05 — Document Processing Service

**Priority:** P1 — Core Flow
**Depends on:** TASK-04
**Estimated complexity:** Small
**Maps to:** FR-01, FR-02

### Description
Process raw documents: normalize text, handle chunking for long documents, and prepare structured input for the classification engine.

### Deliverables
- [ ] `app/services/document_processing.py`:
  - `process_documents(claim_id)` → retrieves documents, normalizes text, returns processed chunks
  - Text normalization: strip excessive whitespace, normalize unicode
  - Chunking: split documents exceeding token limit by paragraph/section
  - Each chunk retains its `document_type` for downstream weighting
- [ ] Handle edge cases: empty content after normalization, very long single-paragraph documents

### Acceptance Test
```
GIVEN  a document with 50,000 characters
WHEN   processed
THEN   it is split into chunks, each ≤ configured token limit
AND    each chunk retains the original document_type label

GIVEN  a document with excessive whitespace and unicode artifacts
WHEN   processed
THEN   the output text is clean and normalized
```

---

## TASK-06 — Classification Engine

**Priority:** P0 — Core Flow
**Depends on:** TASK-05
**Estimated complexity:** Large
**Maps to:** FR-02 (AC-02.1 – AC-02.4), FR-03 (AC-03.1 – AC-03.4), FR-06 (AC-06.1)

### Description
Build the core AI classification engine: prompt construction, LLM invocation, structured output parsing, confidence scoring, and result persistence.

### Deliverables
- [ ] `app/prompts/system_prompt.py`:
  - System prompt encoding the classification decision tree (Spec §8)
  - Output schema instruction (JSON matching ClassificationResult)
  - Guardrails: do not guess, use UNDETERMINED when evidence is insufficient
- [ ] `app/prompts/classification_prompt.py`:
  - User prompt template: injects document type + content per document
  - Final instruction requesting structured JSON output
- [ ] `app/services/classification.py`:
  - `classify_claim(claim_id)`:
    1. Call `process_documents(claim_id)`
    2. Build prompt with system + user messages
    3. Call LLM with structured output mode (JSON mode / function calling)
    4. Parse response into `ClassificationResult`
    5. Validate parsed result (enum values, confidence range, non-empty excerpts)
    6. Persist result to `classification_results` table
    7. Write `CLASSIFIED` event to `review_events`
    8. Return result for routing
  - Model version recorded from config
  - Retry logic for transient LLM failures (up to MAX_RETRIES_LLM)
- [ ] Confidence score includes completeness factor (AC-01.3)

### Acceptance Criteria Mapping
| AC | Test |
|---|---|
| AC-02.1 | Work-related scenario → classification = WORK_RELATED |
| AC-02.2 | Non-work scenario → classification = NON_WORK_RELATED |
| AC-02.3 | Aggravation scenario → NON_WORK_RELATED + WORK_AGGRAVATED |
| AC-02.4 | Conflicting docs → UNDETERMINED |
| AC-03.1 | Output classification is one of the three valid enums |
| AC-03.2 | Secondary tag present only when applicable |
| AC-03.3 | Confidence score between 0 and 1 |
| AC-03.4 | Reasoning includes key_excerpts, logic_applied, conflicts, gaps |
| AC-06.1 | model_version is stamped on every result |

### Acceptance Test
```
GIVEN  Scenario 1 documents (neck pain from sleeping, aggravated by machinery)
WHEN   classify_claim() is called
THEN   classification = NON_WORK_RELATED
AND    secondary_tag = WORK_AGGRAVATED
AND    key_excerpts contain relevant verbatim text
AND    model_version matches configured version

GIVEN  Scenario 2 documents (fell down stairs at work)
WHEN   classify_claim() is called
THEN   classification = WORK_RELATED
AND    secondary_tag is null
```

---

## TASK-07 — Escalation & Queue Routing

**Priority:** P1 — Core Flow
**Depends on:** TASK-06
**Estimated complexity:** Small
**Maps to:** FR-04 (AC-04.1, AC-04.2)

### Description
Route classified claims to the appropriate queue based on confidence and classification.

### Deliverables
- [ ] Routing logic in `app/services/classification.py` (post-classification step):
  - If `confidence_score < ESCALATION_THRESHOLD` OR `classification == UNDETERMINED`:
    - Set claim status to `UNDER_REVIEW`
    - Write `ESCALATED` event to audit log
  - Else:
    - Set claim status to `CLASSIFIED` (auto-approved queue)
- [ ] Configurable threshold via `ESCALATION_THRESHOLD` env var (default 0.70)

### Acceptance Criteria Mapping
| AC | Test |
|---|---|
| AC-04.1 | Confidence 0.55 → claim goes to UNDER_REVIEW (human review queue) |
| AC-04.1 | UNDETERMINED classification → claim goes to UNDER_REVIEW regardless of confidence |
| AC-04.2 | Confidence 0.85 + WORK_RELATED → claim stays in CLASSIFIED (auto-approved) |

### Acceptance Test
```
GIVEN  a classification result with confidence_score = 0.55
WHEN   routing runs
THEN   claim status is UNDER_REVIEW
AND    an ESCALATED event is written to review_events

GIVEN  a classification result with confidence_score = 0.90
WHEN   routing runs
THEN   claim status is CLASSIFIED
AND    no ESCALATED event is written
```

---

## TASK-08 — Review Queue & Override API

**Priority:** P1 — Core Flow
**Depends on:** TASK-07, TASK-03, TASK-10
**Estimated complexity:** Medium
**Maps to:** FR-04 (AC-04.3)

### Description
Build the review queue endpoints and the ability for reviewers to approve or override AI classifications.

### Deliverables
- [ ] `app/services/review.py`:
  - `get_review_queue(queue_type)` → returns claims in UNDER_REVIEW or CLASSIFIED
  - `submit_review(claim_id, action, classification, override_reason, reviewer_id)`:
    - If APPROVED: write APPROVED event, set claim status to CLOSED
    - If OVERRIDDEN: write OVERRIDDEN event with new classification + reason, set status to CLOSED
    - Original AI classification preserved (never mutated)
- [ ] `app/api/routes/review.py`:
  - `GET /api/v1/queue/human-review` — paginated list of UNDER_REVIEW claims
  - `GET /api/v1/queue/auto-approved` — paginated list of CLASSIFIED claims
  - `POST /api/v1/claims/{claim_id}/review` — submit reviewer decision
- [ ] Auth enforcement: only `reviewer` or `admin` roles can submit reviews

### Acceptance Criteria Mapping
| AC | Test |
|---|---|
| AC-04.3 | Override records: reviewer_id, timestamp, new classification, override_reason |
| AC-04.3 | Original AI classification is still retrievable after override |

### Acceptance Test
```
GIVEN  a claim in UNDER_REVIEW queue
WHEN   a reviewer POSTs an OVERRIDDEN action with new classification
THEN   an OVERRIDDEN event is written with reviewer_id, reason, new classification
AND    original AI CLASSIFIED event still exists in audit trail
AND    claim status is CLOSED
```

---

## TASK-09 — Audit Log Service & API

**Priority:** P1 — Core Flow
**Depends on:** TASK-02, TASK-03, TASK-10
**Estimated complexity:** Small
**Maps to:** FR-05 (AC-05.1, AC-05.2, AC-05.3)

### Description
Build the audit trail query service and read-only API endpoint.

### Deliverables
- [ ] `app/services/audit.py`:
  - `write_event(...)` — append-only insert to review_events
  - `get_audit_trail(claim_id)` — returns chronological list of all events for a claim
  - No update or delete methods exist
- [ ] `app/api/routes/audit.py`:
  - `GET /api/v1/claims/{claim_id}/audit` — returns full event history
- [ ] Auth enforcement: only `auditor` or `admin` roles can access
- [ ] Database-level: application DB role has INSERT + SELECT only (no UPDATE/DELETE) on `review_events`

### Acceptance Criteria Mapping
| AC | Test |
|---|---|
| AC-05.1 | Every event is immutable and contains all required fields |
| AC-05.2 | Query by claim_id returns complete chronological log |
| AC-05.3 | No API or service method exists to update or delete audit records |

### Acceptance Test
```
GIVEN  a claim has been classified, escalated, and overridden
WHEN   GET /api/v1/claims/{claim_id}/audit is called
THEN   3 events are returned in chronological order
AND    each event has: claim_id, timestamp, actor, action, classification, model_version

GIVEN  an attempt to call DELETE on audit events
THEN   no such endpoint exists, and the DB role cannot execute DELETE on review_events
```

---

## TASK-10 — Authentication & RBAC Middleware

**Priority:** P0 — Foundation
**Depends on:** TASK-01
**Estimated complexity:** Medium
**Maps to:** NFR-06, Plan §7 (Security)

### Description
Implement JWT-based authentication and role-based access control.

### Deliverables
- [ ] `app/core/security.py`:
  - `verify_token(token)` → validates JWT signature using JWKS from `JWT_PUBLIC_KEY_URL`
  - Extracts `sub` (user_id) and `roles` from token claims
  - `require_role(*roles)` → FastAPI dependency that checks the user has at least one required role
- [ ] `app/api/dependencies.py`:
  - `get_current_user` dependency — extracts and validates JWT from `Authorization: Bearer` header
  - Returns user context (id, roles)
- [ ] Role definitions: `reviewer`, `auditor`, `admin`
- [ ] All protected endpoints use `Depends(require_role(...))`:
  - Claims routes: `reviewer`, `admin`
  - Review routes: `reviewer`, `admin`
  - Audit routes: `auditor`, `admin`
  - Health: public

### Acceptance Test
```
GIVEN  a request to POST /api/v1/claims without a JWT
THEN   response is 401 UNAUTHORIZED

GIVEN  a valid JWT with role "auditor"
WHEN   POST /api/v1/claims is called
THEN   response is 403 FORBIDDEN (auditor cannot submit claims)

GIVEN  a valid JWT with role "reviewer"
WHEN   POST /api/v1/claims is called
THEN   the request proceeds to the ingestion service
```

---

## TASK-11 — Integration Tests & Scenario Validation

**Priority:** P1 — Quality Assurance
**Depends on:** TASK-04, TASK-05, TASK-06, TASK-07, TASK-08, TASK-09
**Estimated complexity:** Large
**Maps to:** Spec §9 (Reference Scenarios), Plan §10 (Testing Strategy)

### Description
Build integration tests covering the 3 reference scenarios end-to-end, plus edge cases.

### Deliverables
- [ ] `tests/fixtures/scenario_work_related.json` — Scenario 2 test data
- [ ] `tests/fixtures/scenario_non_work_related.json` — Scenario 1 test data (with aggravation)
- [ ] `tests/fixtures/scenario_undetermined.json` — Scenario 3 test data
- [ ] `tests/integration/test_api_claims.py`:
  - Test: submit bundle → poll status → verify classification result
  - Test: submit empty bundle → 422
  - Test: get non-existent claim → 404
- [ ] `tests/integration/test_api_review.py`:
  - Test: override a classification → verify event recorded, original preserved
  - Test: approve an escalated claim → status becomes CLOSED
- [ ] `tests/integration/test_api_audit.py`:
  - Test: full lifecycle (classify → escalate → override) → audit trail has 3 events
- [ ] `tests/integration/test_scenarios.py`:
  - Test Scenario 1: NON_WORK_RELATED + WORK_AGGRAVATED
  - Test Scenario 2: WORK_RELATED
  - Test Scenario 3: UNDETERMINED + auto-escalation

### Acceptance Test
```
GIVEN  all 3 reference scenarios are submitted through the full pipeline
THEN   each produces the expected classification, secondary tag, and reasoning
AND    audit trails are complete for each
```

---

## TASK-12 — Docker & CI/CD Pipeline

**Priority:** P2 — Infrastructure
**Depends on:** TASK-01
**Estimated complexity:** Medium
**Maps to:** Plan §2 (Containerization, CI/CD)

### Description
Containerize the application and set up the CI/CD pipeline.

### Deliverables
- [ ] `Dockerfile` — Multi-stage build (slim Python image)
- [ ] `docker-compose.yml` — Services: app, postgres, redis, minio (local S3)
- [ ] `.github/workflows/ci.yml`:
  - Lint: `ruff check`
  - Type check: `mypy`
  - Test: `pytest` with PostgreSQL service container
  - Build: Docker image build
- [ ] `README.md` with setup instructions, env vars, and development workflow

### Acceptance Test
```
GIVEN  the repo is cloned
WHEN   `docker compose up` is run
THEN   the app, database, redis, and minio all start
AND    the health endpoint responds at http://localhost:8000/api/v1/health
```

---

## TASK-13 — Configuration & Environment Management

**Priority:** P1 — Foundation
**Depends on:** TASK-01
**Estimated complexity:** Small
**Maps to:** Plan §9 (Configuration)

### Description
Centralize all configuration into a Pydantic Settings class with validation and defaults.

### Deliverables
- [ ] `app/config.py`:
  - `Settings` class with all env vars from Plan §9
  - Validation: required fields raise at startup if missing
  - Defaults for optional fields (ESCALATION_THRESHOLD=0.70, LOG_LEVEL=INFO, etc.)
  - `get_settings()` cached dependency
- [ ] `.env.example` with all variables, descriptions, and defaults

### Acceptance Test
```
GIVEN  DATABASE_URL is not set
WHEN   the app starts
THEN   a clear error message is shown indicating DATABASE_URL is required

GIVEN  ESCALATION_THRESHOLD is not set
WHEN   the app starts
THEN   it defaults to 0.70
```

---

## Implementation Order — Suggested Sprint Plan

### Sprint 1 — Foundation (TASK-01, 02, 03, 10, 13)
Set up the project, database, schemas, auth, and config. Everything else depends on these.

```
Week 1: TASK-01 (bootstrap) + TASK-13 (config) + TASK-03 (schemas)
Week 2: TASK-02 (DB models) + TASK-10 (auth)
```

### Sprint 2 — Core Pipeline (TASK-04, 05, 06, 07)
Build the ingestion → processing → classification → routing pipeline end-to-end.

```
Week 3: TASK-04 (ingestion) + TASK-05 (doc processing)
Week 4: TASK-06 (classification engine) + TASK-07 (escalation routing)
```

### Sprint 3 — Review, Audit & Testing (TASK-08, 09, 11)
Complete the review loop, audit trail, and comprehensive testing.

```
Week 5: TASK-08 (review/overrides) + TASK-09 (audit)
Week 6: TASK-11 (integration tests + scenario validation)
```

### Sprint 4 — Infrastructure & Polish (TASK-12)
Containerize, set up CI/CD, and prepare for deployment.

```
Week 7: TASK-12 (Docker + CI/CD) + final polish
```

---

## Task Summary Matrix

| Task | Title | Priority | Depends On | FR/AC Coverage | Complexity |
|---|---|---|---|---|---|
| TASK-01 | Project Bootstrap | P0 | — | — | Small |
| TASK-02 | Database Models & Migrations | P0 | 01 | Spec §6 | Medium |
| TASK-03 | Pydantic Schemas | P0 | 01 | Spec §6, Plan §5 | Small |
| TASK-04 | Ingestion Service & API | P1 | 02, 03 | FR-01 | Medium |
| TASK-05 | Document Processing | P1 | 04 | FR-01, FR-02 | Small |
| TASK-06 | Classification Engine | P0 | 05 | FR-02, FR-03, FR-06 | Large |
| TASK-07 | Escalation Routing | P1 | 06 | FR-04 (04.1, 04.2) | Small |
| TASK-08 | Review Queue & Overrides | P1 | 07, 03, 10 | FR-04 (04.3) | Medium |
| TASK-09 | Audit Log Service | P1 | 02, 03, 10 | FR-05 | Small |
| TASK-10 | Auth & RBAC | P0 | 01 | NFR-06 | Medium |
| TASK-11 | Integration Tests | P1 | 04–09 | Spec §9 | Large |
| TASK-12 | Docker & CI/CD | P2 | 01 | — | Medium |
| TASK-13 | Configuration Management | P1 | 01 | Plan §9 | Small |

---

## Tasks Version History

| Version | Date | Changes |
|---|---|---|
| 1.0.0 | 2026-05-12 | Initial task breakdown |
