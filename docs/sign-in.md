# Sign-in

Landlords and the admin sign in to RENTracker with a Google account, and there is no other way
in. Renters never sign in: they sign through the link the landlord sends
(ADR 0005), on routes that take no token
([API endpoints](api.md#rest-api)). This page details the Cognito entry of
[Architecture](architecture.md#the-entry-points): how a sign-in reaches the APIs, what the
stack adds to the user pool it signs in through, how to set it up in a new AWS account, and how
a user's role is decided. The pool is not this repo's: the `vsdb-cognito` stack of
the vpc-subnets-sgs repo owns it, together with its Google client, and several apps of the
account sign in through it (ADR 0004).

**Contents:** [How it works](#how-it-works) · [Setting up Google](#setting-up-google) ·
[Who may sign in](#who-may-sign-in) · [Roles](#roles)

```mermaid
sequenceDiagram
    participant B as Browser app
    participant C as Cognito hosted domain
    participant G as Google
    participant A as REST and WebSocket APIs
    B->>C: /oauth2/authorize with the app client id
    C->>G: Google sign-in
    G->>C: /oauth2/idpresponse
    C->>B: back to the app address with an authorization code
    B->>C: /oauth2/token exchanges the code for tokens
    B->>A: REST request, Authorization: Bearer ID token
    B->>A: WebSocket connect, ?token=ID token
```

## How it works

1. The browser app signs in with `react-oidc-context` (`frontend/src/index.tsx`), configured
   from `frontend/src/appConfig.json`: the pool, the stack's app client and the redirect URL.
   The landing page's Google button sends the browser to the pool's hosted domain, which offers
   Google only; Google returns to the pool, and the pool returns the browser to the redirect URL
   with an authorization code, which the app exchanges for tokens. The app keeps the ID token,
   whose `sub` claim is the user id everywhere in the data.
2. The REST API's Cognito authorizer checks the ID token on every agreement request.
3. The WebSocket connect URL carries the ID token in its `token` parameter. The WebSocket API
   has no authorizer, so the connect function verifies the token itself (`aws-jwt-verify`): its
   signature against the pool's published keys, its expiry, and that the app's client issued
   it. It refuses the connection otherwise.
4. Sign-out drops the tokens and sends the browser to the pool's `/logout` with the app client id
   and the sign-out URL, which ends the session at the hosted domain too.

The stack's app client, `UserPoolClient` in `scripts/template.yaml`, is all this repo adds to the
pool: the authorization code flow, Google as its only identity provider, and these two URLs as
both its callback and its sign-out URLs:

- `https://<distribution>.cloudfront.net`, the frontend distribution's address and the stack's
  `CloudFrontUrl` output (the custom domain instead, when the `CustomDomainName` and
  `CertificateArn` parameters are both set);
- `http://localhost:3000`, for `npm start` in `frontend/`.

Both are without a trailing slash. `deploy.sh` writes the matching one into `appConfig.json`: the
CloudFront address while it builds the published app, `http://localhost:3000` when it finishes,
so a local `npm start` signs in against the branch's stack. The CloudFront address is written
even when a custom domain is set.

## Setting up Google

A fresh account, with no Google Cloud project and no OAuth client yet, starts in the
vpc-subnets-sgs repo, before any step of this one: the Setting up Google section of
vpc-subnets-sgs's docs/sign-in.md walks from no Google project at all to a deployed shared pool.
The Google side belongs to that pool. Its OAuth client redirects only to the pool's hosted
domain, `https://vsdb-<account id>.auth.<region>.amazoncognito.com/oauth2/idpresponse`, never
to this app, so this repo holds no Google client id or secret, and nothing of this app is
entered in the Google Cloud console.

1. **Check for the pool.** Run `./scripts/aws-helper.sh --list-resources`; a deployed shared
   pool shows as `vsdb-cognito-UserPool` with its id and its `vsdb-<account id>` domain. The pool
   must be in this app's region, the region of `scripts/lib/aws-config.sh`, since the authorizer
   names the pool in the stack's own account and region. With the pool there, go to step 3.
2. **Without the pool, set up Google and the pool first.** Follow the Setting up Google section
   of vpc-subnets-sgs's docs/sign-in.md from its first step. It creates the Google Cloud project,
   its consent screen and OAuth client, stores the client id and secret in the Secrets Manager
   secret `/vsdb-cognito/google-secrets`, and deploys the pool with `./scripts/deploy.sh
   --cognito` from that repo's `main`, so the stack is `vsdb-cognito` and the domain prefix
   `vsdb`. Then run step 1 again for the pool's id.
3. **Name the pool.** In `stack_parameter_overrides` of `scripts/lib/stack-params.sh`, set
   `ExistingUserPoolId` to the pool's id from step 1. `ExistingCognitoDomain` derives itself as
   `vsdb-<account id>.auth.<region>.amazoncognito.com` and needs no edit for the shared pool. A
   different pool works too when it has a hosted domain, which then goes into
   `ExistingCognitoDomain`, and a Google identity provider named exactly `Google`.
4. **Deploy.** `./scripts/deploy.sh --frontend` creates the app client with the URLs above and
   prints the CloudFront address; sign in there or at `http://localhost:3000`.
5. **Make yourself the admin.** The admin is the user whose id equals the `AdminUserId`
   parameter, which `stack_parameter_overrides` sets to a user of the original account's pool,
   so a new pool has no admin. Sign in once, then in the AWS console open **Amazon Cognito →
   User pools → vsdb-cognito-UserPool → Users**, open your user (named `google_<number>`) and
   copy its `sub` attribute. Set `"AdminUserId=<sub>"` in `stack_parameter_overrides` and deploy
   again.

## Who may sign in

Google decides first: while the shared Google client's consent screen is in testing, only the
test users listed under **Audience → Test users** of that Google Cloud project get through, as
the vpc-subnets-sgs page describes. The shared pool runs no function before sign-up, so every
account Google lets through becomes a pool user on its first sign-in, and the app has no
allow-list of its own. What a user may do follows from the user's role ([Roles](#roles)). A
person who signs in to another app on the same pool has the same user id here.

The signed-out landing page (`frontend/src/components/Landing.tsx`) is one card: the product
name, a tagline and the running build, a short pitch in three steps, a toggle that adds the
details, and the Google button, with links under the card to invite a friend on WhatsApp and to
the public copy of the source. The language switch sits at the page's bottom; the app's top bar
and menu show only after sign-in. Its stylesheet, `frontend/src/styles/landing.css`, is an exact
copy of loan-simulator's, which the author's other apps copy too, so the page reads the same in
all of them.

Under the Google button, the toggle "Why sign in with Google?"
(`frontend/src/components/SignInReasons.tsx`) explains signing in with Google to a visitor wary
of it. It opens a panel, in English or Hebrew, saying that Google approves the sign-in and
gives the app only a confirmation of who the person is, that the person can disconnect the app in
the Google account's settings, and that the app stores no passwords. It ends with a link to
Google's own help page on Sign in with Google, and a ✕ closes it.

## Roles

A user who signs in is the admin, a landlord or neither, and `WebsocketConnect` decides which on
every connect: the admin when the user id equals the `AdminUserId` parameter, a landlord when a
SaaS tenant record exists for the user and is not disabled. Anyone else, a renter who signed in
for instance, sees only a page saying that renters sign through the link their landlord sends.
A landlord's first reply carries the apartments and the first apartment's activity.

The admin adds a landlord by inviting an email, with a name, on the landlord-records screen. The
invitation is a record keyed by that email. When a user with no record connects, the connect
function looks for an invitation naming the email in the user's verified token, and in one
transaction moves it to the user's id, so the user is a landlord from that connect on; two
connects at once claim it once. The email in the token is the Google account's: the app client
grants no scope that lets a user change it. Only the admin creates, deletes, enables or disables
a landlord record; a landlord may edit the details of their own.
