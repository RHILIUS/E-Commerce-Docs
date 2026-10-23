# Landing page

The storefront landing page lets visitors browse and search products before
opening a product page. **Status: Catalog behavior is implemented; the diagram is
an older overview rather than a complete description of current behavior.** The
[Draw.io file](../../diagrams/ecommerce/landing-page.drawio.svg) covers loading
products, preparing prices and displaying the result.

## Follow the flow

1. Open the storefront. The frontend requests the first catalog page after the
   initial authentication check finishes. Visitors can browse without signing in.
2. The backend selects and ranks visible products, prepares display prices and
   returns a page of results. The browser renders product cards.
3. Search updates the `q` query parameter in the page URL and starts from the
   first page of that search. Obsolete requests are cancelled so they cannot
   replace newer results.
4. Continue through the catalog to load subsequent pages. The frontend appends
   results while avoiding duplicate cards if rankings change between requests.

## Responsibilities and recovery

The frontend owns input, loading indicators, cards and retry controls. The backend
owns catalog visibility, ranking and price conversion. A visitor's display
currency comes from their personal-address country when available, with USD as
the default. The frontend formats the backend's amount and currency; it does not
calculate exchange rates.

Catalog ranking and displayed ordered quantities count units in all non-cancelled
orders, including unpaid and refunded purchases. Cancelled orders never count,
even when refunded. These figures measure customer demand, not completed sales.
Personal category preferences use lifetime viewer quantities; recent popularity
uses everyone's units over seven days by default. Updates affect the next request.
The diagram's general ranking step remains accurate; the exact rules live in the
[backend catalog contract](../../../../backend/docs/api/ecommerce/product-catalog.md#which-products-appear-first).

Frankfurter supplies exchange rates, rather than converted product prices.
The backend performs decimal conversion. When conversion is unavailable, it keeps
the original store amount and currency. Display conversion is not a guarantee
about checkout currency.

An initial load failure shows Try again. A later-page failure keeps existing
products visible and offers another continuation attempt. An empty search shows
No products found and offers Clear search. These recovery paths are absent from
the older diagram.

## Diagram limitations

Despite its `.drawio.svg` extension, this file contains raw Draw.io XML rather
than a rendered SVG with an embedded model. Open it in Draw.io; it cannot currently
be previewed as a browser image. Its existing contents are preserved.

The drawing simplifies ranking to user preferences and shows the external service
returning converted prices. Current behavior and pagination are explained in the
maintained catalog guide. The diagram's End means products have been displayed;
it does not end the customer's browsing session.

## Implementation references

- [Catalog behavior and price rules](../../../../frontend/docs/ecommerce/product-catalog.md)
- [Landing page](../../../../frontend/src/ecommerce/pages/LandingPage.tsx)
- [Catalog lifecycle hook](../../../../frontend/src/ecommerce/hooks/useProductCatalog.ts)
- [Backend display-price service](../../../../backend/apps/ecommerce/services/display_prices.py)

Return to the [ecommerce index](README.md).
