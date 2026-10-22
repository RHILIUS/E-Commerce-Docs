# E-Commerce system documentation

This repository explains how the E-Commerce website, backend, database, and
background services work together. Start here when you need the whole system's
behavior rather than one application's implementation details.

The project also contains planning diagrams for proposed business flows and class
relationships. They have a separate home so readers can distinguish current
behavior from designs that may change before implementation.

## Start reading here

1. Read the [system overview](system-overview.md) for the main parts, everyday
   customer journey, and common terms.
2. Read [architecture](architecture/README.md) to understand how those parts run
   and communicate.
3. Use the [specific business-process index](specific-business-processes/README.md) for feature flows. Read the
   [backend data-model index](../backend/docs/erd/README.md) for stored relationships.
4. Use [planning](planning/README.md) to explore proposed business flows and
   class relationships before implementation.

## Find the right kind of information

| Location                                                                    | Purpose                                                                                                    |
| --------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| `architecture/`                                                             | Implemented application and hosting boundaries, with explicit configuration limits                         |
| [General business processes](planning/general-business-processes/README.md) | System-wide BPMN planning diagrams used to plan business flows before the corresponding features are built |
| [Specific business processes](specific-business-processes/README.md)        | Feature-specific flowcharts showing detailed business-process steps and decisions                          |
| [Backend data models](../backend/docs/erd/README.md)                        | Backend-owned ERD guides and editable Draw.io diagrams                                                     |
| [Planning](planning/README.md)                                              | Proposed business flows and class relationships that may change before implementation                      |
| [Class diagrams](planning/class-diagrams/README.md)                         | Proposed class structures and relationships                                                                |

The editable `.drawio.svg` diagrams in `planning/general-business-processes/` use Business
Process Model and Notation (BPMN) to plan how the whole system should work before
the corresponding features are built. They describe intended flows; check the
maintained guides and application code for implemented behavior.

The editable `.drawio.svg` diagrams in `specific-business-processes/` are
flowcharts for individual features. They show the steps, decisions, and possible
paths within a specific business process, such as login or signup. Check the
maintained feature guides and application code to confirm which behavior is
implemented.

A file's format or location does not prove it matches implementation. Maintained
Markdown must still be checked against the code. Each guide states its own scope.

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
[SBP maintenance conventions](specific-business-processes/README.md) for process
notation and validation. Detailed API and operational instructions stay in the
application repositories rather than being copied here.
