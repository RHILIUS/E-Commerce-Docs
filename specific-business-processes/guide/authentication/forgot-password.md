# Forgot Password

**Status: Approved design; implementation pending.**

Customers who cannot sign in can verify access to their account email and choose
a new password. The [editable flowchart](../../diagrams/authentication/forgot-password.drawio.svg) covers this
recovery process only. Account → Change Password retains its existing behavior.

## Read the flow

Follow the four columns from left to right: Request Code, Receive Email, Verify
Code, and Reset Password. Frontend contains browser screens and customer input.
Backend contains API validation, database changes, the task broker, and the email
worker. Email Provider accepts messages for delivery; acceptance does not prove
that the message reached the inbox.

Ovals mark Start and End. Rounded rectangles are processes, display shapes are
screens or messages, manual-input shapes are customer input, parallelograms are
requests or responses, and diamonds are decisions. The stored-data shape denotes
the browser's HttpOnly cookie. Solid arrows stay within an environment; dashed
arrows communicate across environments. Bracketed notes explain their step.

## Customer experience

1. Select Forgot password on Login, then enter the account email.
2. See the same acknowledgement whether or not an eligible account exists.
3. Read the latest email code, enter it, and select Verify. Resend becomes
   available after cooldown. Incorrect codes allow retry while attempts remain;
   expired or exhausted challenges require resend or restart as permitted.
4. After verification, enter and confirm a new password. Mismatched passwords
   stay on the form; the backend also validates password policy.
5. After a confirmed save, return to Login with a success message. Existing
   sessions are invalidated, and a password-changed notification is queued.

## Approved authorization and recovery rules

Registration codes, account change-password challenges, and password-reset
challenges remain separate. Request and resend apply IP and durable account
limits. Resend replaces the previous code without resetting the failed-attempt
budget. Requests for unknown accounts receive equivalent public responses and
cooldowns. Delivery failures must not reveal which addresses have accounts.

Email delivery runs in the background after challenge commit. The worker checks
that the challenge is current and unexpired before sending, retries temporary
failures within expiry, and logs terminal failures without secrets. Customers can
use controlled resend when an email does not arrive. A queue acknowledgement
does not confirm inbox delivery.

Successful verification consumes the OTP and grants short-lived, single-use
permission to reset that account's password. The backend sets a scoped HttpOnly
cookie; frontend JavaScript cannot read it. Cookie-based reset requests need
appropriate CSRF protection. OTP and password inputs stay in memory. Changes to
the account email or credentials invalidate outstanding reset permission.

The final operation locks the account and reset grant, rechecks authorization
and password policy, and atomically saves the password, consumes permission, and
invalidates prior sessions. Competing requests cannot reuse a consumed grant.
Failure rolls back all these changes. Requesting or verifying a code does not
change the password or invalidate sessions.

An invalid grant restarts email verification; password-policy errors return to
password entry. After an uncertain final response, return to Login without
claiming success or failure and without automatically replaying the mutation.
The notification is queued after commit and cannot roll back a completed reset.

## Details to finalize during implementation

Exact endpoint paths, payloads, response codes, cookie attributes, CSRF handling,
and authorization expiry will be established in the backend API contract. The
diagram uses operation names rather than invented routes. The approved starting
policy is a six-digit code, five-minute expiry, a 60-second resend cooldown, and
five incorrect attempts per challenge, with durable send and verification limits.

Reuse existing UI controls and email infrastructure where suitable; preserve the
current password policy and session-version checks. This diagram does not
authorize changes to Account → Change Password.

Return to the [authentication SBPs](README.md).
