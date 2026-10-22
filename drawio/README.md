# Editable draw.io references

These are the project's original editable visual references. They are kept apart
from the maintained business-process and per-app ERD guides so their role is clear. They may describe
current behavior, partial work, or future direction; storage here does not mean a
file is obsolete or approved as an implementation contract.

The files retain their original bytes and internal pages. Open `.drawio` files
with diagrams.net or a compatible editor. The architecture `.drawio.svg`, stored in `../architecture/asset/`, includes
its editable draw.io content as well as the rendered SVG.

| Reference                                                                                           | Contents                                                        |
| --------------------------------------------------------------------------------------------------- | --------------------------------------------------------------- |
| [Production architecture](../architecture/asset/production-deployment.drawio.svg)                            | Hosting visual embedded by the architecture guide               |
| [Class diagrams](class-diagrams/e-commerce-class-diagram.drawio)                                    | General, checkout, store, and shipment pages                    |
| [Data models](data-models/e-commerce-entity-relationship-diagram.drawio)                            | Full-system and feature ERD pages, including partial views      |
| [General business processes](general-business-processes/e-commerce-general-business-process.drawio) | Order, signup, and online-checkout design flows                 |
| [Authentication processes](specific-business-processes/authentication/authentication.drawio)        | Login, signup, logout, token interception, and auth redirection |
| [Ecommerce process](specific-business-processes/ecommerce/ecommerce.drawio)                         | Landing-page flow                                               |

Use [the maintained diagram index](../diagrams/README.md) for current Mermaid
business processes and per-app Draw.io ERDs with model/field explanations, and
[planning](../planning/README.md) for future policies.
When implementing an idea from these references, check the current request,
relevant code, and API contracts first. A historical drawing alone does not add an
endpoint, authorize a mutation, or settle an unresolved product decision.
