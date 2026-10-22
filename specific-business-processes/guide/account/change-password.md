# Change Password

**Status: Implemented in frontend and backend source.** Matching deployment and
email workers are required; source implementation does not confirm delivery.

Signed-in customers open Account → Change Password to replace their password
after verifying access to their account email. They do not enter the current
password or type an email destination. The [editable flowchart](../../diagrams/account/change-password.drawio.svg)
shows the implemented process.

The [frontend guide](../../../../frontend/docs/account/change-password.md)
explains the revised three-stage interface. The
[backend contract](../../../../backend/docs/api/account/change-password.md)
defines the separate verification and save operations. No current-password input
is required. Release the matching frontend/backend versions together and apply
the state migration before use.

## Read the diagram

Follow Request Code, Receive Email, Verify Code, and Change Password from left
to right. Frontend contains the account tab, customer input, and browser state.
Backend contains account authorization, database records, queue publication,
and email workers. Email Provider accepts messages for delivery; acceptance
does not guarantee arrival in the inbox.

Ovals mark the start and completion. Displays show customer screens; manual-input
shapes show customer actions. Diamonds ask questions with labeled outcomes.
Solid arrows stay within an environment; dashed arrows cross environments.
Bracketed annotations explain rules beside their owning step.

## Customer journey

1. Open Account → Change Password while signed in. Show the masked account email
   and a Send code action. Opening the tab alone does not send email.
2. Select Send code. The backend uses the signed-in account's email, creates a
   short-lived challenge, and queues a one-time password (OTP): a six-digit code.
   Show the code input and resend countdown after queue acknowledgement.
3. Read the latest email and verify its code. A valid code grants temporary
   permission to change this account's password; verification itself does not
   change the password or end sessions.
4. Enter and confirm the new password. The browser checks that the entries match;
   the backend enforces password policy and rejects the existing password.
5. Save once. The backend atomically saves the password, consumes permission,
   and invalidates existing sessions. Return to Login with a success message.
   Queue a password-changed notification after commit.

## Authorization and saved state

The flow uses a dedicated account password-change challenge and permission, separate from
registration and public Forgot Password. The account comes from authenticated
server state, never a customer-supplied email or account identifier. Every operation requires a
valid account session for request, resend, verification, and final save.

The backend uses the existing recovery pattern for temporary permission:
a separate scoped HttpOnly cookie set by the backend. JavaScript cannot read
the secret. The backend stores only its digest, binds it to the account, email,
and credential state, and enforces expiry and single use. Cookie-bearing requests
need CSRF protection as well as account authentication. The backend contract specifies routes, cookie name and scope, and payloads; the public recovery cookie must not authorize this flow.

The backend uses recovery's five-minute codes, 60-second resend cooldown,
ten-minute permission, and durable 15-minute budgets: 20 requests, five sends,
20 verifications, and five incorrect codes. IP throttling also applies. Resend and
restart preserve attempt budgets; replacing a challenge rotates its identifier
and code and invalidates previous permission. The frontend consumes the replacement identifier and handles verification
separately from saving.

OTP and password drafts stay in memory. They are not persisted in browser storage,
and passwords never enter the email queue. Leaving the account page or signing out
clears drafts; restarting verification clears password inputs. Cooldown
and expiry deadlines survive switching account tabs. Backend checks remain
authoritative if a page is reloaded or its countdown is manipulated.

## Decisions and recovery

- **Session unavailable:** sign in again before restarting. Authentication failure
  at any backend step blocks progression; an old cookie alone cannot authorize a
  change for a different or signed-out account.
- **Cooldown or rate limit:** show the server's retry time and wait. Restarting
  must not bypass durable limits.
- **Email unavailable:** retain the cooldown and offer controlled resend. Queue
  failure may occur after challenge creation. Workers skip stale challenges and
  retry temporary delivery failures only within expiry; terminal failures are
  logged without secrets.
- **Wrong code:** allow correction while attempts remain. An expired code needs
  resend. An exhausted budget requires waiting before restarting. Unavailable or
  replaced challenges require a fresh verification flow.
- **Permission unavailable:** verify email again. Account deactivation or changes
  to email or credentials invalidate outstanding permission.
- **Password rejected:** correct the password while permission remains valid.
  A browser mismatch never submits the save request.
- **Competing saves or database failure:** lock the account and permission and
  recheck authorization. Only one confirmation may consume permission. Roll back
  password, permission consumption, and session invalidation together on failure.
- **Final response uncertain:** return to Login without claiming success or
  failure, and never automatically replay the save. Try signing in with the new
  password; use recovery if needed.

Successful save clears the change permission and local authentication. All prior
access and refresh sessions fail the existing authentication-version checks.
Notification failure cannot undo a completed password change. The existing email
architecture has no durable outbox, so guaranteed notification delivery is not
assumed by this flow.

## Relationship to Forgot Password

[Forgot Password](../authentication/forgot-password.md) remains public recovery
for customers who cannot sign in. Its email-verification and save phases inform
this flow, but the account tab uses a fixed account email and authenticated,
separate authorization. Current Forgot Password is already implemented; the account revision is implemented in both applications. Its authenticated
permission remains separate from public recovery.

Sibling application links assume the frontend, backend and documentation
repositories share a parent checkout directory. Return to the [account SBPs](README.md).
