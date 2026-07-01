# Test Gate

## Documentation-only Changes

- Verify files exist in the expected directories.
- Verify required sections are present.
- Verify index files link or describe the new documents.

## Prototype Changes

- Verify primary interactions.
- Verify important responsive viewports when feasible.
- Verify loading, empty, error, and disabled states where applicable.
- Include screenshots or notes for visual changes when meaningful.

## Application Changes

- Run unit tests for touched logic.
- Run integration tests for API/data flow changes.
- Run e2e or browser checks for user-critical flows.
- Add regression tests for bug fixes.

## Safety-sensitive Automation

- Verify pause on captcha, login failure, platform risk, and user handoff conditions.
- Verify no bypassing platform controls.
- Verify execution logs are retained for audit.

## Passing Rule

A test gate passes when required checks run successfully, or skipped checks are explicitly justified with residual risk.
