# Logout

Logout ends the customer's local signed-in experience immediately and asks the
backend to clean up the browser's refresh credential. **Status: Implemented.**
The [flowchart](../../diagrams/authentication/logout.drawio.svg) separates End
Local Session, Revoke Refresh Token and Finish Logout.

## Follow the flow

1. Select Logout from a signed-in page. Repeated clicks are disabled while pending.
2. The frontend clears the access token and user details immediately. It also
   makes any refresh response from the old session unable to restore that session.
3. Independently, the browser sends a logout request with its refresh cookie when
   present. The backend attempts to blacklist that token, meaning it cannot be
   used to request another access token, and returns a cookie-deletion response.
4. The browser applies cookie deletion. A missing or invalid token does not block
   the backend's logout response.
5. A protected page redirects to `/login` without retaining a return destination.
   A public page stays open. Navigation reacts to local state without waiting
   for the backend request to finish.

The diagram has separate End points for local navigation and API cleanup because
they complete independently.

## Limits and failure recovery

If the network request fails, local credentials stay cleared. The cookie might
remain because the browser did not receive its deletion response; the frontend
does not automatically retry logout. Ending the local session prevents a late
refresh from putting credentials back into memory during that session lifecycle.

Ordinary logout revokes only the supplied refresh token. It does not increment
the account's authentication version or revoke every issued access token. Global
session invalidation after a password change is a separate operation.

## Implementation references

- [Backend authentication contract](../../../../backend/docs/api/authentication/README.md)
- [Session provider](../../../../frontend/src/authentication/contexts/Auth/AuthProvider.tsx)
- [Protected-page navigation](../../../../frontend/src/app/routes/ProtectedRoute.tsx)

Return to the [authentication index](README.md).
