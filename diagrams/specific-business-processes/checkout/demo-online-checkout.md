# Stripe test checkout, recovery, and store credits

**Status: implemented in frontend and backend source.** Customers can buy from
multiple stores with one hosted Stripe test payment. The backend saves each
store's pending share in the demo USD ledger. No real money moves, and the browser
return from Stripe does not establish payment success.

## Understand the flow first

1. Review USD prices and choose the delivery recipient.
2. Save pending store orders and stock holds, then create or recover one hosted
   payment session with a stable key.
3. Check provider facts through signed webhooks or reconciliation.
4. Verified payment finalizes stock, orders, cart cleanup, and one ledger receipt
   in the same local transaction. Uncertain outcomes retain recovery.
5. Read saved confirmation and order history. Eligible customer cancellation uses
   the separate original-card refund path described at the end.

A **hold** protects units while payment is unresolved. A **ledger receipt** records
simulated treasury value and matching pending store liabilities. A **webhook** is
a signed Stripe notification; **reconciliation** reads provider state and compares
it with saved purchase facts. Stripe and PostgreSQL do not share a transaction.

Delivery, seven-day settlement, returns, customer wallet spending, and withdrawals
are not part of this implemented flow.

## Scope, currency, and ownership

This guide covers USD test-card checkout, durable stock holds, provider-event
receipt, periodic payment recovery, and pending store credits. It does not create
Stripe Connect accounts or make real transfers. Stores retain separate wallets
even when they share members.

The backend derives ownership from authentication, validates recipient/cart
ownership, and computes the accepted total. Prices use source-to-USD multipliers,
round each unit half-up to cents, then multiply by quantity. Source currency,
original price, dated rates, names, and delivery details are copied into purchase
snapshots. Client totals cannot replace those values.

See the [Stripe API contract](../../../../backend/docs/api/checkout/stripe.md),
[checkout models](../../../../backend/docs/architecture/checkout-models.md), and
[ledger service](../../../../backend/docs/architecture/wallet-ledger.md).
The diagrams below are activity-style Mermaid swimlane flowcharts. Read
[the notation](../../diagramming-specification.md) for their ownership conventions.

## A. Review, reserve and start payment

### A1. Save or recover the pending purchase

```mermaid
swimlane-beta TB
  subgraph A1_UI["Frontend"]
    A1_start(["Review USD total and choose recipient"])
    A1_error(["Show input or conflict error<br/>Preserve recovery for an unknown outcome"])
  end
  subgraph A1_API["Backend"]
    A1_valid{"Authentication and request shape valid?"}
    A1_begin["Begin transaction and lock customer"]
    A1_key{"Existing request key?"}
    A1_same{"Same Stripe request fingerprint?"}
    A1_check{"Signed preview, ownership and locked inventory valid?"}
    A1_save["Commit pending checkout, orders, snapshots and holds"]
    A1_reuse["End transaction and use saved attempt"]
    A1_rollback["Roll back without a new purchase"]
    A1_next(["Continue A2 with the saved attempt"])
    A1_valid -->|Yes| A1_begin --> A1_key
    A1_key -->|Yes| A1_same
    A1_same -->|Yes| A1_reuse --> A1_next
    A1_same -->|No| A1_rollback
    A1_key -->|No| A1_check
    A1_check -->|Yes| A1_save
    A1_check -->|No| A1_rollback
    A1_save -->|Committed| A1_next
    A1_save -->|Write or commit failure| A1_rollback
  end
  A1_start -->|POST placement with exact body and retry key| A1_valid
  A1_valid -->|No: 400 or 401| A1_error
  A1_rollback -->|Validation or conflict response| A1_error
```

A1 saves or recovers one purchase. A2 works on that saved attempt; it never
starts another purchase because a response or provider result is uncertain.

### A2. Create or reconcile the hosted payment session

