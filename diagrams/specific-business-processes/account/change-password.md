# Change Password with email OTP

**Status:** Implemented in backend and frontend source. Apply backend migration
`authentication.0004_user_authentication_version_passwordchangestate` before serving
the new code. The Change Password tab is at `/account?tab=change-password`.
See the [backend contract](../../../../backend/docs/api/account/change-password.md)
and [frontend handoff](../../../../frontend/docs/account/change-password.md).

## Scope and notation

Follow A → C to change a password, with B available to resend a code. Customer
interactions belong in Frontend; Backend owns authentication, validation and
transaction coordination; Database owns durable state; Celery worker validates pending challenges and delivers OTP emails; Email provider accepts
messages for delivery. No completion email is sent after changing a password. These are Mermaid swimlane business-process flowcharts,
not strict UML activity diagrams. See the [diagramming specification](../../diagramming-specification.md).

## A. Request a verification code

```mermaid
swimlane-beta TB
    subgraph A_UI["Frontend"]
        A_start(["Enter current and new passwords<br/>Confirm the new password"])
        A_error(["Show field, queuing or limit error<br/>Correct or wait then retry A"])
        A_ready(["Show queued email, masked address<br/>OTP input and countdown<br/>Continue to C or resend B"])
    end
    subgraph A_API["Backend"]
        A_guard{"Authenticated, within limits<br/>and password details valid?"}
        A_save["Transaction under user lock<br/>Replace challenge and save hash and expiry<br/>Persist account budgets"]
        A_commit{"Challenge committed?"}
        A_queue["After commit: publish expiring OTP task"]
        A_result{"Broker accepted task?"}
    end
    subgraph A_BROKER["Task broker"]
        A_accept["Accept task for D delivery"]
    end
    A_start -->|POST request with password details| A_guard
    A_start -.->|Response lost: outcome uncertain| A_error
    A_guard -->|No: validation, auth or limit error| A_error
    A_guard -->|Yes| A_save --> A_commit
    A_commit -->|No: rollback and 503| A_error
    A_commit -->|Yes| A_queue
    A_queue -->|User ID, challenge ID and code| A_accept
    A_accept -->|Publish outcome| A_result
    A_result -->|No or uncertain: 503 with cooldown| A_error
    A_result -->|Yes: 200 challenge with cooldown| A_ready
```

Password details remain in frontend memory and in the current API request; no
proposed new password is persisted or passed to Celery. Destination email comes
from the signed-in account. Broker acceptance does not prove inbox delivery.
Expected validation failures still persist applicable account budgets under the
existing service transaction. A database failure rolls back the operation.

## B. Resend the code

```mermaid
swimlane-beta TB
    subgraph B_UI["Frontend"]
        B_start(["From A: cooldown ends<br/>Select Resend code"])
        B_error(["Show error or retry time<br/>Wait and retry B or restart A"])
        B_ready(["Clear OTP input and restart countdown<br/>Use the newest code in C"])
    end
    subgraph B_API["Backend"]
        B_guard{"Authenticated owner and eligible challenge<br/>Send limits permit resend?"}
        B_save["Transaction under user lock<br/>Replace hash and expiry<br/>Preserve attempt budget and increment send count"]
        B_commit{"Replacement committed?"}
        B_queue["After commit: publish expiring OTP task"]
        B_result{"Broker accepted task?"}
    end
    subgraph B_BROKER["Task broker"]
        B_accept["Accept replacement task for D"]
    end
    B_start -->|POST resend with challenge ID| B_guard
    B_start -.->|Response lost: outcome uncertain| B_error
    B_guard -->|No: unavailable, auth or limit error| B_error
    B_guard -->|Yes| B_save --> B_commit
    B_commit -->|No: rollback and 503| B_error
    B_commit -->|Yes| B_queue
    B_queue -->|User ID, challenge ID and replacement code| B_accept
    B_accept -->|Publish outcome| B_result
    B_result -->|No or uncertain: 503 with cooldown| B_error
    B_result -->|Yes: 200 challenge with cooldown| B_ready
```

The previous code becomes invalid after replacement commits, including when
queuing or email delivery fails. Send limits remain enforced. Exhausted, consumed
or account-invalidated challenges cannot be revived by resending.

## C. Verify the code and change the password

### C1. Check the challenge and code

