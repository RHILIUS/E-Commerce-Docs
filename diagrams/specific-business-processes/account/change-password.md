# Change Password with email OTP

**Status:** Implemented in backend and frontend source. Apply backend migration
`authentication.0004_user_authentication_version_passwordchangestate` before serving
the new code. The Change Password tab is at `/account?tab=change-password`.
See the [backend contract](../../../../backend/docs/account/change-password-api.md)
and [frontend handoff](../../../../frontend/docs/account/change-password.md).

## Scope and notation

Follow A → C to change a password, with B available to resend a code. Customer
interactions belong in Frontend; Backend owns authentication, validation and
transaction coordination; Database owns durable state; Email provider accepts
messages for delivery. These are Mermaid swimlane business-process flowcharts,
not strict UML activity diagrams. See the [diagramming specification](../../diagramming-specification.md).

## A. Request a verification code

```mermaid
swimlane-beta TB
    subgraph A_UI["Frontend"]
        A_start(["Open Change Password tab"])
        A_form["Enter current password<br/>New password and confirmation"]
        A_valid{"Fields valid and<br/>new passwords match?"}
        A_fix["Show field errors"]
        A_error(["Show field, delivery or rate-limit error<br/>Correct or wait, then retry A"])
        A_ready(["Show masked email, OTP input and countdown<br/>Continue to C or resend in B"])
        A_start --> A_form --> A_valid
        A_valid -->|No| A_fix --> A_form
    end
    subgraph A_API["Backend"]
        A_guard{"Authenticated, within limits<br/>and password details valid?"}
        A_send["Request OTP email<br/>Use registered account email"]
        A_result{"Email provider accepted message?"}
    end
    subgraph A_DB["Database"]
        A_save["Atomically invalidate prior challenge<br/>Save user-bound OTP hash and expiry<br/>Persist account send counters"]
    end
    subgraph A_MAIL["Email provider"]
        A_mail["Accept verification email for delivery"]
    end
    A_valid -->|Yes: POST request| A_guard
    A_guard -->|No: validation, authentication or limit error| A_error
    A_guard -->|Yes: create challenge| A_save
    A_save -->|Commit succeeded| A_send
    A_save -->|Failure: rollback| A_error
    A_send -->|Verification email| A_mail
    A_mail -->|Acceptance or failure| A_result
    A_result -->|No: delivery error| A_error
    A_result -->|Yes: challenge ID, expiry and cooldown| A_ready
```

Provider acceptance does not guarantee inbox delivery. Password details remain
in frontend memory; the proposed new password is not persisted on the backend.
Every request uses the authenticated user's email, never a client-selected address.

## B. Resend the code

```mermaid
swimlane-beta TB
    subgraph B_UI["Frontend"]
        B_start(["From A: cooldown ends<br/>Customer selects Resend code"])
        B_db_error(["Show service error<br/>Wait and retry B or restart A"])
        B_error(["Show retry time or unavailable challenge<br/>Wait and retry B, or restart A"])
        B_ready(["Clear OTP input and restart countdown<br/>Continue to C using newest code"])
    end
    subgraph B_API["Backend"]
        B_guard{"Authenticated owner, eligible challenge<br/>and send limits permit resend?"}
        B_send["Request replacement OTP email"]
        B_result{"Email provider accepted message?"}
    end
    subgraph B_DB["Database"]
        B_replace["Atomically replace OTP hash and expiry<br/>Increment account send counters<br/>Preserve failed-attempt budget"]
    end
    subgraph B_MAIL["Email provider"]
        B_mail["Accept replacement email for delivery"]
    end
    B_start -->|POST resend with challenge ID| B_guard
    B_guard -->|No: unavailable, authentication or limit error| B_error
    B_guard -->|Yes: replace code| B_replace
    B_replace -->|Commit succeeded| B_send
    B_replace -->|503: rollback| B_db_error
    B_send -->|Replacement verification email| B_mail
    B_mail -->|Acceptance or failure| B_result
    B_result -->|No: delivery error| B_error
    B_result -->|Yes: expiry and cooldown| B_ready
```