```mermaid
swimlane-beta TB
  subgraph A2_API["Backend"]
    A2_start(["From A1: saved attempt"])
    A2_pending{"Attempt still pending?"}
    A2_session["Create or reconcile with stable key and parameters<br/>No provider call holds database locks"]
    A2_apply["Validate provider facts<br/>Use phase B or C local transaction"]
    A2_close["Validate confirmed creation rejection<br/>Atomically close and release holds"]
    A2_start --> A2_pending
    A2_pending -->|Yes| A2_session
  end
  subgraph A2_STRIPE["Stripe test environment"]
    A2_result{"Session creation or retrieval result?"}
  end
  subgraph A2_UI["Frontend"]
    A2_pay(["Open hosted test checkout"])
    A2_status(["Show saved receipt or recovery status"])
    A2_error(["Show error and recover saved attempt<br/>Keep holds unless safe closure confirmed"])
  end
  A2_pending -->|No: saved terminal state| A2_status
  A2_session -->|Create or retrieve session| A2_result
  A2_result -->|Pending and hosted URL available| A2_pay
  A2_result -->|Verified paid or terminal unpaid state| A2_apply
  A2_apply -->|Saved result| A2_status
  A2_apply -->|Mismatch or local failure| A2_error
  A2_result -->|Confirmed unsupported-parameter creation rejection| A2_close
  A2_close -->|Definitive 400 or unresolved mismatch| A2_error
  A2_result -->|Request failure or uncertain result| A2_error
```

Only the narrowly verified pre-creation rejection can close that attempt as a
definitive placement error. Other failures preserve recovery; they do not imply
that payment failed or that its stock hold can be discarded.

Within a database transaction, authorize customer/recipient/cart ownership,
validate the signed preview and its USD total, lock products in stable order,
and reserve all selected quantities or roll back the entire selection. Persist
one Checkout, one Order per store, item snapshots and one pending PaymentAttempt.
Replaying a request key returns the same attempt; changed contents conflict.

Call Stripe outside database locks. Persist provider IDs and use the attempt's
stable idempotency key to recover from timeouts. Do not create a second payable
session while an earlier result is unknown. Cart writers freeze selected lines until payment resolves; unselected lines
remain editable. Successful finalization removes the purchased selected lines.
A confirmed unpaid closure releases their holds without silently buying again.

## B. Confirm one payment for all stores

### B1. Validate and retain the notification

```mermaid
swimlane-beta TB
  subgraph B1_STRIPE["Stripe test environment"]
    B1_event(["Payment notification"])
  end
  subgraph B1_API["Backend"]
    B1_verify{"Raw signature and test mode valid?"}
    B1_supported{"Supported event type?"}
    B1_routing{"Required routing identifiers valid?"}
    B1_reject(["Reject invalid delivery"])
    B1_ignore(["Acknowledge unsupported type<br/>No purchase change"])
    B1_inbox["Persist event uniquely for recovery"]
    B1_processed{"Event already processed?"}
    B1_ack(["Acknowledge previously handled event"])
    B1_next(["Continue B2 with saved event and attempt"])
    B1_verify -->|No| B1_reject
    B1_verify -->|Yes| B1_supported
    B1_supported -->|No| B1_ignore
    B1_supported -->|Yes| B1_routing
    B1_routing -->|No| B1_reject
    B1_routing -->|Yes| B1_inbox --> B1_processed
    B1_processed -->|Yes| B1_ack
    B1_processed -->|No| B1_next
  end
  B1_event -->|Raw event and Stripe-Signature| B1_verify
```

### B2. Reconcile and finalize the local result

```mermaid
swimlane-beta TB
  subgraph B2_API["Backend"]
    B2_start(["From B1: durable event and saved attempt"])
    B2_fetch["Retrieve provider state outside database locks"]
    B2_check{"Current matching provider state?"}
    B2_commit["Transaction: lock and recheck purchase and holds<br/>Post receipt, deduct stock, consume holds<br/>Mark orders paid, clean cart and complete checkout"]
    B2_close["Use phase C safe-unpaid-closure transaction"]
    B2_done(["Local result committed or already applied"])
    B2_rollback["Roll back local finalization changes"]
    B2_wait(["Keep pending attempt<br/>Retain failed inbox when processing failed"])
    B2_start --> B2_fetch
    B2_check -->|Verified paid| B2_commit
    B2_check -->|Confirmed terminal unpaid| B2_close
    B2_check -->|Pending or mismatched| B2_wait
    B2_commit -->|Committed or replay| B2_done
    B2_commit -->|Write or commit failure| B2_rollback --> B2_wait
    B2_close -->|Committed or replay| B2_done
    B2_close -->|Mismatch or local failure| B2_wait
  end
  subgraph B2_STRIPE["Stripe test environment"]
    B2_state["Return current session and intent state"]
  end
  subgraph B2_UI["Frontend"]
    B2_show(["Show saved confirmation or recovery status"])
  end
  B2_fetch -->|Session retrieval| B2_state
  B2_state -->|Provider facts| B2_check
  B2_state -->|Provider failure| B2_wait
  B2_done -->|Later authorized status read| B2_show
  B2_wait -->|Later authorized read: no invented success| B2_show
```

