# Requirements

`requirements/` manages formal JobPilot user stories. Every buildable requirement must have a stable `US-xxx` ID.

## Required Files

- `backlog.md`: source of truth for all assigned US IDs and statuses.
- `feature-map.md`: maps each US to docs, prototype, tests, and implementation notes.
- `user-story-template.md`: template for new feature requirements.

## US ID Rules

- Formal requirements use `US-001`, `US-002`, `US-003`.
- Subtasks may use `US-001-A`, `US-001-B`.
- Deprecated IDs are never reused.
- Functional commits must include at least one US ID.

## Requirement Status

- `draft`: idea is being shaped.
- `ready-for-ux`: clear enough for UX research/prototype.
- `ready-for-dev`: requirement and prototype are ready for development.
- `in-dev`: implementation is active.
- `in-review`: implementation is waiting for review.
- `done`: accepted and merged.
- `deprecated`: no longer planned, ID retained.

## Ready Rule

Development should not start until the related requirement passes `quality/definition-of-ready.md`.
