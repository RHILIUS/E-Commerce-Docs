# How the E-Commerce system works

The system lets customers browse products from several stores, save delivery
addresses, build carts, and place orders. Cash on delivery (COD) places an unpaid
order; Stripe test checkout demonstrates card payment without moving real money.

## The main parts

| Part              | Responsibility                                                                                     |
| ----------------- | -------------------------------------------------------------------------------------------------- |
| Frontend          | Runs in the customer's browser, shows screens, collects input, and displays results                |
| Backend API       | Identifies the customer, checks permissions and business rules, and coordinates changes            |
| PostgreSQL        | Stores accounts, products, purchases, stock holds, and accounting records                          |
| Redis             | Carries queued background tasks; it is not the source of purchase truth                            |
| Celery workers    | Deliver verification email and reconcile pending Stripe payments                                   |
| Celery Beat       | Schedules payment recovery every 60 seconds                                                        |
| External services | Provide configured email transport, exchange rates, Stripe test payment, or optional image storage |

An **API** is the set of request URLs the frontend calls. A **worker** is a process
that performs background work. A **broker** carries tasks to workers; Redis fills
that role. A response saying an email was queued does not prove delivery.

## A customer's journey

1. **Browse.** Guests can search active products and view stores. Archived product
   pages can remain readable, while draft visibility is restricted. Display
   prices help browsing; they are not an accepted checkout quote.
2. **Create an account or sign in.** Registration verifies an emailed one-time
   code before account creation. Login supplies an access token and a refresh
   cookie. The backend checks credentials and ownership even when the frontend
   already hides protected screens.
3. **Maintain account details.** A personal address describes the account holder.
   A delivery recipient describes who receives an order. Registration initially
   creates both; later edits are independent. Photos and password changes have
   separate workflows.
4. **Choose products.** Saved carts are grouped by store. Adding to a cart does
   not reserve stock. Customers may also start checkout directly from a product.
5. **Review checkout.** The backend checks quantities, availability, and prices,
   then returns signed review terms. A preview creates no order and holds no stock.
6. **Place the purchase.** The backend rechecks those terms and creates one order
   per store. Payment method determines when stock is deducted and whether funds
   are recorded in the demo ledger.
7. **Read orders and recover outcomes.** Customers can revisit confirmation,
   payment status, cancellation/refunds where eligible, and recorded history.
   A page reload or failed response must not silently create another purchase.

## One purchase can contain several store orders

A **checkout** groups the customer's purchase. An **order** is one store's part of
it. Buying a camera from one store and a case from another creates one checkout
and two orders, with separate fulfillment and cancellation state.

Orders keep **snapshots**: copies of the recipient details, names, quantities,
prices, and rates accepted at purchase time. Later edits do not rewrite those
copies. Receipt images use current catalog records and can change independently.
Checkout completed means COD placement or verified online payment, not delivery.

## Cash on delivery and online test payment

| Behavior           | COD                                                | Stripe test checkout                                         |
| ------------------ | -------------------------------------------------- | ------------------------------------------------------------ |
| Placement          | Saves a completed checkout and unpaid store orders | Saves pending orders, a payment attempt, and stock holds     |
| Physical stock     | Deducted when placement commits                    | Deducted after verified payment                              |
| Accounting         | No wallet entries                                  | One simulated USD receipt credits each store's pending share |
| Uncertain response | Retry the same placement body and key              | Recover the same saved attempt and reconcile provider state  |

A **transaction** groups local database changes so they succeed together or roll
back together. A **row lock** coordinates competing changes, such as two buyers
requesting the last unit. An **idempotency key** identifies an original request
so repeating it can return the same purchase rather than create another one.

A stock hold protects an unresolved online purchase. Its deadline alone does not
prove payment failed, so checkout continues to count it until safe resolution.
A **webhook** is a signed notification sent by Stripe. **Reconciliation** compares
current provider facts with saved state. Returning from Stripe in the browser is
not proof of payment.

Stripe and PostgreSQL do not share a transaction. A provider can accept payment
before local finalization succeeds; the system retains recovery rather than
charging again. Confirmed customer cancellation refunds add compensating ledger
entries and preserve the original receipt. Refund recovery has a management
command, but is not scheduled by the current Beat configuration.

## What is implemented, and what is planned

Implemented source includes customer browsing, registration/login, account and
photo management, password changes, recipients, carts, COD, Stripe test payments,
payment recovery, eligible order cancellation/refunds, and persistent order
history. The production hosting configuration is implemented; real hostnames,
credentials, service health, and provider setup depend on the target deployment.

Seller delivery operations, automatic seven-day completion, settlement release,
returns, withdrawals, customer wallet spending, live payments, and real bank
transfers are not implemented workflows. Models or enum values naming a state do
not establish an available business operation. Read [planning](planning/README.md)
for the separate future direction.

## Continue reading

- [Production architecture](architecture/production-deployment.md)
- [Business processes and data models](diagrams/README.md)
- [Frontend guides](../frontend/docs/README.md)
- [Backend guides](../backend/docs/README.md)
