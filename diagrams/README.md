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

The backend owns the [per-app ERD guides and editable Draw.io diagrams](../../backend/docs/erd/README.md),
alongside its models and migrations. Each guide explains the app’s models, fields,
relationships, constraints, and lifecycle. Keep the app list and schema details in
that backend index instead of maintaining a second copy here.

[Backend checkout architecture](../../backend/docs/architecture/checkout-models.md)
and the [wallet ledger contract](../../backend/docs/architecture/wallet-ledger.md)
explain deeper service rules. The [original editable ERD](../drawio/data-models/e-commerce-entity-relationship-diagram.drawio)
remains a separate historical visual reference, not the maintained field contract.

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
