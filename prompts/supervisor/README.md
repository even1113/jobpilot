# Supervisor Prompts

Use these prompts when a human or supervisor agent checks whether Codex output is good enough.

Supervisor must check:

- Did the work follow the related `US-xxx`?
- Did implementation drift from the prototype or requirement?
- Were tests run or explicitly skipped with a reason?
- Is a human handoff required by `quality/human-handoff.md`?
- Is the Git commit or PR traceable and scoped?
