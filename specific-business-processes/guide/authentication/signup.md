# Signup

Signup creates an account after the customer proves access to their email.
An OTP is a one-time password: a short code sent by email. **Status: Implemented;
the diagram contains older details noted below.** The
[flowchart](../../diagrams/authentication/signup.drawio.svg) follows email
verification, account details and account creation across Frontend, Backend and
Email Provider.

## Follow the flow

1. Open `/register/enter-email`, enter an email and request a code. The backend
   rejects an already registered email, enforces send limits and saves a hashed
   code. A hash lets it check the code without storing it as readable text.
2. The backend queues an email task. The worker sends the code if it is still
   current and unexpired. A queued response does not prove inbox delivery.
3. Enter the six-digit code on the verification page. A correct, unexpired code
   is consumed and the backend sets a short-lived registration cookie.
4. Enter personal details, postal address and password in the account-details
   steps. The frontend validates each step before submitting one creation request.
5. The backend derives the email from the verification cookie, validates the
   details and creates the user, personal address and initial delivery recipient
   together. A failure rolls back these records together.
6. After creation, the frontend makes a separate login request. Success opens
   `/`; if automatic login fails, it opens Login and explains that the account
   was created and the customer should sign in.

The registration cookie is HttpOnly, so JavaScript cannot read it. It authorizes
account creation, not login. The browser sends it to the registration endpoint.

## Decisions and recovery

Incorrect codes can be retried within the attempt limit. Expired or exhausted
codes require another issuance, subject to send limits and cooldown. Resend
replaces the old code. Temporary worker delivery failures have bounded retries;
the customer can resend after cooldown if no email arrives.

Invalid account fields remain on the form for correction. An invalid registration
cookie asks the customer to verify email again. The backend checks uniqueness
again at creation because another request could have claimed the email meanwhile.
A lost creation response can mean the account already exists; it is not proof
that creation failed.

## Older diagram details

The drawing shows a direct email-provider call, latitude/longitude address input,
and a one-hour send window. Current delivery is queued, the address consists of
postal fields, and the configured default send window is 15 minutes. The diagram
also omits the automatic-login failure branch. Its password note is incomplete:
the current policy also requires a lowercase letter. Check the maintained
contract for exact current limits and fields.

## Implementation references

- [Backend account and registration contract](../../../../backend/docs/api/authentication/README.md)
- [Background email delivery](../../../../backend/docs/api/authentication/registration-otp.md)
- [Account-details page](../../../../frontend/src/authentication/pages/RegisterDetailsPage.tsx)

Return to the [authentication index](README.md).
