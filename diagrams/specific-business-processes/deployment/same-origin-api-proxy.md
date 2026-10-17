# Same-origin API proxy on Cloudflare Pages

## Scope and implementation status

Implemented in frontend source, not deployed or browser-verified: Pages Function
`functions/api/[[path]].ts` forwards API requests to the configured HTTPS EC2
backend. React remains CSR. The owner verified the EC2 public endpoint and
certificate renewal at the configured backend HTTPS origin. No business API payload,
authorization rule, data model or authentication-cookie policy changes here.

These are activity-style Mermaid swimlane flowcharts, not strict UML activity
diagrams. See the [diagramming specification](../../diagramming-specification.md).
Use the assigned Pages hostname from deployment configuration when configuring
EC2 origins and Stripe returns. Actual hostnames are not maintained in this document.

## A. Forward a browser API request

```mermaid
swimlane-beta TB
    subgraph A_UI["Frontend and browser"]
        A_start(["User action requests /api path<br/>Attach Pages cookies and bearer token where applicable"])
        A_config_error(["503 configuration error<br/>Operator fixes Pages binding and redeploys"])
        A_unknown(["502 request outcome may be unknown<br/>Use feature recovery instead of blind resubmission"])
    end
    subgraph A_EDGE["Cloudflare Pages Function"]
        A_valid{"API_ORIGIN is a different<br/>valid HTTPS origin only?"}
        A_forward["Retain method, path, query, headers and streaming body<br/>Set upstream Host from configured origin<br/>Disable cache and automatic redirects"]
        A_received{"Upstream response received?"}
        A_next(["Continue to B with response"])
    end
    subgraph A_PROXY["EC2 Nginx"]
        A_tls["Terminate HTTPS<br/>Overwrite proxy headers and forward privately"]
    end
    subgraph A_API["Backend"]
        A_handle["Validate host and business request<br/>Authenticate and authorize<br/>Perform existing feature transaction if needed"]
        A_result["Return existing status, payload and cookie headers<br/>Errors do not imply success"]
    end
    A_start -->|Same-origin HTTPS /api request| A_valid
    A_valid -->|No| A_config_error
    A_valid -->|Yes| A_forward
    A_forward -->|HTTPS to configured EC2 origin| A_tls
    A_tls -->|Private HTTP with trusted HTTPS header| A_handle
    A_handle --> A_result
    A_result -->|HTTP response through Nginx| A_received
    A_forward -.->|Connection or TLS failure| A_received
    A_received -->|No: no automatic retry| A_unknown
    A_received -->|Yes: includes backend errors| A_next
```

## B. Return status and host-only cookies

```mermaid
swimlane-beta TB
    subgraph B_EDGE["Cloudflare Pages Function"]
        B_start(["From A: upstream response received"])
        B_copy["Copy status, body and independent Set-Cookie headers<br/>Disable browser and CDN caching"]
        B_redirect{"Location points to backend /api path?"}
        B_rewrite["Rewrite API Location to Pages origin"]
        B_keep["Retain other Location or no Location"]
        B_return["Return response without interpreting business success"]
    end
    subgraph B_UI["Frontend and browser"]
        B_cookie["Apply host-only cookie set or deletion to Pages hostname<br/>Keep Secure, HttpOnly and SameSite attributes"]
        B_render["Handle existing success or error contract<br/>Future /api calls send Pages cookies"]
        B_end(["Request complete"])
    end
    B_start --> B_copy --> B_redirect
    B_redirect -->|Yes| B_rewrite --> B_return
    B_redirect -->|No| B_keep --> B_return
    B_return -->|HTTPS status, body and cookie headers| B_cookie
    B_cookie --> B_render --> B_end
```

## Handoff contract and guards

| Handoff | Verified source contract |
| --- | --- |
| Frontend to Pages | Build `VITE_API_URL=/`; existing `/api/...` routes; credentials enabled in Axios |
| Function configuration | `API_ORIGIN=<BACKEND_HTTPS_ORIGIN>`, runtime binding, no path/credentials/query/fragment; must differ from Pages origin |
| Function to EC2 | Same HTTP method, path/query, streaming body, Cookie, Authorization and Origin; upstream Host from target; HTTPS certificate verification remains enabled |
| Nginx to Django | Private Gunicorn port; Nginx overwrites forwarded HTTPS/IP headers; Django retains backend hostname in ALLOWED_HOSTS |
| Django to browser through Function | Existing status/body and separate Set-Cookie headers; current authentication cookies omit Domain and use Path=/, HttpOnly, SameSite=Lax and production Secure |
| API redirect | Backend /api Location rewritten to Pages; redirects are never followed by Function fetch |
| Failures | Invalid binding: non-cacheable 503; transport failure: non-cacheable 502 with uncertain outcome; backend validation/auth/permission failures retain their status and payload |

Frontend `/api` traffic alone invokes the Function through `_routes.json`.
Assets and ordinary SPA routes stay static. Configure Functions quota failures to
fail closed where the platform exposes that option. Preview origins must not be
implicitly trusted; previews require a separate backend or explicit acceptance
that they mutate the demo database.

`<BACKEND_HTTPS_ORIGIN>` is a documentation placeholder, not a literal value.
Configure it through Cloudflare runtime variables; configure the actual frontend
origin in backend `.env`. A local frontend `.env` does not supply deployed runtime
bindings. Test fixture domains are independent of this configuration.

The proxy performs no persistence and defines no transaction boundary. Backend
feature transactions, ownership checks, idempotency keys and durable recovery
remain unchanged. No automatic retries are made for mutations: a failed response
can follow a successful commit. Use the relevant feature's recovery process.

Browser cookie storage belongs to the Pages hostname, not DuckDNS. Cookies
previously stored under DuckDNS are not migrated. Login is required on the Pages
origin. Browser testing must verify refresh/logout and registration cookies.
Backend cookie Domain changes would require rechecking this proxy contract.

Django admin, Swagger static dependencies, media/static routes and Stripe webhook
delivery remain on the backend host. Media response bodies/URLs are not rewritten.
Stripe return origin must name Pages, while signed webhooks target EC2 directly.
Nginx currently identifies the immediate network peer for IP-based throttling;
through this proxy, visitors can share a Cloudflare address. End-user IP trust
propagation remains unresolved and must not be implemented by trusting arbitrary
forwarded headers. The proxy does not solve backend availability or free-tier
quota exhaustion.

## References and validation

- [Function](../../../../frontend/functions/api/[[path]].ts)
- [Deployment guide](../../../../frontend/docs/deployment/cloudflare-pages.md)
- [API client](../../../../frontend/src/app/api/axios.ts)
- [Nginx configuration](../../../../backend/deploy/nginx.conf.template)
- [Authentication views](../../../../backend/apps/authentication/views.py)
- [Production architecture](../../../architecture/production-deployment.md)

Twelve mocked Fetch tests passed for forwarding, independent cookies, multipart
uploads, redirects, configuration rejection, validation errors and transport
failures. Type-check/build and lint passed; the build retains its existing large
JavaScript chunk warning. Cloudflare execution and browser checks are pending.
Both Mermaid fences parsed successfully using Mermaid 12.0.0 with JSDOM.
Rendering and visual inspection were not performed because no browser renderer
is available; Markdown-host support remains unverified. Parsing does not establish
visual validation. All 189 tests in 23 files passed with two Vitest workers; an
unrestricted concurrent run had 15 timing/UI lookup failures. No unrelated UI
code was changed.
