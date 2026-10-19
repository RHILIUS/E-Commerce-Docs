# Registration OTP email delivery

An emailed one-time password (OTP) proves access to the customer's email before
creating an account. The API saves a code and queues delivery; the worker sends
it later. A queued response does not guarantee an inbox message, and a later
worker failure cannot change a response the browser already received.

Read A for issuance, B for worker delivery, and C for the existing verification
and account-creation handoff. Code replacement, expiry, and failed attempts belong
to the backend; email transport does not decide whether registration is allowed.

**Status:** Implemented in source. Run a consumer for the `registration-email` queue.
No database migration is required. Email delivery uses Celery; OTP verification
and account creation retain their existing contracts.

## Scope and notation

Registration requests and resends share phase A. Phase B runs asynchronously;
verification does not wait for a worker status response. These are activity-style
Mermaid swimlane business-process flowcharts, not strict UML activity diagrams.
See the [diagramming specification](../../diagramming-specification.md).

## A. Issue and queue a code

```mermaid
swimlane-beta TB
    subgraph A_UI["Frontend"]
        A_start(["Enter email or select Resend"])
        A_error(["Show error and retry time<br/>Correct or wait then retry A"])
        A_wait(["Show queued message and OTP input<br/>Start resend cooldown and continue to C"])
        A_unknown(["Outcome uncertain<br/>Check inbox and resend after cooldown"])
    end
    subgraph A_API["Backend"]
        A_guard{"Valid unregistered email<br/>and send limits permit issuance?"}
        A_save["Transaction: lock OTP record<br/>Save new hash and expiry<br/>Invalidate old code and update send budget"]
        A_commit{"Commit succeeded?"}
        A_publish["After commit: publish delivery task"]
        A_result{"Broker accepted task?"}
    end
    subgraph A_QUEUE["Task broker"]
        A_queue["Accept expiring task<br/>Continue to B"]
    end
    A_start -->|POST send OTP with email| A_guard
    A_start -.->|Response lost| A_unknown
    A_guard -->|No: 400 or 429| A_error
    A_guard -->|Yes| A_save --> A_commit
    A_commit -->|No: rollback and service error| A_error
    A_commit -->|Yes| A_publish
    A_publish -->|Code delivery task| A_queue
    A_queue -->|Publish outcome| A_result
    A_result -->|Yes: 200 with cooldown| A_wait
    A_result -->|Failed or uncertain: 503 with cooldown| A_error
```

The database transaction serializes issuance using row locks and unique email
creation. Code hashes, expiry and counters commit together. Publishing happens
after commit and cannot roll them back. Database errors use existing server-error
handling; they do not claim successful queuing. Publishing errors return the
existing retry response shape. A process crash between commit and publish can
lose dispatch; this implementation has no transactional outbox. Recovery is
controlled resend. An uncertain publishing outcome may still result in an email.

## B. Deliver the email

```mermaid
swimlane-beta TB
    subgraph B_WORKER["Celery worker"]
        B_start(["Receive task from A"])
        B_read["Read current database OTP state"]
        B_valid{"Matching code, unexpired<br/>and not verified?"}
        B_skip(["Discard obsolete task"])
        B_send["Send the same code"]
        B_result{"Provider outcome?"}
        B_retry{"Temporary failure, retry budget<br/>and expiry permit retry?"}
        B_delay["Retry after bounded backoff"]
        B_fail(["Log sanitized terminal failure<br/>Customer can resend after cooldown"])
        B_done(["Provider accepted email<br/>Inbox delivery not guaranteed"])
    end
    subgraph B_MAIL["Email provider"]
        B_mail["Accept verification email for delivery"]
    end
    B_start --> B_read --> B_valid
    B_valid -->|No| B_skip
    B_valid -->|Yes| B_send
    B_send -->|Email| B_mail
    B_mail -->|Acceptance or transport failure| B_result
    B_result -->|Accepted| B_done
    B_result -->|Failed or uncertain| B_retry
    B_retry -->|Yes| B_delay --> B_read
    B_retry -->|No| B_fail
```

The worker owns delivery checks and retry decisions; backend services own OTP
issuance and verification. No database lock spans an SMTP call. A resend can
invalidate a code after the worker checks it, so an old email can still arrive;
verification accepts only the current code. Expired broker tasks are discarded.
Retries use the same destination and code, with three retries at 5, 10 and 20
seconds, only when the next attempt precedes expiry. Transport disconnects,
OS/network failures and SMTP 4xx responses are retryable; other errors wrapped
by the email service are terminal. SMTP timeout follows `EMAIL_TIMEOUT`.

Task re-execution never creates codes or changes issuance budgets. SMTP has no
exactly-once guarantee: uncertain acceptance or duplicate execution can produce
duplicate messages. There is no delivery-status endpoint or automatic frontend
notification of later worker failures. Monitor queue age and terminal warnings.

## C. Verify and finish registration

From A, the customer enters the code received through B. Frontend submits email
and code to Backend. Backend locks the record, rejects missing, consumed or
expired codes, and commits incorrect-attempt increments (deleting the record at
the attempt limit). An eligible correct code is consumed and verification returns
the existing HttpOnly registration cookie. Frontend advances to account details
only after successful verification. Invalid codes remain on the verification
screen; expired/exhausted codes require another request in A.

Backend uses the registration cookie to authorize account creation and atomically
creates User, personal address and default recipient. Invalid account details
cannot create an account. Successful registration clears the cookie. Queueing
an email grants no registration authorization.

## API handoffs

| Operation         | Request                                                             | Success                                                                           | Failure and recovery                                                                                                   |
| ----------------- | ------------------------------------------------------------------- | --------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| Request or resend | `POST /api/auth/send-otp/`, `email`                                 | 200: `email`, `message` = `Verification email queued.`, `retry_after`             | 400 field errors; 429 or publish-failure 503 with `detail`, `retry_after`, `Retry-After`; correct or wait before retry |
| Verify            | `POST /api/auth/verify-otp/`, `email`, `otp`                        | Existing message and HttpOnly registration cookie scoped to `/api/auth/register/` | Existing validation, expiry and attempt-limit errors; correct or restart A                                             |
| Register          | `POST /api/auth/register/`, account details and registration cookie | Existing message and cleared registration cookie                                  | Existing validation/cookie errors; correct details or restart verification                                             |

## Rules, security and exclusions

- Six-digit code; hash-only database storage; ten-minute lifetime from issuance.
- Sixty-second resend cooldown; five sends per email per fifteen minutes; five
  incorrect verification attempts. Delivery failures do not refund send budgets.
- Resending replaces the code and resets its failed-attempt count, as before.
- Task argument representations are redacted and results disabled. The broker
  payload contains the plaintext code and email: restrict broker access, protect
  transport where required and avoid payload logging or unprotected snapshots.
- Queue routing isolates email from payment recovery. A dedicated consumer must
  run; broker acceptance alone does not establish consumer health.
- Password-change email uses the [separate password-change process](../account/change-password.md)
  on the same dedicated queue. Forgot-password flows, outbox persistence, and
  delivery-status polling are not implemented by registration delivery.

## Implementation references

- [Issuance](../../../../backend/apps/authentication/services/otp_service.py)
- [Post-commit publishing](../../../../backend/apps/authentication/services/otp_delivery.py)
- [Worker](../../../../backend/apps/authentication/tasks.py)
- [API contract](../../../../backend/docs/api/authentication/registration-otp.md)
- [Frontend OTP page](../../../../frontend/src/authentication/pages/RegisterOTPPage.tsx)
