---
name: data-analysis-consultant-specs
description: Use when working in the data_analysis_consultant project and you need to understand or update the orchestration, frontend, API mock, review flow, governance rules, or agent responsibilities. This skill is for keeping backend_spec.md, frontend_spec.md, and frontend_api_mock.json aligned.
---

# Data Analysis Consultant Specs

Use this skill when editing or reviewing this project's specifications or mock contracts.

## Core Files

- `../../backend_spec.md`
  Source of truth for orchestration, agents, tools, governance, QA, override policy, and state schema.
- `../../frontend_spec.md`
  Source of truth for workspace UX, review UX, artifact presentation, governance visibility, and frontend/backend projections.
- `../../frontend_api_mock.json`
  Mock API payloads for project summary, reviews, artifacts, governance, timeline, deliverables, and notifications.

## Workflow

1. Read the relevant spec first instead of assuming old behavior.
2. If backend governance, QA, risk, override, or tool contracts change, check whether `frontend_spec.md` needs UI or contract updates.
3. If frontend projections or screens change, check whether `frontend_api_mock.json` still reflects the updated fields and states.
4. Keep these concepts aligned across files:
   - `approval routing`
   - `justified override`
   - QA stages and downstream gates
   - `draft` / `internal` / `QA-approved` / `user-reviewed` / `final-delivery`
   - provisional mode
   - governance summaries and access masking
5. Prefer updating the spec and the mock in the same change when a user-facing contract changes.

## Guardrails

- Do not introduce frontend fields that are unsupported by `backend_spec.md` unless you update the backend spec too.
- Do not collapse `provisional` outputs into final outputs in mocks or UI specs.
- Do not describe Supervisor override as unconditional; use the justified override model.
- Do not let State Manager become a recommendation engine; keep it typed and deterministic.

## Quick Checks

- If a new backend field affects UI, ensure it appears in `frontend_spec.md` and, when useful, in `frontend_api_mock.json`.
- If a new artifact quality state is added, reflect it in artifact display rules and mocks.
- If a new governance tool is added, check whether timeline, status, or governance panels should expose it.
