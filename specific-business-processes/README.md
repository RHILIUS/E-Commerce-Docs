# Specific business processes

A specific business process (SBP) explains the steps, decisions, and handoffs for
one feature. This folder contains the E-Commerce system's editable Draw.io SBPs,
organized by module.

## Modules

| Module                                     | Available SBPs                              |
| ------------------------------------------ | ------------------------------------------- |
| [Authentication](authentication/README.md) | Login, signup, logout, and JWT interceptor. |
| [Ecommerce](ecommerce/README.md)           | Landing page.                               |

Use each module's README to find its diagrams and a short explanation of each.
Diagram coverage does not establish whether a feature is implemented; check the
application guides and current code for that distinction.

## Maintaining SBPs

Use the frontend repository's
[drawio-sbp skill](../../frontend/.agents/skills/drawio-sbp/SKILL.md) for drawing
conventions, editable SVG handling, and validation. New SBPs use editable
`.drawio.svg` files rather than Mermaid.

Store diagrams at `<module>/<process-name>.drawio.svg`, using lowercase kebab-case
names. Each module must have a `README.md` linking every SBP in that folder with a
short purpose description. Add a module to this index when creating it. Where
rules or API handoffs need more explanation, add an adjacent
`<process-name>.md` guide and link it from the same module entry.

Inspect the relevant frontend, backend, and API contracts before documenting an
implemented flow. State each new diagram's scope and whether it describes current
behavior or a future design in its index entry or companion guide. Keep proposals
separate from existing coverage and retain actual limitations. Exact UI and API
details belong in the application guides; link to them rather than duplicating
their contracts. Update affected diagrams and indexes when workflows change.

Follow drawio-sbp's structural checks, decision and recovery-path review, SVG
rendering, and Draw.io reopening check when available. Check relative links and
report actual validation and limitations in the handoff. Index-only changes do
not require redrawing existing diagrams.

## Related documentation

- [System overview](../system-overview.md) introduces the application.
- [Architecture](../architecture/README.md) explains system responsibilities and hosting.
- [Backend data models](../../backend/docs/erd/README.md) contain ERD guides and editable Draw.io diagrams.
- [General business processes](../planning/general-business-processes/README.md) cover system-wide BPMN planning.
- [Documentation home](../README.md) provides the full reading map.
