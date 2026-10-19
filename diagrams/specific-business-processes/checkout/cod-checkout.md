# Cash on Delivery checkout

Cash on delivery places the purchase now and leaves payment unpaid. A customer
reviews current prices, chooses a delivery recipient, and submits one request.
The backend saves one order per store, deducts stock, and cleans selected cart
lines together. If the response is lost, the same request/key recovers that purchase.

An **idempotency key** identifies that original request. A **snapshot** preserves
the names, delivery details, and prices accepted then, even if those live records
later change. Checkout completed means placement, not payment or delivery.

**Status:** Implemented COD workflow. Saved-recipient listing/creation/editing, checkout
preview, atomic COD placement, owner-only confirmation APIs, and frontend checkout
and confirmation pages are implemented. See the
[frontend handoff](../../../../frontend/docs/checkout/README.md) for UI behavior,
recovery and verification. The request-fingerprint migration exists in the backend
repository and must be applied to the target database before placement is used.
Stripe test payments are implemented in the [separate online flow](demo-online-checkout.md).
Delivery and COD payment collection remain outside this flow.

## Scope and notation

Authenticated customers buy one product or selected cart items, choose one
recipient, review store totals, place an atomic COD purchase, and view confirmation.
Follow phases A → B → C → D. A blocked branch cannot advance to placement.
API routes and payloads are listed below to keep decision nodes readable.

Preview implementation and response details: [backend API contract](../../../../backend/docs/api/checkout/preview.md).

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
        B_edit["Prefill saved recipient<br/>Save changes or cancel to list"]
        B_edit_error["Show validation error<br/>Keep edit draft"]
        B_action -->|Edit saved| B_edit
        B_edit -->|Cancel| B_list
        B_action -->|Add new| B_form
        B_action -->|Choose saved| B_select
        B_error --> B_form
        B_select --> B_next
    end
    subgraph B_API["Backend"]
        B_update["PATCH owned recipient<br/>Validate and update defaults atomically"]
        B_get["GET own recipients"]
        B_validate{"New recipient fields valid?"}
        B_save["Save for authenticated customer<br/>Update defaults atomically"]
        B_validate -->|Yes| B_save
    end
    B_edit -->|PATCH recipient| B_update
    B_update -->|200: preserve checkout selection| B_list
    B_update -->|400| B_edit_error
    B_edit_error --> B_edit
    B_start --> B_get
    B_get -->|Recipient list| B_list
    B_form -->|POST recipient| B_validate
    B_validate -->|No: 400| B_error
    B_save -->|201: select created recipient| B_select
```

The recipient list response is required before preselection. An empty list requires
adding a recipient. Failed requests show an error with retry and do not advance.
The first saved recipient becomes default. Explicitly creating a new default
clears the previous default atomically.

Editing a saved recipient uses `PATCH /api/account/recipients/{id}/`, preserves
checkout selection, and affects future purchases only. Existing order snapshots
remain unchanged. The current default stays default until another replaces it.

Selecting a recipient for this purchase
alone does not change the saved default. Placement remains disabled while the
recipient is missing/incomplete or preview invalid/expired.

## C. Place the purchase exactly once

[Placement and confirmation contract](../../../../backend/docs/api/checkout/cash-on-delivery.md).

### C1. Serialize submissions and recover an existing purchase

```mermaid
swimlane-beta TB
    subgraph C1_UI["Frontend"]
        C1_send(["Submit exact placement body and UUID key<br/>Block duplicate clicks"])
        C1_retry(["Outcome unknown<br/>Offer retry of original body and key"])
        C1_error(["Show input or key-conflict error"])
        C1_done(["Open confirmation in phase D"])
        C1_send -.->|Response lost in C1 or C2| C1_retry
        C1_retry -->|Customer retries| C1_send
    end
    subgraph C1_API["Backend"]
        C1_auth{"Authentication and request shape valid?"}
        C1_begin["Begin transaction and lock customer"]
        C1_key{"Customer already used this key?"}
        C1_same{"Matching completed COD purchase?"}
        C1_replay["End transaction and return original purchase"]
        C1_rollback["Roll back transaction"]
        C1_next(["Continue C2 within this transaction<br/>Customer lock remains held"])
        C1_auth -->|Yes| C1_begin --> C1_key
        C1_key -->|Yes| C1_same
        C1_key -->|No| C1_next
        C1_same -->|Yes| C1_replay
        C1_same -->|No: key conflict| C1_rollback
    end
    C1_send -->|POST placement| C1_auth
    C1_auth -->|No: 400 or 401| C1_error
    C1_rollback -->|409 conflict| C1_error
    C1_replay -->|200 original purchase| C1_done
```

The continuation to C2 is not a commit. Existing-key lookup happens while the
customer lock is held and before validating preview expiry or cleaned-up cart lines.

### C2. Validate and commit a new purchase

```mermaid
swimlane-beta TB
    subgraph C2_API["Backend"]
        C2_start(["From C1: new key and locked customer<br/>Same open transaction"])
        C2_check{"Signed preview and owned recipient valid?"}
        C2_lock["Lock products, stores and selected cart lines<br/>Recheck review expiry and accepted terms"]
        C2_stock{"Current terms and available stock valid?"}
        C2_write["Create checkout, store orders and snapshots<br/>Deduct stock and clean purchased cart lines"]
        C2_commit["Commit all local changes together"]
        C2_rollback["Roll back transaction<br/>No partial purchase"]
        C2_start --> C2_check
        C2_check -->|Yes| C2_lock --> C2_stock
        C2_check -->|No| C2_rollback
        C2_stock -->|No| C2_rollback
        C2_stock -->|Yes| C2_write
        C2_write -->|Write succeeded| C2_commit
        C2_write -->|Failure| C2_rollback
        C2_commit -->|Commit failed| C2_rollback
    end
    subgraph C2_UI["Frontend"]
        C2_error(["Show error and review again when required<br/>Resolve key conflicts before a new request"])
        C2_done(["Open saved confirmation in phase D"])
    end
    C2_rollback -->|Validation, conflict or server error| C2_error
    C2_commit -->|201 completed checkout and unpaid orders| C2_done
```

A response lost after commit returns to C1 with the original body/key. An
uncertain response does not authorize the frontend to create another purchase.

Successful replay lookup precedes preview expiry/cart revalidation: a retry must
return the original checkout even after cart cleanup or token expiry. Database
serialization/uniqueness must prevent simultaneous duplicate placement. Any
transaction error, including commit failure, leaves no partial purchase. If the
commit outcome is uncertain to the client, retry the same request/key to discover
its result rather than create another purchase.

Lock relevant customer, cart and product rows and revalidate inside the transaction.
Available stock subtracts every unresolved Stripe hold, even after its local
deadline. Selected cart lines held by pending payment cannot be purchased again.
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
| Edit recipient       | `PATCH /api/account/recipients/{id}/`  | Partial contact/address/default changes → 200 saved recipient; owner-only                                           |
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
