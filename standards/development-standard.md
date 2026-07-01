# Development Standard

## Scope

- Implement against a specific `US-xxx` whenever behavior changes.
- Keep changes small and traceable.
- Do not add broad abstractions before the codebase needs them.
- Follow existing project patterns before introducing new ones.

## Implementation

- Read the related requirement and prototype before coding.
- Preserve product safety boundaries around platform automation, privacy, and truthful job-search data.
- Add clear fallback behavior for AI output, platform failures, and long-running tasks.
- Record important implementation decisions in `docs/decisions/` when they affect future work.

## UI

- Match the prototype and document intentional deviations.
- Cover loading, empty, error, disabled, and manual handoff states.
- Keep text readable and layouts stable on target viewports.

## Done

Implementation is not done until it satisfies `quality/definition-of-done.md`.
