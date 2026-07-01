# Codex Workflow Standard

## Before Work

- Check `git status --short --branch`.
- Identify the related `US-xxx`.
- Read `standards/README.md` and the task-specific standards.
- Read the related requirement and prototype spec.
- Confirm whether existing worktree changes are user changes.

## During Work

- Keep edits scoped to the requested US.
- Do not rewrite PRD/TDD unless the task asks for it.
- Update indexes when adding important docs.
- Preserve user edits and unrelated changes.
- For UI work, verify important responsive states before completion.

## Verification

- Run the smallest meaningful checks available.
- If checks cannot run, record why.
- Compare implementation against acceptance criteria and prototype states.
- For documentation-only changes, verify links and folder structure.

## Reporting

Final response should include:

- What changed.
- Where to look.
- What verification ran.
- Any skipped checks or known risks.

If Git actions were requested, follow `standards/git-standard.md`.
