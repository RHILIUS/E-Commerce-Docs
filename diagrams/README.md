# E-Commerce diagrams

Specific business processes (SBPs) use **Mermaid flowcharts grouped into swimlanes**
to show responsibility, decisions, validation and recovery. **UML sequence diagrams**
are supplementary views for request ordering and interactions.

Read the [diagramming specification](diagramming-specification.md)
for notation, placement and validation requirements.


## Checkout

- [Cash on Delivery checkout](specific-business-processes/checkout/cod-checkout.md) — implemented purchase flow.
- [Demo online checkout and USD wallets](specific-business-processes/checkout/demo-online-checkout.md) — implemented Stripe test payment and pending wallet credits; seven-day settlement and mock withdrawals remain proposed.
- [Proposed wallet and settlement ERD](erd/demo-wallet-ledger.md) — wallet and USD journal foundation implemented; settlement, holds and withdrawals remain proposed.

## Authentication

- [User personal-address model](erd/user-address.md) — implemented ownership,
  registration, authentication-country and pricing contract; supersedes the
  personal-country placement in the historical draw.io ERD/UML.

## Account

- [Change Password with email OTP](specific-business-processes/account/change-password.md) — implemented verification, resend, atomic password change and session invalidation; migration 0004 must be applied.

- [Password-change model](erd/password-change.md) — dedicated user challenge state, durable budgets and authentication version.
