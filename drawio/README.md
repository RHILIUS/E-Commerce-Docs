# Editable draw.io references

These are the project's original editable visual references. They are kept apart
from the maintained business-process and per-app ERD guides so their role is clear. They may describe
current behavior, partial work, or future direction; storage here does not mean a
file is obsolete or approved as an implementation contract.

Open `.drawio` files with diagrams.net or a compatible editor. Editable
`.drawio.svg` files include both the rendered SVG and draw.io content. The
architecture visual lives in `../architecture/asset/`; individual process visuals
live in `../specific-business-processes/`.

| Reference                                                                                           | Contents                                                        |
| --------------------------------------------------------------------------------------------------- | --------------------------------------------------------------- |
| [Production architecture](../architecture/asset/production-deployment.drawio.svg)                            | Hosting visual embedded by the architecture guide               |
| [Class diagrams](class-diagrams/e-commerce-class-diagram.drawio)                                    | General, checkout, store, and shipment pages                    |
| [General business processes](general-business-processes/e-commerce-general-business-process.drawio) | Order, signup, and online-checkout design flows                 |
| [Login](../specific-business-processes/authentication/login.drawio.svg) | Login process |
| [Signup](../specific-business-processes/authentication/signup.drawio.svg) | Signup process |
| [Logout](../specific-business-processes/authentication/logout.drawio.svg) | Logout process |
| [JWT interceptor](../specific-business-processes/authentication/jwt-interceptor.drawio.svg) | Token interception process |
| [Ecommerce process](../specific-business-processes/ecommerce/landing-page.drawio.svg)                         | Landing-page flow                                               |

Use the [backend ERD index](../../backend/docs/erd/README.md) for maintained
data-model guides and editable diagrams.

Use [the maintained diagram index](../diagrams/README.md) for current Mermaid
business processes and per-app Draw.io ERDs with model/field explanations, and
[planning](../planning/README.md) for future policies.
When implementing an idea from these references, check the current request,
relevant code, and API contracts first. A historical drawing alone does not add an
endpoint, authorize a mutation, or settle an unresolved product decision.
