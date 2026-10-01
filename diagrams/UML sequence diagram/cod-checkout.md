# Cash on Delivery checkout

**Status:** Approved API and workflow design; checkout endpoints and frontend
integration are not yet implemented. This diagram describes the intended behavior,
not a claim about the current application.

This sequence covers authenticated checkout from a product or selected cart items,
recipient selection, price review, atomic COD placement, and confirmation.

```mermaid
sequenceDiagram
    autonumber
    actor Customer
    participant UI as Frontend
    participant API as Backend API
    participant DB as Database
    participant FX as Exchange-rate service

    Customer->>UI: Checkout selected cart items or one product
    UI->>API: GET /api/account/recipients/
    API->>DB: Read customer's saved recipients
    DB-->>API: Recipient list
    API-->>UI: Recipients
    UI->>UI: Preselect default recipient, if available

    UI->>API: POST /api/shop/checkout/preview/
    API->>DB: Read owned cart items or product, prices and stock
    DB-->>API: Current purchase details
    API->>API: Resolve customer currency, fall back to USD if unsupported

    opt Currency conversion required
        API->>FX: Get valid rates, using cache when available
        FX-->>API: Exchange rates or unavailable
    end

    API->>API: Calculate item, store and checkout totals

    alt Preview is valid
        API-->>UI: Store groups, totals and signed token valid for 5 minutes
        UI-->>Customer: Show checkout review with COD selected
    else Items invalid or conversion unavailable
        API-->>UI: Item errors or conversion error
        UI-->>Customer: Explain issue and prevent placement
    end

    opt Customer needs a new recipient
        Customer->>UI: Enter recipient details
        UI->>API: POST /api/account/recipients/
        API->>DB: Validate and save recipient for customer
        API-->>UI: Saved recipient
        UI->>UI: Select newly added recipient
    end

    Customer->>UI: Select recipient and click Place Order
    UI->>UI: Disable repeat clicks, retain request idempotency key
    UI->>API: POST /api/shop/checkouts/<br/>preview_token, recipient_id, cod, idempotency_key

    API->>DB: Check previous submission for this customer and key

    alt Same request already succeeded
        DB-->>API: Existing checkout
        API-->>UI: 200 — existing confirmation
    else Key reused with different input
        API-->>UI: 409 — idempotency conflict
    else New submission
        API->>API: Verify preview signature, expiry and recipient ownership
        API->>DB: Begin transaction, lock relevant rows
        API->>DB: Recheck selected cart quantities, prices and available stock

        alt Preview expired or purchase details changed
            API->>DB: Roll back any transaction work
            API-->>UI: 409 — review required
            UI-->>Customer: Show issue and request a fresh preview
        else All checks pass
            API->>DB: Create one Checkout and one Order per store
            API->>DB: Save recipient, product and price snapshots
            API->>DB: Deduct purchased stock
            opt Checkout came from cart
                API->>DB: Remove purchased quantities from selected cart items
            end
            API->>DB: Mark checkout completed, store orders pending and unpaid
            API->>DB: Commit transaction
            API-->>UI: 201 — confirmation with checkout reference
        end
    end

    opt Placement succeeded or a successful request was replayed
        UI->>UI: Open confirmation route using checkout reference
        UI->>API: GET /api/shop/checkouts/{reference}/
        API->>DB: Read customer's saved checkout and order snapshots
        API-->>UI: Confirmation details
        UI-->>Customer: Order placed — payment due on delivery
    end
```

## Flow rules

- Every endpoint requires authentication. Only the owning customer can read their
  recipients, selected cart lines and checkout confirmation.
- Placement requires a valid preview and exactly one complete recipient. Errors
  prevent progression until resolved; the customer must review a fresh preview
  after expiry or selected cart quantity/price changes.
- Preview requests use either product ID and quantity or selected cart-item IDs.
  The backend reads current cart quantities. Preview never reserves stock.
- Customer currency defaults to USD when it cannot be resolved or is unsupported.
  Missing exchange rates do not change the meaning of original store prices;
  required conversion failure blocks checkout.
- The signed preview expires after five minutes. Quantity changes happen on the
  product or cart page before requesting another preview.
- Idempotency lookup precedes preview/cart revalidation. A successful retry returns
  the original checkout even when its preview has expired or its cart items have
  already been removed. Reusing the key for different input is a conflict.
- Invalid signatures/fields return 400, missing or inaccessible resources 404,
  expired previews or changed purchase details 409, and unavailable conversion 503.
  Validation failures before a transaction starts do not require a rollback.
- Final creation, stock deduction and cart cleanup commit together. Concurrent
  submissions must be serialized or handled through database uniqueness so an
  early idempotency lookup cannot allow duplicate orders.
- Available stock must account for active holds once Stripe reservations exist.
  COD does not create temporary reservations or Stripe payment attempts.
- A network retry uses the same idempotency key. Confirmation reads saved purchase
  snapshots rather than recalculating from current products or exchange rates.

## State after successful placement

| Record | Result |
| --- | --- |
| Checkout | `completed`, payment method `cod` |
| Each store order | Fulfillment `pending`, payment `unpaid` |
| Product stock | Reduced by the purchased quantity |
| Selected cart items | Purchased quantities removed; unselected items retained |
| Direct product checkout | Cart unchanged |
| Stripe payment attempts | None |
| COD stock reservations | None |

Checkout completion does not mean payment collection or delivery. Payment becomes
`paid` only when collection is confirmed in a later fulfillment workflow.

See the [diagram conventions and transition](../README.md).
