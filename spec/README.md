# Healthcare Claim Injury Classification System

> **Spec-Driven Development (SDD) — Complete Artifact Set**

This repository uses Spec-Driven Development. AI coding agents and human developers MUST read the spec documents before writing implementation code.

---

## Quick Start for AI Agents (Cursor / Copilot / Cody)

**Before implementing anything, read these files in order:**

1. **Spec** → [`.spec/specs/claim-classification.md`](.spec/specs/claim-classification.md) — *What* we're building and *why*
2. **Plan** → [`.spec/plans/claim-classification-plan.md`](.spec/plans/claim-classification-plan.md) — *How* we'll build it (architecture, stack, API contracts)
3. **Tasks** → [`.spec/tasks/claim-classification-tasks.md`](.spec/tasks/claim-classification-tasks.md) — *Work units* with dependencies and acceptance tests
4. **Rules** → [`.spec/rules/sdd-standards.md`](.spec/rules/sdd-standards.md) — *Enforcement rules* for implementation

---

## SDD Phases

| Phase | Document | Status |
|---|---|---|
| **1. SPECIFY** | [claim-classification.md](.spec/specs/claim-classification.md) | Complete |
| **2. PLAN** | [claim-classification-plan.md](.spec/plans/claim-classification-plan.md) | Complete |
| **3. TASKS** | [claim-classification-tasks.md](.spec/tasks/claim-classification-tasks.md) | Complete |
| **4. IMPLEMENT** | Source code in `app/` | Not Started |

---

## Project Summary

An AI-powered system that classifies healthcare claim injuries as:

- **WORK_RELATED** — Injury occurred during work hours, on work premises, or performing work duties
- **NON_WORK_RELATED** — Injury occurred outside of work
- **UNDETERMINED** — Insufficient or conflicting evidence

With an optional secondary tag:
- **WORK_AGGRAVATED** — A non-work injury that was worsened by work activities

Key capabilities:
- Document ingestion (claim forms, doctor notes, transcripts, incident reports)
- AI-powered classification with confidence scoring
- Confidence-based escalation to human reviewers
- Reviewer approval and override workflow
- Immutable, auditable decision trail

---

## Getting Started (Implementation)

> **Prerequisites:** API key for an LLM provider (OpenAI, Azure OpenAI, or Anthropic)

```bash
# 1. Clone the repo
git clone <repo-url> && cd healthcare_claims_classifier

# 2. Copy environment config
cp .env.example .env
# Edit .env with your API keys and database URL

# 3. Start services (after implementation)
docker compose up

# 4. Run tests
pytest
```

---

## User Story

See [userstory.md](userstory.md) for the original business requirements.

---

## Folder Structure

```
.spec/
├── specs/          # Phase 1: What and why (functional requirements)
├── plans/          # Phase 2: How (architecture, tech stack, API design)
├── tasks/          # Phase 3: Work units (discrete, testable tasks)
└── rules/          # Enforcement rules for AI agents and developers
```
