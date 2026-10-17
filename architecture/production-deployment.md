# Production deployment architecture

## Scope and status

Demo/portfolio deployment, updated 2026-10-17. The owner deployed the backend
from `prep/prod` to a Singapore EC2 `t3.small` instance with Ubuntu 26.04. Its
public HTTPS origin is configured through deployment environment settings. The owner
verified API docs return 200, Nginx resumes after renewal, and the Certbot renewal
timer is scheduled. These are owner-reported runtime checks, not agent access to
the cloud account.

The frontend is an implemented React CSR SPA. Cloudflare Pages and its API proxy
are configured in source but not yet deployed or browser-verified. No paid domain
is required for the selected plan. API traffic will use the Pages hostname and a
Pages Function to reach Nginx on EC2. The chosen production branch is `prep/prod`.

The backend uses PostgreSQL, Redis, Celery workers/Beat, Brevo SMTP and Stripe
test-only Checkout. The owner selected filesystem media in `/app/media` persisted
by a Docker volume; Azure remains an implemented alternative, currently disabled.
Production Compose, Nginx TLS, and the demo reset wrapper are defined. The API
image excludes seed assets; the optional maintenance image includes them.

## Stack and hosting

| Layer          | Stack / host                                     | Responsibility                                                                                       |
| -------------- | ------------------------------------------------ | ---------------------------------------------------------------------------------------------------- |
| Frontend       | React 19, TypeScript, Vite, React Router         | Existing browser-rendered SPA; builds to `dist`                                                      |
| Static hosting | Cloudflare Pages + API Function                       | HTTPS frontend delivery; AWS Amplify is an alternative                                               |
| Backend host   | AWS EC2 with Docker Compose            | Owner-deployed single-server demo                                                                |
| Reverse proxy  | Nginx                                  | HTTPS termination, API forwarding, Django static assets                                              |
| Application    | Gunicorn runtime image; Django 6 and DRF         | Production API process                                                                               |
| Database       | PostgreSQL 17                                    | Private container and persistent volume                        |
| Broker         | Redis 7                                          | Private Celery broker with persistence                                                               |
| Workers        | Celery default worker and dedicated email worker | Payment recovery and registration/password-change OTP delivery                                       |
| Scheduler      | One Celery Beat instance                         | Existing payment recovery dispatch every 60 seconds                                                  |
| Media | Filesystem volume, served by Nginx | Selected deployment storage; Azure remains an optional integration |
| Email          | Brevo via Django SMTP backend                    | Existing provider and email integration; verify environment-specific configuration during deployment |
| Payments       | Stripe hosted Checkout and signed webhooks       | Existing test-only integration; no real payments                                                     |

The selected EC2 instance has 2 vCPUs and 2 GiB RAM. Monitor memory under API,
worker and seeding load. The owner reported $100 AWS credits and a Free Plan
ending March 28, 2027 or when credits run out. Credits are temporary; confirm
current balance and ongoing costs in AWS billing.

## How the proposed production setup would look

Each enclosing box represents a hosting boundary. Everything inside Docker
Compose runs as containers on the same EC2 virtual machine. Cloudflare Pages,
Azure Blob Storage, Brevo, and Stripe run outside that machine.

![Proposed production architecture showing Cloudflare, EC2 container boundaries, Azure, and external services](production-deployment.drawio.svg)

**Connections between the boundaries**

- The browser downloads the frontend from Pages and sends same-origin `/api/*`
  calls to a Pages Function. That Function forwards requests/cookies to Nginx on
  the EC2 HTTPS origin. Browser media reads use the backend host in filesystem mode.
- Nginx forwards API requests to Gunicorn and serves collected Django static
  assets. It does not serve the Pages frontend or proxy ordinary Azure media reads.
- Django reads/writes PostgreSQL, publishes OTP tasks to Redis, manages Azure
  media, and calls Stripe. Direct browser uploads are not introduced here.
