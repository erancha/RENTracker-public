# API endpoints

This document details every route behind the entry points that
[Architecture](architecture.md#the-entry-points) summarizes: who may call it, what it takes and
answers, and its limits. The browser reaches the app through a WebSocket API, for every command
on apartments, activity and SaaS tenants, and through a REST API, for the rental agreements. The
work a route hands off runs in the workers
([Workers' responsibilities and flows](workers.md)).

## WebSocket API

The WebSocket API (`<stack>-WebSocketAPI`) selects the route by the `action` field of each
message.

- `$connect` goes to `WebsocketConnect` (`backend/websockets/connections/connect.js`). The
  browser passes its Cognito ID token in the connect URL. The function verifies the token's
  signature, expiry and client, and refuses the connection otherwise. It claims an invitation
  for the user's email when the user has no landlord record. It records the connection and its
  user in Redis for one hour, tells the other connected browsers who is online, and decides the
  user's role ([Roles](sign-in.md#roles)). The reads have five seconds; past that the reply says
  the database did not answer.
- `$disconnect` goes to `WebsocketDisconnect` (`disconnect.js`), which removes the connection
  from Redis and sends the remaining browsers the updated list of who is online.
- `CommandsHandlerRouteKey` is the `action` of every command the browser sends, and goes to
  `WebsocketCommandsHandler` (`websocketCommandsHandler.js`). It looks up the sender's user id
  in Redis by connection id, then runs one `create`, `read`, `update` or `delete` command on
  apartments, activity or SaaS tenants through the commands-handlers layer. It refuses a write
  of a SaaS tenant record unless the sender is the admin, or a landlord editing the details of
  their own record. The reply goes to every open connection of the same user.

## REST API

The REST API (`<stack>-RestAPI`) answers JSON. `DocumentsRestFunction`
(`backend/documents/documents.js`) serves every route: the landlord's rental agreements, their
attachments and signing links, and the two signing routes a renter calls without signing in
(`backend/documents/signing.js`). Its presigned URLs, of a PDF or an attachment, are valid for
24 hours and served through the documents distribution ([S3](data-model.md#s3)).

**The landlord's routes** require a token the shared user pool issued, and act only on the
caller's own agreements: one of another landlord answers 404.

- `POST /documents` — creates a rental agreement.
- `GET /documents` — lists the agreements of the apartment named by the `apartmentId` query
  parameter (400 without it), archived ones left out.
- `GET /documents/{documentId}` — one agreement, with presigned URLs of its attachments.
- `PUT /documents/{documentId}` — updates an agreement and raises its version, which makes
  every link sent for the previous version stale. Refused once a renter has signed through a
  link, or when the agreement is complete or archived.
- `DELETE /documents/{documentId}` — deletes a draft; archives an agreement anyone signed or
  that was sent for signing, keeping its files and signing events. Answers `{ document_id,
  archived }`.
- `GET /documents/{documentId}/pdf` — a presigned URL of the agreement's PDF: the final copy
  once there is one, the latest rendering otherwise.
- `POST /upload` — stores an attachment or the landlord's signature image; `documentId`,
  `fileName` and `fileType` are required query parameters (400 without them).
- `POST /documents/{documentId}/signing-links` — takes `{ renter }` (1, or 2 when the agreement
  names a second renter) and answers `{ url, phone, expires_at }`. It needs the landlord's
  signature and the renter's phone, and refuses a renter who signed or whose phone is the
  landlord's (400, code `LANDLORD_PHONE`). It freezes the current version's PDF and issues a
  link valid for 2 days, revoking that renter's previous link. 409
  with code `PDF_RENDERING` while the PDF of the latest save is still rendering.
- `DELETE /documents/{documentId}/signing-links/{renter}` — revokes a renter's link.

Each create or update, and each renter's signature, queues an email to the landlord with the
agreement's terms and a signed mark beside every party who signed; an update also queues the
PDF's cache invalidation.

**The signing routes** take no sign-in: the link's token, in the request body, is the
authorization (ADR 0005). A token reads and signs only
the one agreement its link was issued for. API Gateway holds `/signing/state` to 5 requests a
second (bursts of 10) and `/signing/sign` to 2 (bursts of 5). A refusal answers
`{ code, message }`: 404 `LINK_INVALID` for an unknown, revoked or replaced link, or an archived
agreement; 410 `LINK_EXPIRED` after 2 days; 410 `LINK_STALE` when the agreement changed after the
link was sent.

- `POST /signing/state` — takes `{ token }` and answers the agreement's summary (address, dates,
  rent, the landlord's and the renter's names), whether this renter signed and whether the
  agreement is complete, and a presigned URL of the PDF: the frozen copy until the agreement is
  complete, the final copy after. A complete agreement's URL is `null` for the few seconds until
  its final copy is written.
- `POST /signing/sign` — takes `{ token, signatureImage }`, a PNG data URL of at most 512 KB, and
  answers the same state. One transaction marks the link used, records the signature on the
  agreement and adds a signing event, on condition that the link is unused, unexpired and still
  the renter's current one, and that the agreement's version is the one the link was sent for.
  A second request with a used link signs nothing and answers the state, so a double tap records
  one signature.
