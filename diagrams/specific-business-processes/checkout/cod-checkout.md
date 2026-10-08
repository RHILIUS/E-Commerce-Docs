# Cash on Delivery checkout

**Status:** Implemented COD workflow. Saved-recipient listing/creation, checkout
preview, atomic COD placement, owner-only confirmation APIs, and frontend checkout
and confirmation pages are implemented. See the
[frontend handoff](../../../../frontend/docs/checkout/README.md) for UI behavior,
recovery and verification. The request-fingerprint migration exists in the backend
repository and must be applied to the target database before placement is used.
Stripe, delivery and payment collection remain outside the implemented scope.

## Scope and notation

Authenticated customers buy one product or selected cart items, choose one
recipient, review store totals, place an atomic COD purchase, and view confirmation.
Follow phases A → B → C → D. A blocked branch cannot advance to placement.
API routes and payloads are listed below to keep decision nodes readable.

Preview implementation and response details: [backend API contract](../../../../backend/docs/checkout/preview-api.md).

## A. Enter checkout and review prices

```mermaid
swimlane-beta TB
    subgraph A_UI["Frontend"]
        A_start(["Customer starts checkout"])
        A_ready{"Valid product quantity or<br/>at least one selected cart item?"}
        A_disabled["Disable Checkout until corrected"]
        A_send["Request preview for product<br/>or selected cart IDs"]
        A_error["Show issue and block placement"]
        A_fix(["Correct source items or retry<br/>then request a new preview"])
        A_review(["Show store groups and totals<br/>COD selected → phase B"])
        A_start --> A_ready
        A_ready -->|No| A_disabled
        A_ready -->|Yes| A_send
        A_error --> A_fix
    end
    subgraph A_API["Backend"]
        A_read["Authenticate and read owned items<br/>Current quantities, prices and stock"]
        A_valid{"Items and quantities valid?"}
        A_currency["Resolve customer currency<br/>Use USD if unresolved or unsupported"]
        A_rates{"Required conversion rates available<br/>from valid cache or FX service?"}
        A_token["Calculate item, store and grand totals<br/>Issue signed preview valid for 5 minutes"]
        A_read --> A_valid
        A_valid -->|Yes| A_currency --> A_rates
        A_rates -->|Yes or no conversion needed| A_token
    end
    A_send --> A_read
    A_valid -->|No: item or request issues| A_error
    A_rates -->|No: 503| A_error
    A_token -->|Valid preview| A_review
```

Preview does not reserve stock or create orders. Only a valid preview with
`can_place_order` true can advance. Display item issues otherwise. Product checkout
carries the product ID and quantity. Cart checkout carries selected cart IDs and
uses their saved quantities. Product details and prices are refreshed from the
backend. A changed price must be shown for review before placement.

Quantity changes occur on the product/cart page. If a cart quantity exceeds stock,
show an error and reset it to 1 only when stock is positive. At zero stock, mark the
item unavailable and block purchase until removed or available again.

Use stored `Store.currency` as the source currency. USD fallback selects the
customer's target currency only: it must never relabel an unknown source price.
Missing required rates block checkout. The backend owns all amount calculations.

## B. Select or add a recipient

```mermaid
swimlane-beta TB
    subgraph B_UI["Frontend"]
        B_start["From A: request saved recipients"]
        B_list["Preselect default if available<br/>Change opens a single-selection list"]
        B_action{"Customer action?"}
        B_form["Enter name, phone and complete address"]
        B_error["Show field errors and keep form open"]
        B_select["Select exactly one complete recipient"]
        B_next(["Review recipient, items and COD total<br/>Place Order → phase C"])
        B_list --> B_action
        B_action -->|Add new| B_form
        B_action -->|Choose saved| B_select
        B_error --> B_form
        B_select --> B_next
    end
    subgraph B_API["Backend"]
        B_get["GET own recipients"]
        B_validate{"New recipient fields valid?"}
        B_save["Save for authenticated customer<br/>Update defaults atomically"]
        B_validate -->|Yes| B_save
    end
    B_start --> B_get
    B_get -->|Recipient list| B_list
    B_form -->|POST recipient| B_validate
    B_validate -->|No: 400| B_error
    B_save -->|201: select created recipient| B_select
```

The recipient list response is required before preselection. An empty list requires
adding a recipient. Failed requests show an error with retry and do not advance.
The first saved recipient becomes default. Explicitly creating a new default
clears the previous default atomically. Selecting a recipient for this purchase
alone does not change the saved default. Placement remains disabled while the
recipient is missing/incomplete or preview invalid/expired.

## C. Place the purchase exactly once

[Placement and confirmation contract](../../../../backend/docs/checkout/cod-api.md).

