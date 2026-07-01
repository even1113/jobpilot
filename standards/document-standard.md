# Document Standard

## Document Areas

- `docs/`: product-level and technical-level documents.
- `requirements/`: formal user stories and feature-level requirements.
- `prototype/features/`: feature-level UX and prototype specs.
- `research/ux/`: benchmark research and design references.
- `standards/`: operating rules for the project.
- `quality/`: readiness, done, tests, and handoff gates.
- `prompts/`: reusable prompts for agents.

## Naming

- Requirement files should start with `US-xxx-`.
- Decision records should use `YYYY-MM-DD-short-title.md`.
- Prototype specs should include the related `US-xxx`.
- Prefer lowercase English filenames for stable linking.

## Indexing

- Add important docs to the nearest `README.md`.
- Add formal features to `requirements/backlog.md`.
- Add traceability to `requirements/feature-map.md`.

## Change Control

- Keep large PRD/TDD documents stable.
- Put detailed feature evolution in smaller linked files.
- Record irreversible or high-impact decisions in `docs/decisions/`.
