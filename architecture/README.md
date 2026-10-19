# System architecture

Architecture explains which part of the system owns a responsibility and how the
parts communicate. Read the [system overview](../system-overview.md) first if the
application is unfamiliar.

- [Production deployment](production-deployment.md) describes the implemented
  Pages/EC2 configuration and its operational boundaries. Deployment-specific
  credentials, DNS, certificates, and provider settings still need configuration.
- [Same-origin API proxy](../diagrams/specific-business-processes/deployment/same-origin-api-proxy.md)
  follows one browser request and its cookies across the frontend and backend.
- [Backend architecture](../../backend/docs/architecture/README.md) explains
  code responsibilities and persistence inside the API.
- [Frontend architecture](../../frontend/docs/architecture.md) explains routing,
  browser state, and API calls inside the website.

The [editable architecture visual](../drawio/architecture/production-deployment.drawio.svg)
is stored with the draw.io references and embedded by the production guide. Its
content is unchanged by this reorganization; it is a high-level visual, not a
complete service or release-job inventory.
