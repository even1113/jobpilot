# JobPilot Standards

This is the first file Codex or any agent should read before changing JobPilot.

## Required Reading By Task Type

For any feature implementation:

1. `requirements/backlog.md`
2. Related `US-xxx` requirement file or backlog entry
3. Related prototype spec in `prototype/features/`, if UI is involved
4. `standards/development-standard.md`
5. `standards/testing-standard.md`
6. `standards/git-standard.md`
7. `quality/definition-of-ready.md`
8. `quality/definition-of-done.md`

For UX prototype work:

1. Related `US-xxx`
2. `standards/prototype-standard.md`
3. `standards/document-standard.md`
4. `research/ux/benchmark-template.md`

For documentation work:

1. `standards/document-standard.md`
2. `standards/requirement-standard.md`, if requirements are touched
3. `standards/git-standard.md`

## Core Principles

- Work starts from a traceable `US-xxx` whenever it changes product behavior.
- Requirements, prototypes, tests, and commits must be linked.
- Do not let AI jump from vague idea to code.
- Preserve existing user changes unless explicitly asked to revert them.
- Stop and hand off when safety, compliance, platform automation, or repeated failure requires human judgment.

## Standards Index

- `codex-workflow.md`: how Codex should start, execute, verify, and report work.
- `document-standard.md`: where documents live and how to index them.
- `requirement-standard.md`: how to write and manage US requirements.
- `prototype-standard.md`: how to manage UX and high-fidelity prototypes.
- `development-standard.md`: coding and implementation rules.
- `testing-standard.md`: test design and verification rules.
- `git-standard.md`: branches, commits, PRs, and pre-commit checks.
- `agent-collaboration-standard.md`: multi-agent roles and handoff rules.
