# E-Commerce diagrams

Specific business processes (SBPs) use **Mermaid flowcharts grouped into swimlanes**
to show responsibility, decisions, validation and recovery. **UML sequence diagrams**
are supplementary views for request ordering and interactions.

Read the [diagramming specification](diagramming-specification.md)
for notation, placement and validation requirements.

## Checkout

- [Cash on Delivery checkout](specific-business-processes/checkout/cod-checkout.md) — implemented purchase flow.
- [Demo online checkout and USD wallets](specific-business-processes/checkout/demo-online-checkout.md) — implemented Stripe test payment, Celery payment recovery and pending wallet credits; seven-day settlement and mock withdrawals remain proposed.
- [Proposed wallet and settlement ERD](erd/demo-wallet-ledger.md) — wallet and USD journal foundation implemented; settlement, holds and withdrawals remain proposed.

## Account

- [Change Password with email OTP](specific-business-processes/account/change-password.md) — implemented Celery OTP delivery, resend, atomic password change and session invalidation without a completion email; migration 0004 must be applied.

- [Password-change model](erd/password-change.md) — dedicated user challenge state, durable budgets and authentication version.

## Development

- [Reset development environment](specific-business-processes/development/reset-development.md) — guarded schema and media reset, migrations, taxonomy import and seed.

## Authentication

- [Registration OTP email](specific-business-processes/authentication/registration-otp-email.md) — Celery delivery, controlled retries and existing verification flow.

## Deployment

- [Same-origin API proxy](specific-business-processes/deployment/same-origin-api-proxy.md) — Pages-to-EC2 API forwarding and host-only cookie behavior; implemented in source, deployment verification pending.
