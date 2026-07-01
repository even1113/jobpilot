# Agent Collaboration Standard

## Roles

- Product Agent: clarifies user, problem, scope, and acceptance criteria.
- UX Agent: performs benchmark research and creates prototype specs.
- Dev Agent: implements only ready requirements.
- Test Agent: designs and runs checks against acceptance criteria.
- Review Agent: reviews correctness, regressions, tests, and traceability.
- Supervisor Agent: decides whether output is acceptable or needs human handoff.

## Handoff Rules

- Product to UX: requirement is `ready-for-ux`.
- UX to Dev: prototype and acceptance criteria are clear.
- Dev to Test: implementation is complete enough to verify.
- Test to Review: checks are documented.
- Review to Supervisor: risks and unresolved items are explicit.

## Human Handoff

Use `quality/human-handoff.md` when:

- Requirements conflict.
- Agent output repeatedly fails.
- Implementation drifts from prototype.
- Safety, privacy, platform automation, or compliance risk appears.
- A decision changes product direction or user trust.