```mermaid
swimlane-beta TB
    subgraph C_UI["Frontend"]
        C_start(["From A or B: submit code<br/>with current and new passwords"])
        C_code_error(["Show incorrect code<br/>Retry C if attempts remain, otherwise restart A"])
        C_error(["Show error without success<br/>Correct, wait or restart A"])
        C_unknown(["Response lost: outcome uncertain<br/>Clear sensitive state and sign in"])
    end
    subgraph C_API["Backend"]
        C_auth{"Authenticated and within<br/>verification rate limits?"}
        C_challenge{"Owned, active, unexpired challenge<br/>and account unchanged?"}
        C_otp{"OTP matches?"}
        C_reject["End transaction<br/>No password change"]
        C_ready(["Continue to C2 in this request<br/>Keep transaction and user lock"])
    end
    subgraph C_DB["Database"]
        C_lock["Begin transaction<br/>Lock user and challenge"]
        C_attempt["Commit failed attempt<br/>Invalidate at attempt limit<br/>No password change"]
        C_rollback["Roll back database work"]
    end
    C_start -->|POST confirm| C_auth
    C_start -.->|Network outcome unknown| C_unknown
    C_auth -->|No: authentication or rate-limit error| C_error
    C_auth -->|Yes: begin verification| C_lock
    C_lock -->|Locked state| C_challenge
    C_lock -->|Failure| C_rollback
    C_challenge -->|No| C_reject
    C_reject -->|Validation error| C_error
    C_challenge -->|Yes| C_otp
    C_otp -->|No: record failed code| C_attempt
    C_attempt -->|400: code error| C_code_error
    C_attempt -->|Failure| C_rollback
    C_rollback -->|503: uncertain to frontend| C_unknown
    C_otp -->|Yes| C_ready
```

### C2. Commit the password change

This continues the **same confirmation request and transaction** from C1. There
is no second frontend request or separate verified-code token.

```mermaid
swimlane-beta TB
    subgraph C2_UI["Frontend"]
        C2_error(["Show password field error<br/>Correct details or restart A"])
        C2_unknown(["Server error: outcome uncertain to frontend<br/>Clear sensitive state and sign in"])
        C2_done(["Open login with success message<br/>Clear passwords, OTP and authentication"])
    end
    subgraph C2_API["Backend"]
        C2_start(["From C1: code verified<br/>Transaction and locks remain active"])
        C2_password{"Current password correct<br/>New password valid and different?"}
        C2_reject["End transaction<br/>No password change"]
        C2_success["After commit: clear refresh cookie<br/>Return success without another email"]
        C2_start --> C2_password
        C2_password -->|No| C2_reject
    end
    subgraph C2_DB["Database"]
        C2_write["Set hashed password<br/>Increment authentication version<br/>Consume challenge"]
        C2_commit["Commit all changes together"]
        C2_rollback["Roll back password, version<br/>and challenge changes"]
        C2_write -->|Writes succeeded| C2_commit
        C2_write -->|Failure| C2_rollback
        C2_commit -->|Failure| C2_rollback
    end
    C2_reject -->|Validation error| C2_error
    C2_password -->|Yes: authorize update| C2_write
    C2_rollback -->|503: no partial password change| C2_unknown
    C2_commit -->|Committed| C2_success
    C2_success -->|Success response| C2_done
```

A database transaction serializes competing confirmation requests. Incorrect OTP
attempts commit independently of the password-change success path. Password
validation failures release locks without consuming the challenge. Recheck every
guard on retry; a previously valid OTP is not a lasting authorization grant.

## D. Deliver an OTP in the background

This phase receives tasks from A or B, independently of frontend confirmation.
Successful password change in C sends no completion notification.

```mermaid
swimlane-beta TB
    subgraph D_WORKER["Celery worker"]
        D_start(["Receive OTP task from A or B"])
        D_read["Read challenge and current owner"]
        D_valid{"Current challenge and matching code<br/>Active owner, same email and credential state<br/>Unconsumed and unexpired?"}
        D_skip(["Discard obsolete task"])
        D_send["Send the same OTP to registered email"]
        D_outcome{"Email provider outcome?"}
        D_retry{"Temporary failure, retries remain<br/>and next attempt precedes expiry?"}
        D_delay["Schedule retry with bounded backoff"]
        D_fail(["Log sanitized failure with task ID<br/>Controlled resend remains available"])
        D_done(["Provider accepted email<br/>Inbox delivery remains unconfirmed"])
    end
    subgraph D_MAIL["Email provider"]
        D_mail["Accept verification email for delivery"]
    end
    D_start --> D_read --> D_valid
    D_valid -->|No| D_skip
    D_valid -->|Yes| D_send
    D_send -->|OTP email| D_mail
    D_mail -->|Acceptance or transport failure| D_outcome
    D_outcome -->|Accepted| D_done
    D_outcome -->|Failed or uncertain| D_retry
    D_retry -->|Yes| D_delay --> D_read
    D_retry -->|No| D_fail
```

Delivery uses the existing `registration-email` queue, consumed by the dedicated
email worker, independently of payment recovery. Each attempt rechecks the
challenge against persisted state and current owner credentials. Three retries
use delays of 5, 10 and 20 seconds; no retry is scheduled beyond expiry. Retries
never generate another code or increment issuance budgets. The shared transport
policy retries network/disconnect errors and SMTP 4xx failures, including
recipient refusals; permanent transport failures terminate.

