# Demo online checkout and USD wallets

**Status: Stripe test checkout, recovery and pending wallet credits are implemented. Delivery, seven-day settlement, returns and withdrawals remain proposed.**

Implementation contract and setup: [Stripe test checkout API](../../../../backend/docs/checkout/stripe-api.md). The [wallet foundation](../../../../backend/docs/wallet/README.md) records USD cents with zero demo fees.

One Stripe test payment buys from multiple stores. The platform records simulated
USD funds and each store's pending share. Seven days after that store order is
delivered, its eligible share becomes available for mock withdrawal. Stores owned
by the same seller retain separate wallets, including stores in different countries.

This replaces the earlier Stripe Connect proposal. Stripe test transactions do not
move real money ([Stripe testing](https://docs.stripe.com/testing)). The treasury,
wallets and destination bank accounts described here are application simulations;
Stripe Connect accounts and real bank transfers are outside this demo.

## Scope and decisions

| Category | Decision |
| --- | --- |
| Confirmed direction | One payment for multi-store checkout; USD simulated treasury; customer and store wallets; local-currency presentation; seven-day return window starting at delivery; release only when no unresolved return blocks it. |
| Proposed first version | Hosted Stripe Checkout, cards only, charged in USD; zero commission and no simulated processor/FX fees; mock withdrawals only; one settlement per store order. |
| Proposed timing | Seven days means 168 hours in UTC. Requests must be durably accepted before the deadline. Worker delay does not extend the window. |
| Deferred | Return logistics, adjudication, partial refunds, customer wallet spending, real banking, Stripe Connect and live-mode payments. Customer wallet ownership is designed now; refund credits arrive with the return feature. |
| Before implementation | Confirm delivery authority, the timing convention, fee policy, cancellation compensation and withdrawal permissions. These recommendations do not create approved endpoint contracts. |

Source: the **Online Checkout Process** page in the
[general business-process draft](../../general-business-processes/e-commerce-general-business-process.drawio).
The draft's seven-day branch must be guarded by absence of an unresolved return.
Its three-day seller review starts when the store receives the returned goods;
a timely appeal must stop automatic refund while platform adjudication is pending.
Those are future return requirements, not functionality delivered by this proposal.

See the [proposed wallet ERD](../../erd/demo-wallet-ledger.md), the
[current checkout models](../../../../backend/docs/checkout/models.md), and the
[implemented COD flow](cod-checkout.md).

## Money and currency rules

- The ledger uses integer USD cents. Existing checkout Decimal fields can remain,
  but online snapshots must be exactly representable in cents before calling Stripe.
- For each item, multiply its source unit price by the saved source-to-USD rate,
  round once to two decimals using the agreed rounding rule (proposed half-up),
  then multiply by quantity. Sum these stored subtotals into order and checkout totals.
  The total charge must equal the sum of all store allocations; no frontend totals.
- Retain source currency, original price, rate and rate date on purchase snapshots.
  Equal currencies use rate 1. A fresh quote is required if a source rate is missing.
- Show the USD charge explicitly. User/store currency equivalents are labeled
  estimates with rate time; changing display currency never changes USD balances.
- A mock withdrawal snapshots its USD debit, destination currency, conversion rate,
  rate time, rounding and destination amount. The fake bank receipt preserves those
  terms, even if display rates later change. Conversion does not create ledger profit.
- Store proceeds are platform liabilities, not platform revenue. With the proposed
  zero-fee demo, every paid cent is allocated to stores; treasury assets back the
  combined customer/store liabilities.

## A. Review, reserve and start payment

```mermaid
swimlane-beta TB
  subgraph A_UI["Frontend"]
    A_start(["Review all stores and USD total"])
    A_error(["Show error or recover saved attempt"])
    A_pay(["Open hosted test checkout"])
  end
  subgraph A_API["Backend"]
    A_check{"Valid request and stock?"}
    A_save["Commit pending orders and stock holds"]
    A_session["Create session with stable attempt key"]
    A_check -->|Yes| A_save --> A_session
  end
  subgraph A_STRIPE["Stripe test environment"]
    A_create{"Session creation result?"}
  end
  A_start -->|Submit reviewed selection and retry key| A_check
  A_check -->|No: no purchase committed| A_error
  A_session -->|Create or recover session| A_create
  A_create -->|Known session URL| A_pay
  A_create -->|Failure or uncertain: recover attempt| A_error
```

Within a database transaction, authorize customer/recipient/cart ownership,
validate the signed preview and its USD total, lock products in stable order,
and reserve all selected quantities or roll back the entire selection. Persist
one Checkout, one Order per store, item snapshots and one pending PaymentAttempt.
Replaying a request key returns the same attempt; changed contents conflict.

Call Stripe outside database locks. Persist provider IDs and use the attempt's
stable idempotency key to recover from timeouts. Do not create a second payable
session while an earlier result is unknown. The proposed cart policy freezes
selected cart quantities until resolution; unselected lines remain editable.
This requires coordinated backend cart mutation rules and frontend feedback.

## B. Confirm one payment for all stores

```mermaid
swimlane-beta TB
  subgraph B_STRIPE["Stripe test environment"]
    B_event(["Payment event"])
  end
  subgraph B_API["Backend"]
    B_verify{"Signature and event valid?"}
    B_reject(["Reject invalid delivery"])
    B_inbox["Persist event uniquely"]
    B_check{"Verified payment state?"}
    B_wait(["Reconcile pending or mismatched state"])
    B_commit["Commit all orders paid and ledger receipt"]
    B_done(["Saved checkout completed"])
    B_verify -->|No| B_reject
    B_verify -->|Yes| B_inbox --> B_check
    B_check -->|Paid and matching snapshots| B_commit --> B_done
    B_check -->|Pending or mismatch| B_wait
  end
  subgraph B_UI["Frontend"]
    B_show(["Read saved confirmation or show processing"])
  end
  B_event -->|Signed webhook| B_verify
  B_done -->|Authorized status response| B_show
  B_wait -->|Pending status response| B_show
```

Verify the raw-body signature and test environment. Durably deduplicate provider
event IDs; retries of the same payment through different events must also resolve
to one business operation. Acknowledge only after durable receipt and recover
unprocessed events through a periodic database scan. Retrieve provider state where
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
requires reconciliation and compensation, not unchecked stock consumption.
Out-of-order failure events must never regress an already paid checkout.

## C. Recover abandoned or uncertain payments

A periodic worker scans unresolved attempts, including those past their local
deadline. It retrieves/expires the provider session outside locks, then locks and
rechecks local state before changing stock. Provider-paid state routes through B.
Release holds and unfreeze cart lines only after confirming the attempt cannot
still collect payment and has not already succeeded. Unknown/processing results
remain reserved and retry with backoff; persistent failures become visible to staff.
A new purchase after terminal expiry needs a fresh review and request key.

**Implemented inventory rule:** COD placement and preview count every unresolved
held reservation, including past its local deadline. Cart writers freeze selected
lines; product saves reject inventory below held quantities. A local timeout is
not evidence that Stripe can no longer collect payment. Schedule `reconcile_stripe`
every minute for abandoned-payment recovery; see the API guide for bounded retries
and manual investigation when a session creation result remains unknown.

## D. Delivery and seven-day release (future)

```mermaid
swimlane-beta TB
  subgraph D_OPS["Authorized fulfillment actor"]
    D_deliver(["Confirm store order delivered"])
  end
  subgraph D_API["Backend"]
    D_record["Save first delivery and release deadline"]
    D_pending(["Store share remains pending"])
  end
  subgraph D_WORK["Settlement worker"]
    D_scan["Scan due settlements"]
    D_ready{"Paid, delivered, due and no hold?"}
    D_retry(["Leave pending and retry when eligible"])
    D_release["Commit pending to available once"]
    D_done(["Store can request mock withdrawal"])
    D_scan --> D_ready
    D_ready -->|No| D_retry
    D_ready -->|Yes| D_release --> D_done
  end
  D_deliver -->|Authorized delivery command| D_record
  D_record --> D_pending
  D_record -->|Persisted deadline discovered by scan| D_scan
```

Persist `delivered_at` on the first validated delivery transition, plus
`eligible_at = delivered_at + 168 hours`. Repeated notifications must neither
reset this deadline nor increment delivered-unit `sold_count` twice. Existing
`concluded_at` also covers other conclusions, so it must not be treated as proof
of delivery. Proposed first scope assumes the whole store order is delivered
at once; split deliveries require allocation-level deadlines later.

Each store order has its own clock. One store's delayed shipment or return must
not prevent another store's eligible share from becoming available. Settlement
moves a liability from pending to available; treasury assets do not change and
no Stripe transfer occurs. Pending funds cannot be spent or withdrawn.

Store durable deadlines in PostgreSQL. Celery Beat can scan due rows periodically
(proposed every minute), with Celery workers performing short idempotent jobs and
Redis acting as the broker. Lock/recheck settlement and account rows during each
posting. Scans recover after outages and publication failures; overlapping jobs
must be safe. Do not enqueue a task with a seven-day countdown: Celery documents
memory and broker timeout concerns for distant ETA tasks.
[Celery ETA guidance](https://docs.celeryq.dev/en/stable/userguide/calling.html#eta-and-countdown)

## E. Mock withdrawal (future)

```mermaid
swimlane-beta TB
  subgraph E_UI["Wallet owner"]
    E_start(["Request withdrawal"])
    E_error(["Show validation error"])
    E_done(["Show saved receipt or failure"])
  end
  subgraph E_API["Backend"]
    E_check{"Authorized and sufficient available funds?"}
    E_reserve["Commit withdrawal and reserve balance"]
    E_finish["Commit success or restore reservation"]
    E_wait(["Keep reserved and reconcile"])
    E_check -->|Yes| E_reserve
  end
  subgraph E_BANK["Mock bank adapter"]
    E_result{"Transfer result for withdrawal key?"}
  end
  E_start -->|Amount, owned destination and retry key| E_check
  E_check -->|No| E_error
  E_reserve -->|Idempotent simulated transfer| E_result
  E_result -->|Success or definitive failure| E_finish
  E_result -->|Unknown| E_wait
  E_finish -->|Persisted withdrawal status| E_done
```

Store withdrawal permission must be an explicit server-side role check, proposed
store administrators only. Customer withdrawals require ownership. Protect against
concurrent requests by locking accounts and reserving the amount before calling
the adapter. Amounts must be positive cents and within available funds; destinations
must belong to that wallet. Do not collect real bank credentials.

The mock adapter records one durable receipt per withdrawal reference. A timeout
must recover that receipt or retry the same key. Success reduces the simulated treasury asset exactly once; definite
failure restores available funds exactly once. An uncertain result stays reserved
until reconciled. Never generate a fresh transfer key on retry.

## Future returns and customer credits

An accepted return request must atomically lock the same settlement used by the
release worker and create a hold before the deadline. Request acceptance uses the
server's transactional acceptance time; simply opening a form does not reserve
eligibility. Release at or after the deadline must respect every already accepted
hold, even if review finishes weeks later. Return submission closes at the deadline.

Initially, a future return can conservatively hold the entire store settlement.
Partial adjudication can credit the approved USD amount to the customer wallet,
then release the remaining store amount after all holds resolve and the original
deadline passes. Persist item/quantity allocations and cumulative refund limits
when implementing returns. A rejected return removes its hold; it must not start
a new seven-day window. The draft's three-day review and appeal need their own
durable deadlines and mutually exclusive resolution operations.

Internal customer credit reallocates a liability from store pending to customer
available, leaving treasury unchanged. It does not also issue a Stripe refund.
Any future original-payment refund needs a separate coordinated accounting path
to prevent double reimbursement. Post-release disputes need an explicit recovery
policy; never silently debit a store below zero. Return effects on historical
`sold_count` remain a product decision.

## Implementation sequence and acceptance cases

1. Review this proposal and its ERD; settle the proposed defaults above.
2. **Implemented:** USD ledger, wallet ownership, posting invariants and read-only admin.
3. **Implemented:** Stripe test checkout, holds, webhook inbox, recovery command and frontend payment status. Actual Stripe-account setup is required to run it.
4. Add authorized delivery, idempotent sold counts and periodic settlement.
5. Add mock withdrawals and wallet views; implement returns as a later feature.

Required future tests: all-store rollback on stock failure; duplicate/concurrent
webhooks; provider success with local transaction failure; lost session response;
expiry racing payment; stale holds respected by COD; no release before delivery or
deadline; independently delivered stores; release racing a timely return hold;
repeated delivery/settlement; concurrent withdrawals; adapter timeout after success;
FX rounding and exact charge/allocation/ledger equality. UI must label simulated
balances, estimated conversions, pending funds and processing payment states.

## Validation

Online payment routes, a migration, provider adapter, webhook inbox, recovery
command, pending wallet credits and frontend payment flow now exist. Settlement
jobs and wallet balance/withdrawal screens remain future work. See the API guide
for current setup and verification; the diagrams below the payment phases remain
design proposals, not implemented delivery/settlement behavior.
All four swimlanes and both linked ERD diagrams parsed and rendered with Mermaid
12.0.0 in Chromium on 2026-10-09 and were visually inspected. Relative links were
checked. The target Markdown host was not tested for `swimlane-beta` support.


## Implemented customer cancellation and card refunds

The customer Orders feature now supports cancellation of Pending, Accepted and
Packed orders. Paid Stripe test orders refund their entire store-order share to
the original test card, including partial refunds of multi-store checkout charges.
This is separate from the proposed return-to-wallet flow above. No customer wallet
UI, wallet credit, return workflow or automatic seven-day completion is included.

A durable OrderRefund is committed with cancellation and one-time stock restoration
before provider calls. Its stable idempotency key, saved provider ID and scheduled
reconciliation command protect unknown outcomes. Verified success posts a linked
compensating journal transaction from store pending to treasury. The original paid
receipt remains immutable. See the [implemented API and recovery contract](../../../../backend/docs/ecommerce/user-orders.md)
and [frontend handoff](../../../../frontend/docs/ecommerce/user-orders.md).
