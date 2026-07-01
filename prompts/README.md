# Prompt Library

`prompts/` stores reusable prompts for Codex and other agents.

## Areas

- `ux/`: UX audit, benchmark, prototype, and visual review prompts.
- `dev/`: implementation prompts.
- `test/`: test design and verification prompts.
- `review/`: code review and document review prompts.
- `supervisor/`: prompts for supervising agent output and deciding when a human should take over.

## Rules

- Every prompt must tell the agent to read `standards/README.md` first.
- Development prompts must require a `US-xxx` ID.
- Test prompts must link to acceptance criteria and `quality/test-gate.md`.
- Review prompts must check Git and US traceability.