- Beat publishes scheduled payment recovery tasks to Redis. The payment worker
  consumes the default queue, reconciles Stripe state, and updates PostgreSQL.
- The email worker consumes `registration-email`, checks OTP validity in
  PostgreSQL, and sends email through the existing Brevo SMTP integration.
- Stripe sends signed webhooks through Nginx to Django.

Only Nginx exposes public application ports; database, broker, application, and
worker services remain private. Internal port numbers above are
container ports, not public endpoints. The same-site domain/authentication
configuration below must be applied before frontend deployment.

## Domain and authentication decision

Selected free layout: frontend at the assigned `*.pages.dev` hostname; API origin
at the backend HTTPS origin configured for that environment. A Pages Function proxies `/api/*`, keeping
the browser on the Pages origin. DuckDNS alone does not make the two hosts
same-site. The Function preserves current host-only `SameSite=Lax` cookies and
never follows redirects server-side. Authentication payloads and cookie policy
remain unchanged; deployed cookie behavior must still be verified.

See the [API proxy process](../diagrams/specific-business-processes/deployment/same-origin-api-proxy.md)
and [frontend setup](../../frontend/docs/deployment/cloudflare-pages.md) for origin
configuration, failure handling, response caching and preview restrictions.

Configure exact CORS origins, allowed hosts, Stripe frontend origin, applicable
CSRF protections, and trusted proxy HTTPS handling. Nginx must overwrite forwarded
headers; Gunicorn must not be publicly reachable. Verify login, refresh, logout,
and registration cookies through the proxy.

## Container and operational boundaries

Retain backend `Dockerfile.dev` and `compose.dev.yaml`. `Dockerfile.prod` now
defines the runtime image; `compose.prod.yaml` and `deploy/nginx.conf.template` now define orchestration
and HTTPS proxying. Frontend hosting remains separate on Pages. A separate
`collectstatic` job prepares the static volume before Nginx starts. Reuse one versioned
backend image for API, workers, Beat, and a controlled one-shot migration job.
Production must use Gunicorn rather than Django runserver, without source bind
mounts or development reload/dependencies. Frontend hosting needs compiled assets,
not a production frontend container.

Only Nginx publishes public application ports. PostgreSQL, Redis, Gunicorn, and
workers stay private. Do not publish pgAdmin. Run exactly one Beat instance and
preserve the dedicated `registration-email` queue consumer. API health alone does
not establish email delivery or worker health.

PostgreSQL and Redis need persistent volumes. Volumes do not protect against
host loss.
PostgreSQL owns business state; Redis is the broker. Broker payloads can contain
OTPs and addresses and require restricted access and retention.

Keep secrets outside Git and images. Frontend `VITE_*` values are public build-time
configuration; only the public API URL belongs there. Azure, database, SMTP,
Django, and Stripe secrets remain server-side. Collect Django static files for
Nginx separately from user media. Implement certificate renewal, health checks,
restart policies, bounded logs, and queue monitoring.

This single-host setup accepts backend downtime during host failure or some
releases. It does not offer high availability or automatic horizontal scaling.
Managed PostgreSQL such as RDS is a later option, especially before handling real
customer data. Kubernetes, load balancers, and managed Redis are outside the
initial demo scope.

## Local production verification plan

1. Run frontend checks and build; inspect `dist` with Vite preview. Preview is a
   local verification server, not the eventual production hosting service.
2. Implement and run backend production containers and Nginx locally with isolated
   demo credentials and persistent test data.
3. Verify API HTTPS certificate trust and proxy headers. Frontend/API cookie
   integration must be tested with the separately hosted frontend and the chosen
   same-site domain or verified routing arrangement. Hosted staging is deferred.
4. Verify route refreshes, login/refresh/logout, registration and password-change
   OTPs, media, and Stripe test checkout/recovery.
5. Verify restart persistence, controlled migrations, and
   application rollback. Reverting an image does not reverse database changes.
