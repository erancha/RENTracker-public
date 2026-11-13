# 0004. Google sign-in through a user pool the account already runs

- Status: Accepted
- Date: 2025-04-15

## Context

Landlords and renters are private people who already have Google accounts, and a rental app
asks them to sign in rarely, so a password of the app's own would be one more to forget and one
more to store. The account already ran a Cognito user pool for another of the owner's apps,
with Google configured as its sign-in provider and a hosted sign-in domain. A second pool would
need its own Google OAuth client and domain setup for the same result.

## Decision

- The stack creates no user pool. It adds an app client to the existing pool, named by the
  `ExistingUserPoolId` and `ExistingCognitoDomain` parameters.
- The app client allows only Google, with the authorization code flow, and redirects to the
  frontend's address and to `http://localhost:3000` for local work.
- The REST API checks every request's token with a Cognito authorizer on that pool.
- The browser passes its ID token in the WebSocket connect URL; the connect function decodes it,
  refuses an expired one and takes the user id from it.
- The Cognito user id is the user's id everywhere in the data, and the admin is the one user id
  set in the `AdminUserId` parameter.

## Consequences

- The app holds no passwords and has no sign-up, reset or email-confirmation screens.
- Sign-in depends on a pool another stack owns; moving the app to another account means pointing
  the parameters at a pool there.
- The connect function does not check the token's signature, so the WebSocket API trusts any
  well-formed, unexpired token; only the REST API verifies tokens.
- A user who signs in to another app on the same pool has the same user id here.
