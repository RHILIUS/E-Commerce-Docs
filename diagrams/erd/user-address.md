# User personal-address model

Implemented on 2026-10-13. This scoped model reference supersedes the User country
placement in the historical draw.io ERD/UML for authentication and personal
addresses. Those historical diagrams are not synchronized with this change.

| Model | Relationship / fields | Responsibility |
| --- | --- | --- |
| authentication.User | id, username, email, full_name, phone_number, profile_image, profile_image_crop; no country_code | Account identity and contact information |
| authentication.UserAddress | Unique user_id FK to User; country_code, address_line1, address_line2, city, region, postal_code | One personal address per user, separate from saved delivery recipients |
| account.RecipientInfo | user_id FK to User; name, phone_number, country_code, structured address, default/deletion flags | Zero or more delivery contacts and addresses per account |

Deleting User cascades to UserAddress. Accounts created manually or through
createsuperuser may have no address. A saved address requires country_code,
address_line1 and city; address_line2, region and postal_code are optional.
Registration and seed fixtures create complete personal addresses. No country
is inferred from a phone number or delivery recipient.

Registration keeps its flat address request and creates User, UserAddress and
an initial default RecipientInfo in one transaction. Address details are copied
at creation only. Later changes to either address do not synchronize the other.
Login/refresh expose nullable `user.address_country_code`, derived from the
personal address. Frontend country-dependent resources use that value.

Catalog display conversion and checkout target-currency resolution read the
personal address, with the existing USD target when absent/unresolved. Checkout
retains its capability checks and strict FX failure behavior. Stripe still uses
its existing explicit USD target; store currencies and saved purchase snapshots
are unchanged.

The new authentication migration intentionally discards the former User country
column without backfill for the authorized disposable test database. Reset and
reseed it. Historical migrations remain intact. Profile editing endpoints,
Profile-tab UI and photo cropping are subsequent work.

See [backend API documentation](../../../backend/docs/api-documentation.md),
[seed instructions](../../../backend/docs/seed.md), and
[frontend ownership notes](../../../frontend/docs/account/recipients.md).
