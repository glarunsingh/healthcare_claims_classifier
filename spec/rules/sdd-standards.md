# SDD Rules & Standards

> Enforcement rules for AI coding agents (Cursor, Copilot, Cody, etc.) working in this repository. These rules ensure that implementation stays aligned with the specification at all times.

---

## Rule 1 — Spec-First, Always

**Before writing any implementation code, the agent MUST:**

1. Read the relevant spec: `.spec/specs/claim-classification.md`
2. Read the relevant plan: `.spec/plans/claim-classification-plan.md`
3. Read the relevant task: `.spec/tasks/claim-classification-tasks.md`
4. Identify which TASK-XX the current work maps to
5. Identify which FR and AC the task maps to

**If the requested work does not map to any existing task or spec requirement:**
- STOP and ask the user whether to update the spec first
- Do NOT implement unspecified features

---

## Rule 2 — Acceptance Criteria Are Tests

Every acceptance criterion (AC-XX.X) in the spec MUST have a corresponding test.

- Unit test for logic-level ACs (e.g., confidence calculation, enum validation)
- Integration test for API-level ACs (e.g., submit empty bundle → 422)
- Scenario test for end-to-end ACs (e.g., Scenario 1 → NON_WORK_RELATED + WORK_AGGRAVATED)

**A task is NOT complete until all its mapped ACs have passing tests.**

---

## Rule 3 — Data Model Fidelity

The implementation MUST match the data models defined in Spec §6:

- `ClaimBundle`, `Document`, `ClassificationResult`, `ReasoningTrace`, `ReviewEvent`
- Field names, types, and constraints must be exactly as specified
- Enum values must match exactly: `WORK_RELATED`, `NON_WORK_RELATED`, `UNDETERMINED`, `WORK_AGGRAVATED`
- Do NOT add fields not in the spec without updating the spec first

---

## Rule 4 — API Contract Compliance

API endpoints MUST match the contracts defined in Plan §5:

- HTTP methods, paths, request/response shapes, and status codes are authoritative
- Any deviation (adding a field, changing a status code, renaming a path) requires a spec/plan update first
- All responses must use the Pydantic schemas defined in TASK-03

---

## Rule 5 — Classification Decision Tree

The classification logic MUST follow the decision tree in Spec §8:

```
Work evidence present?
├── YES → WORK_RELATED
├── NO → Non-work evidence present?
│        ├── YES → Work aggravated?
│        │        ├── YES → NON_WORK_RELATED + WORK_AGGRAVATED
│        │        └── NO  → NON_WORK_RELATED
│        └── UNCLEAR → UNDETERMINED
└── INSUFFICIENT → UNDETERMINED
```

- UNDETERMINED always triggers escalation to HUMAN_REVIEW
- The secondary tag WORK_AGGRAVATED is ONLY allowed when classification is NON_WORK_RELATED
- Do NOT invent new classification values or tags

---

## Rule 6 — Constraints Are Inviolable

The constraints in Spec §10 (DO NOTs) are hard rules:

| # | Constraint |
|---|---|
| C-01 | No legal determinations or compensation calculations |
| C-02 | English only in v1 |
| C-03 | No external CMS integration in v1 |
| C-04 | No model retraining pipeline in v1 |
| C-05 | No PII in logs beyond reasoning trace excerpts |
| C-06 | No deletion or mutation of audit log entries |
| C-07 | No programmatic override of human reviewer decisions |

**If a user requests work that violates a constraint: flag it, reference the constraint number, and ask for confirmation before proceeding.**

---

## Rule 7 — Audit Immutability

The `review_events` table is **append-only**:

- The application code MUST NOT contain any UPDATE or DELETE operations on this table
- The database role used by the application SHOULD have only INSERT + SELECT grants on this table
- Every state change (classification, escalation, approval, override) MUST produce a new event row
- Original AI classifications are NEVER mutated, even after human override

---

## Rule 8 — Security Defaults

- All endpoints except `/api/v1/health` require authentication
- JWT validation uses RS256 with keys from a JWKS endpoint
- Role-based access control is enforced via FastAPI dependencies
- Secrets (API keys, DB credentials) are loaded from environment variables, never hardcoded
- PII is never logged at INFO level or below

---

## Rule 9 — Testing Requirements

| Test Level | Minimum Coverage | Runner |
|---|---|---|
| Unit tests | ≥ 90 % of service functions | pytest |
| Integration tests | ≥ 80 % of API endpoints | pytest + httpx |
| Scenario tests | 100 % of reference scenarios (Spec §9) | pytest |

- Tests MUST be runnable with `pytest` from the project root
- Tests MUST NOT depend on external LLM APIs in CI (use mocked responses)
- Each test file should clearly indicate which TASK and AC it validates

---

## Rule 10 — Version Tracking

- Every `ClassificationResult` MUST include `model_version` from application config
- Spec, Plan, and Task documents have version histories — update them when changes are made
- Database migrations are sequential and named descriptively

---

## Rule 11 — Change Protocol

When the spec, plan, or tasks need to change:

1. **Propose** the change with a clear rationale
2. **Update** the relevant `.spec/` document(s)
3. **Update** dependent documents (e.g., spec change may require plan/task updates)
4. **Update** version history table in each modified document
5. **Then** implement the change in code

**Never implement first and document later.**

---

## Rule 12 — File & Code Organization

- Follow the project structure defined in Plan §6
- One ORM model per file in `app/models/`
- One route group per file in `app/api/routes/`
- Business logic lives in `app/services/`, NOT in route handlers
- Prompt templates live in `app/prompts/`
- Test fixtures live in `tests/fixtures/`

---

## Rules Version History

| Version | Date | Changes |
|---|---|---|
| 1.0.0 | 2026-05-12 | Initial SDD rules and standards |