Verify the raw-body signature and test environment. Durably deduplicate provider
event IDs; retries of the same payment through different events must also resolve
to one business operation. Acknowledge accepted events only after durable receipt and the processing
attempt. Processing failures are caught so the accepted notification can still
return 200 and remain recoverable. An unsupported event type is acknowledged
without a purchase change. A processed notification is not proof of a paid
purchase; a pending attempt can still need later reconciliation. Recover
unprocessed events and pending attempts through the periodic scan. Retrieve provider state where
needed and match session/intent, checkout, amount, currency and paid status.
Redirects and a session-completed event alone are not proof of payment.
[Stripe fulfillment guidance](https://docs.stripe.com/checkout/fulfillment)

In one database transaction, lock the checkout, reservations/products and financial
accounts; mark every store order paid, consume reservations and decrement stock,
clean up the frozen purchased cart quantities, post the balanced ledger receipt,
link the ledger receipt and complete the checkout. Store settlement records are deferred; the journal retains each store allocation. Roll back all local changes
on failure and retry. Do not increment sold counts here.

“Atomic multi-store payment” means one charge and all-or-nothing local finalization.
Stripe and PostgreSQL do not share a transaction: a successful charge may temporarily
show as processing while recovery retries local finalization. Never charge again
because that transaction failed. A paid attempt whose holds were already released
requires investigation, not unchecked stock consumption. Automatic compensation
for payment mismatches is not implemented; it is distinct from the eligible
customer cancellation-refund workflow.
Out-of-order failure events must never regress an already paid checkout.

## C. Recover abandoned or uncertain payments

**Implemented:** Beat dispatches payment recovery every 60 seconds through Redis;
the worker calls `reconcile_stripe` with `limit=100`. The following is an
activity-style Mermaid swimlane business-process flowchart, not strict UML.

```mermaid
swimlane-beta TB
  subgraph C_BEAT["Celery Beat"]
    C_tick(["Recovery interval due"])
    C_send["Dispatch payment recovery task"]
    C_tick --> C_send
  end
  subgraph C_REDIS["Redis broker"]
    C_queue["Queue task for worker"]
  end
  subgraph C_WORK["Backend Celery worker"]
    C_scan["Read bounded unfinished events and pending attempts"]
    C_fetch["Reconcile next eligible row"]
    C_check["Validate provider state and local snapshots"]
    C_result{"Recovery result?"}
    C_paid["Phase B transaction<br/>Finalize paid state once"]
    C_terminal["Lock and recheck<br/>Release terminal unpaid holds"]
    C_retry(["Retain state<br/>Retry on later scan"])
    C_done(["Continue batch<br/>Then await next scan"])
    C_scan -->|For each eligible row| C_fetch
    C_check --> C_result
    C_result -->|Paid and matching| C_paid
    C_result -->|Confirmed terminal unpaid| C_terminal
    C_result -->|Still pending, mismatch or provider error| C_retry
    C_paid -->|Committed or already applied| C_done
    C_paid -->|Failure: rollback| C_retry
    C_terminal -->|Committed or already applied| C_done
    C_terminal -->|Failure: rollback| C_retry
    C_scan -->|No eligible rows| C_done
  end
  subgraph C_STRIPE["Stripe test environment"]
    C_provider["Return session state<br/>or expiry result"]
  end
  C_send -->|Task message| C_queue
  C_queue -->|Worker receives task| C_scan
  C_fetch -->|Provider request outside database locks| C_provider
  C_provider -->|Current session state| C_check
  C_provider -->|Request failure| C_retry
```

| Handoff                   | Implemented contract                                                                                          |
| ------------------------- | ------------------------------------------------------------------------------------------------------------- |
| Beat → Redis → worker     | Task `apps.checkout.tasks.reconcile_stripe_payments`, no arguments, every 60 seconds.                         |
| Worker → recovery command | `call_command("reconcile_stripe", limit=100)`; independently bounds events and attempts.                      |
| Worker → Stripe           | Existing session retrieval/expiry outside database locks; provider uncertainty stays recoverable.             |
| Recovery → persistence    | PostgreSQL retains inbox/attempt state; paid finalization uses phase B's transaction and unique posting keys. |

Local development uses scheduled recovery without webhook delivery; the customer
status check is another reconciliation entry point. Deployment should configure
direct signed HTTPS webhook delivery and retain the periodic scan as recovery.
Only one Beat instance should dispatch this schedule. Worker or broker downtime
can delay execution; database scans resume when services return. A successful
Celery task may still report unresolved operations because the command catches
row-level errors. Refund recovery and settlement are excluded from this task.
See the [implemented payment-recovery contract](../../../../backend/docs/api/checkout/stripe.md#scheduled-payment-recovery).

A periodic worker scans unresolved attempts, including those past their local
deadline. It retrieves/expires the provider session outside locks, then locks and
rechecks local state before changing stock. Provider-paid state routes through B.
Release holds and unfreeze cart lines only after confirming the attempt cannot
still collect payment and has not already succeeded. Unknown/processing results
remain reserved for later periodic scans; persistent failures become visible to staff.
A new purchase after terminal expiry needs a fresh review and request key.

**Implemented inventory rule:** COD placement and preview count every unresolved
held reservation, including past its local deadline. Cart writers freeze selected
lines; product saves reject inventory below held quantities. A local timeout is
not evidence that Stripe can no longer collect payment. Beat schedules `reconcile_stripe`
every 60 seconds for abandoned-payment recovery; see the API guide for bounded batches
and manual investigation when a session creation result remains unknown.

## API handoffs and recovery boundaries

| Operation              | Exact contract                                                                                                              | Outcome                                                                                               |
| ---------------------- | --------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| Review                 | `POST /api/shop/checkout/preview/` with product/cart selection and `payment_method: "stripe"`                               | USD total, blocking issues or signed token; creates no purchase                                       |
| Place/replay           | `POST /api/shop/checkouts/stripe/` with `preview_token`, `recipient_id`, `payment_method: "stripe"`, UUID `idempotency_key` | 200 saved payment state; identical body/key recovers the same purchase                                |
| Read status            | `GET /api/shop/checkouts/{reference}/payment/`                                                                              | Owner-only saved state, no provider call                                                              |
| Reconcile              | `POST /api/shop/checkouts/{reference}/payment/` with empty body                                                             | Owner-only provider reconciliation, no new purchase                                                   |
| Cancel pending payment | `POST /api/shop/checkouts/{reference}/payment/cancel/` with empty body                                                      | Save cancellation intent and release only after confirmed safe unpaid closure                         |
| Provider notification  | `POST /api/shop/stripe/webhook/` with raw event and `Stripe-Signature`                                                      | Signature-authenticated test event; 200 after durable receipt; processing failures remain recoverable |

Missing/invalid customer authentication returns 401; foreign or missing references
return 404. Input failures and the narrowly confirmed session-creation rejection
return 400; stale reviews and key/cart conflicts return 409. Provider uncertainty,
misconfiguration, or mismatches can return 503. After an interrupted placement or
uncertain 503, retain its original body/key and holds. A key conflict must be
resolved, not bypassed with a new key.

A local deadline never authorizes releasing a hold by itself. Payment already
confirmed during cancellation completes the purchase instead. Unknown creation
beyond the service's 23-hour guard requires evidence-based investigation, not a
new provider request or arbitrary stock release. See the backend contract for
all limits and exception rules.

## Implemented customer cancellation and card refunds

The customer Orders feature now supports cancellation of Pending, Accepted and
Packed orders. Paid Stripe test orders refund their entire store-order share to
the original test card, including partial refunds of multi-store checkout charges.
This is separate from the proposed return-to-wallet flow above. No customer wallet
UI, wallet credit, return workflow or automatic seven-day completion is included.

A durable OrderRefund is committed with cancellation and one-time stock restoration
before provider calls. Its stable idempotency key and saved provider ID protect
unknown outcomes. `reconcile_order_refunds` can recover unfinished refunds, but
the current Celery Beat schedule does not dispatch that command. Scheduling it
is a separate operator responsibility; customer status checks use the same service. Verified success posts a linked
compensating journal transaction from store pending to treasury. The original paid
receipt remains immutable. See the [implemented API and recovery contract](../../../../backend/docs/api/ecommerce/user-orders.md)
and [frontend handoff](../../../../frontend/docs/ecommerce/user-orders.md).
