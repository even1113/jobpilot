# Testing Standard

## Test Scope

Tests should cover:

- Acceptance criteria for the related `US-xxx`.
- Core success path.
- Important failure, empty, and manual handoff states.
- AI output validation or fallback behavior where applicable.
- Platform automation pause behavior when risk or verification appears.

## Test Levels

- Documentation-only changes: verify structure, links, and required sections.
- Prototype changes: verify key interactions and responsive states.
- Application changes: run unit/integration/e2e checks appropriate to the touched area.
- Safety-sensitive automation: include negative tests for pause, handoff, and no-bypass behavior.

## Reporting

Every final report should say:

- What checks ran.
- What passed.
- What was not run and why.
- Residual risk.

Use `quality/test-gate.md` as the release gate.
