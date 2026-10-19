# Proposed settlement, withdrawals, and returns

**Status: planning only.** These notes preserve the future direction from the
original online-checkout design. They are not implemented API contracts or
permission to add those workflows. Current payments stop at pending store credits;
current customer cancellation refunds return a paid store-order share to the
original Stripe test card, not a customer wallet.

A **settlement** would make an eligible store's pending share available. A
**withdrawal** would move available demo value to a simulated destination. A
**return** would handle sending goods back and deciding a reimbursement. They are
different operations and need explicit ownership, authorization, and recovery.

## Recorded direction and unresolved decisions

The design direction is a USD simulated treasury, separate store wallets even for
stores sharing sellers, and a seven-day window starting with delivery. A timely
unresolved return would block release. The proposed timing is 168 hours in UTC;
worker delay would not extend submission eligibility.

Still resolve delivery authority, return acceptance and appeals, withdrawal
permissions and destination ownership, split deliveries, accounting/fee policies,
and recovery after release before implementing them. Earlier cancellation questions
must account for the already implemented original-card cancellation-refund path.
These notes do not establish a customer-wallet refund policy or a settlement ERD.

Source context: the **Online Checkout Process** page in the
[editable general business-process reference](../drawio/general-business-processes/e-commerce-general-business-process.drawio).
Its three-day seller-review idea starts when returned goods arrive; an appeal
would need a durable hold while adjudication is pending. These are future design
constraints, not current customer behavior.

## Proposed money and conversion rules

- Keep ledger values in integer USD cents and store pending/available liabilities
  distinct from the simulated treasury asset.
- A mock withdrawal would snapshot its USD debit, destination currency, exchange
  rate/date, rounding, and resulting amount. A retry must preserve those terms.
- Changing display currency must not change USD balances or create accounting profit.
- The demo direction uses zero fees; adding fees needs an explicit allocation policy.

The diagrams below are activity-style Mermaid swimlane flowcharts of proposals,
not implemented services. Lane names identify intended owners, not existing APIs.

## Proposed delivery and seven-day release

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

## Proposed mock withdrawal

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

## Next implementation decisions

Establish authorized delivery and its history first, then design settlement
records, holds, durable deadlines, and release. Define withdrawal and return
contracts separately, including races with release and uncertain provider results.
Required acceptance cases include release before/after deadlines, independently
delivered stores, repeated delivery, release racing a timely return hold, concurrent
withdrawals, and unknown transfer outcomes. None of these cases implies that the
corresponding service already exists. Any future wallet interface must label
balances as simulated, display conversions as estimates, distinguish pending
funds, and show uncertain payment/transfer outcomes as processing.

Read [implemented online checkout](../diagrams/specific-business-processes/checkout/demo-online-checkout.md),
[the wallet ledger](../../backend/docs/architecture/wallet-ledger.md), and
[the diagramming specification](../diagrams/diagramming-specification.md) before
turning this proposal into code or revising its diagrams.
