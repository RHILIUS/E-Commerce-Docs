# JWT interceptor

The interceptor keeps protected requests working when an access token expires.
An interceptor is code that runs around an HTTP request or response. JWT means
JSON Web Token, the credential format used for access and refresh tokens.
**Status: Implemented; the diagram has older details noted below.** Open the
[flowchart](../../diagrams/authentication/jwt-interceptor.drawio.svg) to follow
normal requests, refresh and retry.

## Follow the flow

1. For a normal API request, the frontend adds its current access token as a
   Bearer credential when one is available. Authentication endpoints handle
   their own errors and are excluded from this automatic interception.
2. The backend validates the token and applicable permissions. Valid requests
   continue to their normal operation.
3. An eligible request returning HTTP 401, meaning authentication failed, can
   trigger refresh. It must have a current local token and must not already have
   been retried. An unrelated error does not start refresh.
4. The browser sends the refresh cookie to the refresh endpoint. The backend
   checks the account, token and authentication version. Success rotates the
   refresh token and returns a new access token and public user data.
5. The browser stores the new cookie; the frontend updates its in-memory session
   and retries the original request once with the new access token.

Concurrent refresh callers share one pending refresh request. Session-generation
checks prevent a refresh started before sign-out from restoring local credentials.
Application startup also attempts refresh to restore a session from the cookie.

## Errors and recovery

An invalid refresh response with HTTP 401 clears local credentials. A protected
page then redirects through its route guard; a public page need not navigate.
Other refresh failures propagate to the caller and do not automatically erase
local credentials. The original request is not retried indefinitely.

## Older diagram details

The drawing shows `/api/auth/login/` as a navigation destination. That is the API
operation; the browser's login page is `/login`. Navigation belongs to the route
guard, not the interceptor itself. Access-token expiry is also only one validity
check: the backend checks active accounts and authentication versions. The diagram
does not show the shared refresh request, startup restoration or late-response guard.
Use the code and contract below for those details.

## Implementation references

- [Backend authentication contract](../../../../backend/docs/api/authentication/README.md)
- [Interceptor hook](../../../../frontend/src/authentication/hooks/useAxiosInterceptor.ts)
- [Protected-page navigation](../../../../frontend/src/app/routes/ProtectedRoute.tsx)

Return to the [authentication index](README.md).
