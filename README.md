# E-Commerce system documentation

This repository explains how the E-Commerce website, backend, database, and
background services work together. Start here when you need the whole system's
behavior rather than one application's implementation details.

The project also contains personal draw.io references and future design notes.
They have separate homes so readers can distinguish current behavior from ideas
that are not implemented.

## Start reading here

1. Read the [system overview](system-overview.md) for the main parts, everyday
   customer journey, and common terms.
2. Read [architecture](architecture/README.md) to understand how those parts run
   and communicate.
3. Use the [maintained diagram index](diagrams/README.md) for a particular process
   or data-model explanation.
4. Use [planning](planning/README.md) only when researching future behavior, or
   [draw.io references](drawio/README.md) to explore the editable visual notes.

## Find the right kind of information

| Location        | Purpose                                                                                   |
| --------------- | ----------------------------------------------------------------------------------------- |
| `architecture/` | Implemented application and hosting boundaries, with explicit configuration limits        |
| `diagrams/`     | Mermaid business processes and links to backend-owned ERD guides                          |
| `planning/`     | Proposed settlement, withdrawal, and return behavior; superseded ideas are archived       |
| `drawio/`       | Editable visual references, whose pages can include current, partial, or planned behavior |

A file's format or author does not prove it matches implementation. Maintained
Markdown must still be checked against the code; a draw.io reference is not
obsolete just because it is stored separately. Each guide states its own scope.

## Related repositories

| Repository                                                         | What belongs there                                                            |
| ------------------------------------------------------------------ | ----------------------------------------------------------------------------- |
| [Frontend](https://github.com/RHILIUS/E-Commerce-API-Frontend)     | Customer screens, browser state, and UI implementation                        |
| [Backend](https://github.com/RHILIUS/E-Commerce-API)               | Exact API contracts, models, business services, and operator commands         |
| [System documentation](https://github.com/RHILIUS/E-Commerce-Docs) | This repository: cross-application architecture, processes, and relationships |

Links assume the three repositories are checked out beside one another as
`frontend/`, `backend/`, and `docs/`. This repository is the sibling `docs/`;
the application repositories also have their own local `docs/` folders. When
browsing on GitHub, open the relevant repository from the table to follow a
sibling reference.

## Maintaining the documentation

Describe current behavior in plain language before introducing routes or model
names. Keep proposals distinct, preserve actual limitations, and update related
links and indexes when moving a guide. Follow the
[diagramming specification](diagrams/diagramming-specification.md) for process
notation and validation. Detailed API and operational instructions stay in the
application repositories rather than being copied here.
