# Agent Runbook

## When Starting

1. Read `standards/README.md`.
2. Identify related `US-xxx`.
3. Read requirement, prototype, development, testing, and Git standards.
4. Inspect current repo state.

## When Stuck

- Re-read the requirement and acceptance criteria.
- Reduce the change to the smallest failing case.
- Run the narrowest relevant check.
- Search existing code and docs for the established pattern.
- If the same blocker repeats, use `human-handoff.md`.

## When Codex Stops Mid-task

The next run should:

- Check `git status --short --branch`.
- Inspect recently modified files.
- Compare against the requested `US-xxx`.
- Continue from verified state instead of restarting from assumptions.

## Before Final Report

- Summarize changed files and purpose.
- State checks run.
- State skipped checks and risks.
- Mention whether Git actions were performed.
