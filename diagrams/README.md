# E-Commerce diagrams

## Workflow diagram transition

UML sequence diagrams written in **Mermaid** are now the primary format for
feature-specific frontend/backend workflows. They show the customer, frontend,
backend, database and external services, including request order, validation,
transactions, alternative outcomes and retries.

New feature workflows belong in [`UML sequence diagram/`](<UML sequence diagram/>).
This replaces `specific-business-processes/` as the destination for new detailed
workflow diagrams. The existing authentication and ecommerce Draw.io files remain
as legacy references while their flows are migrated incrementally; they have not
yet been converted. When a replacement is approved, link it here and mark the
older diagram as superseded so there is one maintained workflow description.

This transition applies to detailed feature workflows. ERDs, class diagrams and
the general business-process diagram continue to document data structure, static
relationships and the overall business process.

## Diagram index

| Diagram | Purpose / status |
| --- | --- |
| [COD checkout](<UML sequence diagram/cod-checkout.md>) | Approved sequence and API design; implementation pending |
| [Authentication business process](specific-business-processes/authentication.drawio) | Legacy reference; sequence replacement pending |
| [Ecommerce business process](specific-business-processes/ecommerce.drawio) | Legacy reference; sequence replacement pending |
| [Entity-relationship diagram](e-commerce-entity-relationship-diagram.drawio) | Data entities and relationships |
| [Class diagram](e-commerce-class-diagram.drawio) | Static model structure |
| [General business process](e-commerce-general-business-process.drawio) | Overall customer and store journey |

## Authoring and maintenance

- Store one feature or flow per Markdown file using a descriptive kebab-case name,
  for example `cod-checkout.md`. Keep the Mermaid source inside a fenced
  `mermaid` block using `sequenceDiagram` so it renders in GitHub.
- Start with the flow's scope and status: proposed, approved design, or implemented.
  Do not present a design as current behavior before it is implemented.
- Name participants by their actual responsibility. Label API calls with the
  agreed HTTP method and route, and distinguish calls from responses.
- Include meaningful `alt`, `opt` or `loop` sections for failures, retries and
  optional steps. Show transaction boundaries and when state changes persist.
- Explain important invariants or deferred behavior below the diagram when they
  cannot be represented clearly in the sequence itself.
- Once implemented, reconcile the diagram with frontend/backend behavior and
  update it alongside API, ownership, payment, stock or transaction-flow changes.
- Preview the Mermaid rendering and check local links when editing. Keep the
  editable source as the maintained artifact; screenshots are optional exports.

No external drawing application is required to edit the new sequence diagrams.
