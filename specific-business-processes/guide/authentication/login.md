# Login

Login lets a customer use account features by entering their email and password.
**Status: Implemented.** Follow the [flowchart](../../diagrams/authentication/login.drawio.svg)
through Enter Credentials, Validate Credentials and Establish Session.

## Follow the flow

1. Open `/login` directly or arrive after trying to open a protected page.
2. Enter an email and password. The browser checks required fields and email
   format, then disables repeated submission while the request is pending.
3. The backend validates the request, finds the account and checks the password.
   Failed checks return an error and leave the customer able to correct the form.
4. Success establishes a session: the browser stores the refresh cookie and the
   frontend keeps the access token and public user details in memory.
5. The frontend opens the saved local destination, including its query string,
   when safe, or `/` otherwise. This is the diagram's successful End.

## Understand the session

An access token is a short-lived credential attached to protected API requests.
A refresh token allows the browser to obtain another access token later. The
refresh token is in an HttpOnly cookie, so JavaScript cannot read it. The access
token and user details are not saved in localStorage.

The refresh cookie persists across browser restarts until its token expires,
unless browser settings or the customer clear it. Its maximum age follows the
issued token's remaining validity. The configured refresh lifetime is seven days;
each successful token rotation starts a new seven-day window and updates the
cookie. When the website opens, the frontend uses this cookie to restore login.

The frontend clears any previous local session before a new login attempt. The
backend decides whether credentials are accepted and includes the account's
authentication version in tokens. This version lets password changes invalidate
older sessions. A browser route guard does not replace backend authorization.

## Errors and retries

Wrong credentials show an invalid-email-or-password message. Other request
failures show a general retry message. No successful session is established
from an error response; the customer corrects the form and submits again.

## Implementation references

- [Backend authentication contract](../../../../backend/docs/api/authentication/README.md)
- [Login page](../../../../frontend/src/authentication/pages/LoginPage.tsx)
- [Session provider](../../../../frontend/src/authentication/contexts/Auth/AuthProvider.tsx)

Return to the [authentication index](README.md).
