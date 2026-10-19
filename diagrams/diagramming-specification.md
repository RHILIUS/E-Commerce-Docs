# System diagram specification

## Business-process decision

Use a swimlane business-process flowchart as the primary diagram for a specific
business requirement.

## Swimlane conventions

- Write fenced `mermaid` blocks using `swimlane-beta TB` and named `subgraph`
  lanes for actual responsibility partitions. Record the Mermaid version used
  during validation; this syntax is not guaranteed to work in every Markdown host.
  Ordinary `flowchart` subgraphs are an alternative only when grouped ownership
  is sufficient: cross-lane edges can prevent a rigid lane layout.
  Inspect the render and split large flows when necessary.
- Assign each action to its owner. Customer interactions belong in Frontend.
  Backend owns authorization, business validation and persistence. Add external
  service lanes only when their interactions need to be shown explicitly.
- Use rounded start/end nodes, rectangles for actions and diamonds for decisions.
  Label every decision outcome with Yes/No or explicit mutually exclusive cases.
- Label lane-crossing requests/responses. Put exact API routes and payload fields
  in an adjacent handoff table when embedding them would make nodes unreadable.
- Show where errors stop progression, what the customer sees and where recovery
  resumes. Never let an invalid branch fall through to success.
- Identify transaction scope, commit/rollback, idempotency, ownership and durable
  state changes when applicable. Common guards may be documented beside the flow
  to avoid repeating identical branches in each phase.
- Split long workflows into named phases with explicit continuation points.
  Keep rules and status tables adjacent to their diagrams. Avoid decorative lanes
  or undocumented API calls introduced solely to improve layout.
- Quote node labels and keep punctuation simple. Do not use raw semicolons in
  sequence message labels, where they can be interpreted as statement delimiters.

  ### Example

  ```mermaid
  swimlane-beta TB
    subgraph D_UI["Frontend"]
        D_open["From C: open confirmation by reference"]
        D_show(["Order placed<br/>Payment due on delivery"])
        D_error(["Show unavailable or request error<br/>Retry reading without placing again"])
    end
    subgraph D_API["Backend"]
        D_get{"Authenticated customer<br/>owns this checkout?"}
        D_read["Read saved checkout and order snapshots<br/>Do not recalculate current prices or FX"]
        D_get -->|Yes| D_read
    end
    D_open -->|GET confirmation| D_get
    D_get -->|No: 401 or 404| D_error
    D_read -->|200| D_show
    D_read -->|Request failure| D_error
  ```

## Acceptance and maintenance

A diagram update is reviewable when:

- Every decision has an outcome and every phase has a clear entry/exit.
- Success cannot be reached through an invalid or failed branch.
- Retries, uncertain results and atomic changes match the approved contract.
- Implemented behavior is distinguished from approved future design.
- All Mermaid fences parse and render with a recorded Mermaid version.
- Rendered labels are legible, unclipped and ownership is clear. Check the actual
  Markdown host too when available; local rendering does not prove host compatibility.
- Relative links resolve and the diagram index points at the maintained document.

Update affected business-process diagrams with changes to API contracts, payment,
stock, ownership, or cross-application flows. Maintain those processes directly in
Mermaid Markdown; separate process-image exports are not required.

## Entity-relationship diagrams

ERDs use editable Draw.io SVGs rather than Mermaid. Maintain one image at
`erd/diagrams/{app_label}.drawio.svg` and one explanation at
`erd/guides/{app_label}.md`, with the same lowercase Django app label for the
file stem and the image’s single internal Draw.io page. Embed the image from its
guide using `../diagrams/{app_label}.drawio.svg`; Markdown cannot select a page
from the editable data of a multipage SVG.

Full entities belong to their owning app. Cross-app relationships use name-only
reference rectangles and links to the owning guide. Model database table names,
concrete column names, inherited fields, primary/foreign keys, and Crow’s Foot
cardinality must match the verified implementation. Explain every owned model and
field, plus relationships, deletion behavior, model-level constraints, and relevant
persistence/lifecycle rules in its guide. Keep proposed behavior separate.

The SVG’s visible image and embedded editable document must describe the same
app schema. Validate XML and graph references, inspect the exact rendered SVG,
and check editability and intended Markdown embedding when available. Report
unavailable checks accurately. Keep execution evidence in handoffs or PRs.
The [diagram index](README.md#maintained-data-model-explanations) links the
maintained app guides. Business-process diagrams continue to use Mermaid.
