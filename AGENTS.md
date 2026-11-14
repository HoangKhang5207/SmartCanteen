# Software Engineering Project Rules

## Project Type
GREENFIELD.

## Authoritative Sources
1. Approved requirements and acceptance criteria.
2. Approved architecture decisions.
3. Existing source code and contracts.
4. DESIGN.md for frontend tasks.
5. Verified test and runtime evidence.

## Required Workflow
Understand → Plan → Implement → Test → Review → Verify → Handoff.

Follow installed Skills relevant to the current task.

Use Superpowers as the primary SDLC workflow when available.
Do not combine competing workflow implementations unnecessarily.

## Before Editing
- Inspect Git status and existing changes.
- Read relevant project documentation.
- Identify the task scope.
- Identify tests and acceptance criteria.
- Preserve unrelated work.

## During Implementation
- Make the smallest appropriate change.
- Follow existing architecture.
- Use appropriate tests.
- Avoid unrelated refactoring.
- Do not fabricate requirements.
- Do not disable quality checks to obtain a passing result.

## Frontend
Read DESIGN.md.

## Backend
Preserve API contracts, authorization and data consistency.

## Database
Never perform risky migrations or destructive operations without approval.

## Completion
Run relevant quality gates.

Report:
- Files changed.
- Tests executed.
- Actual results.
- Acceptance criteria status.
- Remaining risks.

Do not claim completion without evidence.

## Restricted Actions
Do not push, merge, deploy, delete data,
expose secrets or modify production without authorization.
