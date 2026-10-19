# Maintained system diagrams

These guides explain cross-application behavior and the data that supports it.
Each document states its scope and implementation status. Read the
[system overview](../system-overview.md) first if the project is unfamiliar.

Processes use activity-style Mermaid swimlane flowcharts to show actual owners,
decisions, requests, persistence, and recovery. They are not strict UML activity
diagrams. Sequence diagrams can supplement request ordering. ERDs explain stored
entities and relationships using editable Draw.io SVGs and per-app Markdown guides.
Follow the [diagramming specification](diagramming-specification.md) when editing them.

## Implemented processes

| Area           | Guide                                                                                      | Scope                                                                          |
| -------------- | ------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------ |
| Authentication | [Registration email](specific-business-processes/authentication/registration-otp-email.md) | Code issuance, background delivery, verification, and account-creation handoff |
| Account        | [Change password](specific-business-processes/account/change-password.md)                  | Challenge, resend, queued email, confirmation, and session invalidation        |
| Checkout       | [Cash on delivery](specific-business-processes/checkout/cod-checkout.md)                   | Review, recipient selection, atomic placement, and saved confirmation          |
| Checkout       | [Stripe test checkout](specific-business-processes/checkout/demo-online-checkout.md)       | Pending orders/holds, verified payment, store credits, and payment recovery    |
| Deployment     | [Same-origin API proxy](specific-business-processes/deployment/same-origin-api-proxy.md)   | Browser requests and cookies across Pages, Nginx, and Django                   |

## Maintained data-model explanations

Each app has a guide in `erd/guides/` and a matching editable SVG in
`erd/diagrams/`. The guide renders the diagram and explains every owned model,
column, relationship, constraint, and relevant lifecycle. Name-only rectangles
refer to entities defined by another app. Open each SVG in Draw.io to edit its
single app-named page; Markdown displays that file’s exported image.

| App            | Guide                                          | Scope                                                                                                  |
| -------------- | ---------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| Authentication | [Authentication](erd/guides/authentication.md) | Accounts, personal addresses, registration verification, password-change budgets, and permission joins |
| Account        | [Account](erd/guides/account.md)               | Saved delivery recipients, default selection, and soft deletion                                        |
| Store          | [Store](erd/guides/store.md)                   | Store identity/currency, addresses, and account membership                                             |
| Ecommerce      | [Ecommerce](erd/guides/ecommerce.md)           | Taxonomy, specifications, catalog, carts, purchase snapshots, and order milestones                     |
| Checkout       | [Checkout](erd/guides/checkout.md)             | Purchase identity, Stripe test attempts/events, and store-order refunds                                |
| Inventory      | [Inventory](erd/guides/inventory.md)           | Per-line inventory holds and payment resolution                                                        |
| Wallet         | [Wallet](erd/guides/wallet.md)                 | Simulated USD ownership, accounting buckets, and immutable journal                                     |

The [backend checkout architecture](../../backend/docs/architecture/checkout-models.md)
and [wallet ledger contract](../../backend/docs/architecture/wallet-ledger.md)
provide deeper service rules. The [original editable ERD](../drawio/data-models/e-commerce-entity-relationship-diagram.drawio)
is a separate historical visual reference; it is not the maintained per-app field
contract. Built-in Django auth/session/admin/token tables are outside the owned
schema of these seven project apps; authentication’s generated user-permission
join tables are included in its guide.

## Related references and planning

- [Future settlement, withdrawals, and returns](../planning/settlement-and-withdrawals.md)
  describes proposals, separately from implemented payment phases.
- [Editable draw.io index](../drawio/README.md) lists personal visual references.
- [Backend disposable reset guide](../../backend/docs/operations/reset-environment.md)
  describes implemented operator behavior. This repository does not currently
  contain the formerly indexed reset-process Markdown document.

A missing guide is not evidence that a workflow is implemented or absent. Use the
current code and maintained application contracts, and add a scoped document when
that work is actually requested. Do not keep index links to nonexistent documents.