```mermaid
swimlane-beta TB
    subgraph C_UI["Frontend"]
        C_send["Place Order: retain payload and UUID key<br/>Disable repeated clicks"]
        C_retry["Response unknown: retry identical<br/>payload with the same key"]
        C_error(["Show error and resolve it<br/>Fresh reviewed purchase uses a new key"])
        C_done(["201 created or 200 replay<br/>Open confirmation → phase D"])
        C_send -.->|Network outcome unknown| C_retry
        C_retry --> C_send
    end
    subgraph C_API["Backend"]
        C_auth["Authenticate and validate request shape"]
        C_key{"Successful request for this<br/>customer and idempotency key?"}
        C_same{"Same payload?"}
        C_check{"Signature, expiry and<br/>owned complete recipient valid?"}
        C_begin["Begin transaction and serialize submissions<br/>Recheck key under lock before mutations"]
        C_stock{"Locked cart quantities, prices<br/>and available stock still valid?"}
        C_write["Create Checkout and one Order per store<br/>Save item, recipient and price snapshots<br/>Deduct stock and clean selected cart quantities"]
        C_commit["Commit all changes together<br/>Checkout completed, orders pending and unpaid"]
        C_rollback["Roll back all transaction work"]
        C_auth --> C_key
        C_key -->|Yes| C_same
        C_key -->|No| C_check
        C_check -->|Yes| C_begin
        C_begin -->|New key| C_stock
        C_begin -->|Concurrent success found| C_same
        C_stock -->|Yes| C_write
        C_stock -->|No| C_rollback
        C_write -->|Success| C_commit
        C_write -->|Failure| C_rollback
    end
    C_send -->|POST placement| C_auth
    C_same -->|Yes: 200 existing checkout| C_done
    C_same -->|No: 409 conflict| C_error
    C_check -->|No: validation or expiry error| C_error
    C_rollback -->|Failure: no partial purchase| C_error
    C_commit -->|201 with reference| C_done
```

Successful replay lookup precedes preview expiry/cart revalidation: a retry must
return the original checkout even after cart cleanup or token expiry. Database
serialization/uniqueness must prevent simultaneous duplicate placement. Any
transaction error, including commit failure, leaves no partial purchase. If the
commit outcome is uncertain to the client, retry the same request/key to discover
its result rather than create another purchase.

Lock relevant customer, cart and product rows and revalidate inside the transaction.
Available stock must account for active holds once Stripe reservations exist.
Price, cart quantity, stock or expiry conflicts require resolving the issue and
reviewing a fresh preview before a new placement. An idempotency payload conflict
requires resolving the conflicting submission, not blindly retrying a new key.

COD creates neither Stripe payment attempts nor temporary stock reservations.
Only cart-origin purchases remove purchased quantities from selected cart lines.
Unselected lines remain, and direct product checkout leaves the cart unchanged.

## D. Read confirmation

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

## API handoffs

All endpoints require authentication. Missing or foreign resources return 404;
unauthenticated requests return 401 and stop the flow until authentication succeeds.
These common guards apply to every diagram, including the request-shape guard in C.

| Operation            | Endpoint                               | Request / result                                                                                                    |
| -------------------- | -------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| List recipients      | `GET /api/account/recipients/`         | Own saved recipients, default first                                                                                 |
| Add recipient        | `POST /api/account/recipients/`        | Name, E.164 phone, country/address fields, optional default flag → 201 recipient                                    |
| Preview product      | `POST /api/shop/checkout/preview/`     | `source: "product"`, `product_id`, `quantity`                                                                       |
| Preview cart         | Same preview route                     | `source: "cart"`, `item_ids` containing selected cart-line IDs                                                      |
| Valid preview result | Preview response                       | Store groups, decimal-string amounts and currency, issues, `can_place_order`, signed `preview_token` and expiry     |
| Place COD            | `POST /api/shop/checkouts/`            | `preview_token`, `recipient_id`, `payment_method: "cod"`, UUID `idempotency_key` → 201 new or 200 successful replay |
| Confirmation         | `GET /api/shop/checkouts/{reference}/` | Owner-only saved checkout/order snapshots                                                                           |

## Failure and recovery rules

| Condition                                                    | Result                        | Customer/frontend recovery                                  |
| ------------------------------------------------------------ | ----------------------------- | ----------------------------------------------------------- |
| Malformed fields or invalid signature                        | 400                           | Correct input and obtain a valid preview as needed          |
| Authentication missing/expired                               | 401                           | Authenticate before retrying                                |
| Missing/foreign recipient, cart item or checkout             | 404                           | Refresh selection or show unavailable confirmation          |
| Invalid preview items                                        | Issues, placement blocked     | Resolve on source page and preview again                    |
| Preview expired or stock/price/cart changed during placement | 409                           | Resolve issue, request fresh preview, review again          |
| Key reused with different payload                            | 409                           | Resolve conflicting submission, never silently duplicate it |
| Required conversion unavailable                              | 503                           | Keep placement blocked and retry preview later              |
| Transaction failure                                          | No partial persisted purchase | Show failure, retry according to known/unknown outcome      |
| Network outcome unknown                                      | Placement may have committed  | Retry identical request/key                                 |

## Persisted result

| Record                            | State after successful COD placement                           |
| --------------------------------- | -------------------------------------------------------------- |
| Checkout                          | `completed`, method `cod`                                      |
| One order per store               | Fulfillment `pending`, payment `unpaid`                        |
| Order items                       | Purchased quantities and original/converted price/FX snapshots |
| Recipient and store details       | Snapshots preserve the purchase record                         |
| Stock                             | Purchased quantities deducted atomically                       |
| Cart                              | Purchased selected quantities removed only for cart checkout   |
| PaymentAttempt / StockReservation | No COD records                                                 |

Checkout completion is not payment collection or delivery. Marking payment `paid`
belongs to a later collection workflow. Stripe, delivery, shipping fees, taxes and
discounts are outside this COD flow; the agreed total is the sum of store subtotals.

See the [diagram index](../../README.md) and
[diagramming specification](../../diagramming-specification.md).
