# Git Standard

## Branches

Default development branch: `dev`.

Feature branches:

- `codex/US-001-short-topic`
- `feature/US-001-short-topic`
- `fix/US-001-short-topic`

## Commit Format

Use:

```text
type(US-001): summary
```

Allowed types:

- `docs`
- `ux`
- `proto`
- `feat`
- `fix`
- `test`
- `refactor`
- `chore`

Examples:

```text
docs(US-001): add job preference requirement draft
proto(US-001): add preference setup high fidelity flow
feat(US-002): implement JD classification list view
test(US-002): cover JD classification fallback states
```

## Multi-US Commits

Prefer one primary US per commit. If a commit must span multiple requirements:

```text
feat(US-001,US-002): connect preference setup to JD ranking
```

The commit body must explain why the work could not be split.

## Pre-commit Checklist

- `git status --short` reviewed.
- No unrelated user changes included.
- Related US exists in `requirements/backlog.md`.
- Relevant README/index files updated.
- Meaningful checks run or skipped with reason.
- Commit message includes US ID for functional work.

## PR Description

Every PR should include:

- Related US.
- Summary.
- Screenshots or prototype links when UI changed.
- Tests run.
- Risks and rollback notes.
- Areas needing human attention.