The previous code becomes invalid after replacement commits, including when
email delivery fails. Send limits remain enforced on delivery failure. Exhausted,
consumed or account-invalidated challenges cannot be revived by resending.

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
        C2_success["After commit: clear refresh cookie<br/>Return success and attempt D notification"]
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

## D. Send the completion notification

```mermaid
swimlane-beta TB
    subgraph D_API["Backend"]
        D_start(["From C: password-change transaction committed"])
        D_send["Attempt password-change notification<br/>Never include the password"]
        D_result{"Email provider accepted message?"}
        D_record["Record delivery failure without secrets"]
        D_end(["Notification attempt finished<br/>Password change remains committed"])
        D_start --> D_send
        D_result -->|Yes| D_end
        D_result -->|No| D_record --> D_end
    end
    subgraph D_MAIL["Email provider"]
        D_mail["Accept notification email for delivery"]
    end
    D_send -->|Completion notification| D_mail
    D_mail -->|Acceptance or failure| D_result
```

Notification is a post-commit side effect. Its failure must not roll back the
password or turn the committed change into a failed password-change response.
No background worker or guaranteed delivery is provided.

## API handoffs

All operations require authentication and use the signed-in account.

| Operation | Endpoint | Request body | Successful result |
| --- | --- | --- | --- |
| Request code | `POST /api/account/change-password/request/` | `current_password`, `new_password` | `challenge_id`, `masked_email`, `expires_at`, `retry_after` |
| Resend code | `POST /api/account/change-password/resend/` | `challenge_id` | `challenge_id`, `masked_email`, `expires_at`, `retry_after` |
| Confirm change | `POST /api/account/change-password/confirm/` | `challenge_id`, `otp`, `current_password`, `new_password` | Success message and cleared refresh cookie |

Confirmation-password matching is frontend validation. Backend password checks
run during both request and confirmation. Registration OTP endpoints cannot
issue password-change authorization.

## Rules and recovery

| Rule or condition | Required behavior |
| --- | --- |
| OTP policy | Six random digits, protected storage, five-minute expiry, single use |
| Resend limits | 60-second cooldown, five sends per account per 15 minutes; account/IP throttling |
| Verification limits | Five incorrect attempts per challenge plus account/IP throttling |
| Restart or resend | Account limits survive challenge replacement; resend preserves failed-attempt budget |
| Challenge binding | User, purpose, destination email and account credential state; one active challenge per user |
| Email or password changed since issue | Invalidate challenge and restart A |
| Invalid current/new password | Field validation error, not authentication-expired HTTP 401 |
| Invalid OTP | Commit attempt counter; retry C if eligible, otherwise restart A |
| Expired code | Resend B only if challenge remains eligible; otherwise restart A |
| Authentication expired | Existing refresh/sign-in flow; restart A if sign-in is required |
| Delivery unavailable | Retain rate limits, display error, permit controlled retry |
| Unknown confirmation outcome | Do not automatically replay mutation or claim failure; direct to sign in |
| Leave tab, refresh or sign out | Clear in-memory passwords and OTP; server challenge expires normally |
| Old sessions after success | Reject old access and refresh tokens through authentication-version checks |
| Pending frontend refresh | Prevent late refresh responses from restoring authentication after success |

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

## Diagram validation

All five process Mermaid fences and the companion model diagram were rendered
with Mermaid CLI **12.0.0** and Mermaid **12.2.0**. Rendered lanes, labels and
phase continuations were inspected. Relative links resolve. Local rendering does
not establish support in GitHub or another Markdown host; `swimlane-beta`
requires a compatible Mermaid 12 host, whose rendering remains unverified.

See the [diagram index](../../README.md).