6. Before public hosting, resolve domain/auth routing, configure Pages SPA fallback
   and API build variable, and verify signed Stripe webhooks at
   `/api/shop/stripe/webhook/`. Local reconciliation does not prove webhook delivery.

## Source references

- [Frontend build scripts](../../frontend/package.json), [entry point](../../frontend/src/app/main.tsx), [router](../../frontend/src/app/router.tsx), and [API client](../../frontend/src/app/api/axios.ts)
- [Backend development Compose](../../backend/compose.dev.yaml) and [Docker guide](../../backend/docs/docker-development.md)
- [Backend settings](../../backend/ecommerce_backend/settings.py) and [authentication views](../../backend/apps/authentication/views.py)
- [Stripe test contract](../../backend/docs/checkout/stripe-api.md)
- [Diagram index](../diagrams/README.md) and [diagramming specification](../diagrams/diagramming-specification.md)

Source links assume sibling frontend/backend/docs checkouts and may not resolve
when viewing only this repository online.

### Production image implementation

See the [backend production image guide](../../backend/docs/docker-production.md)
for build and inspection commands. Frontend production build, type-check, and lint
passed; the project owner confirmed testing the frontend preview on port 4173.

The production image built successfully. Isolated checks verified Gunicorn 26.2.0,
non-root UID/GID 10001, dependency consistency, Django system checks, and absence
of `.env*` files under `/app`. Database and integration tests remain pending.

### Production Compose implementation

The backend defines production Compose with separate development/production
projects and volumes, private service ports, migration/static collection jobs,
Nginx TLS termination, and production Django settings. Nginx serves backend
routes and assets; frontend hosting is separate. The overview image shows the
planned public topology and omits the static collection release job. Follow the
[backend run guide](../../backend/docs/docker-production.md) for certificate, build,
and startup commands. The owner subsequently started the backend and verified
public HTTPS and certificate renewal. No business endpoint or payload contract was changed.

### Disposable portfolio reset

The optional maintenance image contains taxonomy and seed source images; regular
API images exclude them. The `reset-demo.sh` wrapper validates targets, asks for
confirmation, stops application services, clears the dedicated broker, resets and
reseeds the schema/media, and restarts services only after success. `--dry-run`
performs preflight without
service shutdown or data deletion. Production settings now honor the explicit
`ALLOW_DEV_RESET` environment toggle (default false). See the maintained
[reset workflow](../diagrams/specific-business-processes/development/reset-development.md)
and [backend run guide](../../backend/docs/docker-production.md).

### Pages proxy implementation

Source implementation includes a root Pages Function and `_routes.json` limiting
invocation to API paths. Twelve focused mocked tests passed; type-check/build and
lint passed. Edge execution and real browser cookies remain unverified. Direct
admin/media/static and signed webhook traffic still use the backend hostname.
The host renewal hooks currently stop Nginx briefly for standalone certificate
validation; this introduces a short outage. Backend IP throttling sees the
immediate Cloudflare peer; real browser IP propagation remains a separate trust
configuration decision. Existing overview artwork retains Azure as an optional
storage alternative; deployed media currently uses a filesystem volume.

Proxy verification: all 189 tests passed with two workers, including 12 focused
proxy cases. Build with `VITE_API_URL=/` and lint passed. An unrestricted concurrent
test run had 15 timing/UI lookup failures. Both new Mermaid process fences parsed
with Mermaid 12.0.0. The updated overview SVG and embedded draw.io source parsed
as XML; rendering and visual/host verification remain pending.

Deployment-specific addresses are intentionally omitted from maintained source
and documentation. Configure `API_ORIGIN` in Cloudflare runtime variables,
`VITE_API_URL=/` in Pages build variables, and backend/frontend host and origin
values in the EC2 `.env`. Proxy tests use reserved `.test` fixture domains and do
not depend on any deployment.