No database lock spans SMTP. Concurrent resend or confirmation can invalidate a
code after the check, so an obsolete email may still arrive, but cannot authorize
a change. Provider timeouts and duplicate execution can produce duplicate emails;
exactly-once delivery is not guaranteed. Task payloads contain identifiers and
the plaintext OTP, never passwords; task argument displays are redacted on both
initial publication and retry. Results are disabled. Protect broker access,
transport and snapshots, and avoid payload logging.

Publishing happens after commit. Broker failure retains the challenge and
cooldown and returns the existing `delivery_unavailable` 503 contract. A process
crash between commit and publish can lose dispatch; an uncertain publication may
still deliver email. There is no outbox, delivery-status endpoint or later frontend
failure notification. Recovery is controlled resend. Monitor queue age and terminal
warnings. A pending OTP task is discarded after confirmation consumes its challenge.

## API handoffs

All operations require authentication and use the signed-in account.

| Operation              | Endpoint                                     | Request body                                              | Successful result                                           |
| ---------------------- | -------------------------------------------- | --------------------------------------------------------- | ----------------------------------------------------------- |
| Request and queue code | `POST /api/account/change-password/request/` | `current_password`, `new_password`                        | `challenge_id`, `masked_email`, `expires_at`, `retry_after` |
| Resend and queue code  | `POST /api/account/change-password/resend/`  | `challenge_id`                                            | `challenge_id`, `masked_email`, `expires_at`, `retry_after` |
| Confirm change         | `POST /api/account/change-password/confirm/` | `challenge_id`, `otp`, `current_password`, `new_password` | Success message and cleared refresh cookie                  |

Confirmation-password matching is frontend validation. Backend password checks
run during both request and confirmation. Registration OTP endpoints cannot
issue password-change authorization.

## Rules and recovery

| Rule or condition                     | Required behavior                                                                                          |
| ------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| OTP policy                            | Six random digits, protected storage, five-minute expiry, single use                                       |
| Resend limits                         | 60-second cooldown, five sends per account per 15 minutes; account/IP throttling                           |
| Verification limits                   | Five incorrect attempts per challenge plus account/IP throttling                                           |
| Restart or resend                     | Account limits survive challenge replacement; resend preserves failed-attempt budget                       |
| Challenge binding                     | User, purpose, destination email and account credential state; one active challenge per user               |
| Email or password changed since issue | Invalidate challenge and restart A                                                                         |
| Invalid current/new password          | Field validation error, not authentication-expired HTTP 401                                                |
| Invalid OTP                           | Commit attempt counter; retry C if eligible, otherwise restart A                                           |
| Expired code                          | Resend B only if challenge remains eligible; otherwise restart A                                           |
| Authentication expired                | Existing refresh/sign-in flow; restart A if sign-in is required                                            |
| Queuing unavailable                   | Retain challenge and rate limits, display 503 and cooldown, permit controlled retry                        |
| Worker delivery failure               | Retry temporary transport failures within expiry; no new API response, controlled resend remains available |
| Unknown confirmation outcome          | Do not automatically replay mutation or claim failure; direct to sign in                                   |
| Leave tab, refresh or sign out        | Clear in-memory passwords and OTP; server challenge expires normally                                       |
| Old sessions after success            | Reject old access and refresh tokens through authentication-version checks                                 |
| Pending frontend refresh              | Prevent late refresh responses from restoring authentication after success                                 |

Incrementing the authentication version must invalidate tokens on both protected
requests and refresh. Legacy tokens without a version claim work only while the account version is
zero; once a password changes they fail. Refresh adds the checked version claim
when rotating a legacy token. Blacklisting only the current refresh token does
not satisfy signing out all devices.

Current implementation context: registration uses six-digit OTPs with a
10-minute expiry, a 60-second cooldown and five attempts; its policy and records
remain separate. Authentication uses access tokens and an HttpOnly refresh cookie.
The current frontend omits bearer headers on `/api/auth/`, so this feature uses the authenticated account routes above.

## Scope exclusions

Forgot Password recovery, account-email changes and authenticator/passkey MFA
are separate processes. The feature changes passwords only after valid current-password and email-code checks.

## Persisted state and limits

`User.authentication_version` starts at zero and increments atomically with the
password update. `PasswordChangeState` is a dedicated one-to-one user record;
replacing its challenge UUID preserves account send/request/verification counters.
It stores the email snapshot, a keyed credential-state digest, OTP hash, code
expiry, last-send time, consumed state, failed attempts and independent window
start times. It stores no proposed password and shares no registration OTP state.
See the [model diagram](../../erd/password-change.md).

Account request and verification budgets each allow 20 operations per 15 minutes,
including failed password checks or stale identifiers. These database budgets
survive application restarts and challenge replacements. IP throttling allows
20 operations per minute across the three endpoints using the configured Django
cache. Cache-based IP throttling is an additional approximate guard; configure
shared cache and trusted proxy handling for multi-worker deployments.

See the [diagram index](../../README.md).
