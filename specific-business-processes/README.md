# Specific business processes

A specific business process (SBP) explains how one feature works across the
browser, backend and any external services. Start with its guide for a plain-language
walkthrough, then open the matching diagram to follow the decisions and arrows.

## Find a process

| Module                                           | Processes                                                                                               |
| ------------------------------------------------ | ------------------------------------------------------------------------------------------------------- |
| [Authentication](guide/authentication/README.md) | Login, signup, logout, JWT interceptor, and Forgot Password recovery. |
| [Ecommerce](guide/ecommerce/README.md)           | Landing-page catalog flow.                                                                              |

Module indexes link both the guide and diagram for every process. They separate
proposed flows from existing coverage and identify older diagrams with known
limitations. A file's location does not establish that its behavior is implemented.

## Folder structure

```text
specific-business-processes/
├── README.md
├── guide/
│   └── <module>/
│       ├── README.md
│       └── <process-name>.md
└── diagrams/
    └── <module>/
        └── <process-name>.drawio.svg
```

Every diagram requires a beginner-friendly guide with the same module and
basename. For example, `diagrams/authentication/login.drawio.svg` pairs with
`guide/authentication/login.md`. The `guide/` spelling is intentional.

## Maintain a process

Create or update the diagram and guide together within the requested workflow
scope. Explain the feature's purpose and entry point before walking through its
phases. Define unfamiliar terms, identify who owns each action, and explain
decisions, saved state, errors, retry paths and completion. State whether the
flow is implemented or proposed, and retain known limitations.

Read the relevant frontend, backend and documented contracts before describing
current behavior. Keep exact API contracts in the application repositories and
link to them. Explain any known differences between an older diagram and current
code rather than presenting its historical labels as current requirements.

Use the frontend's [system-diagrams skill](../../frontend/.agents/skills/system-diagrams/SKILL.md)
for the paired-document workflow, [drawio-sbp skill](../../frontend/.agents/skills/drawio-sbp/SKILL.md)
for drawing conventions and validation, and [documentation-writing skill](../../frontend/.agents/skills/documentation-writing/SKILL.md)
for explanations. New or edited diagrams use editable Draw.io SVGs. The landing-page
file is an existing exception: it contains raw Draw.io XML despite its SVG extension.

Add both links and a short explanation to `guide/<module>/README.md`; add new
modules to this index. Keep proposed entries separate. Check guide/diagram
pairing, index coverage, relative links and implementation status. For drawing
changes, perform the structural, rendering and editor checks required by
drawio-sbp. Moving files alone does not require redrawing them. Preserve diagram
bytes during a folder-only move. Put execution reports in the handoff or PR.

## Related documentation

- [System overview](../system-overview.md) introduces the application.
- [Architecture](../architecture/README.md) explains application and hosting responsibilities.
- [Backend data models](../../backend/docs/erd/README.md) explain stored relationships.
- [General business processes](../planning/general-business-processes/README.md) cover system-wide BPMN planning.
- [Documentation home](../README.md) provides the full reading map.
