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
  browser passes its Cognito ID token in the connect URL. The function reads the user from the
  token and refuses an expired one. It records the connection and its user in Redis for one
  hour, tells the other connected browsers who is online, and decides the user's role
  ([Roles](sign-in.md#roles)). The reads have five seconds; past that the reply says the
  database did not answer.
- `$disconnect` goes to `WebsocketDisconnect` (`disconnect.js`), which removes the connection
  from Redis and sends the remaining browsers the updated list of who is online.
- `CommandsHandlerRouteKey` is the `action` of every command the browser sends, and goes to
  `WebsocketCommandsHandler` (`websocketCommandsHandler.js`). It looks up the sender's user id
  in Redis by connection id, then runs one `create`, `read`, `update` or `delete` command on
  apartments, activity or SaaS tenants through the commands-handlers layer. The reply goes to
  every open connection of the same user.

## REST API

The REST API (`<stack>-RestAPI`) answers JSON, and every route requires a token the shared user
pool issued. `DocumentsRestFunction` (`backend/documents/documents.js`) serves them all: it
creates, lists, reads, updates and deletes rental agreements, stores uploaded attachments in S3,
and answers with presigned URLs, valid for 24 hours, for the PDF and the attachments
([S3](data-model.md#s3)).

- `POST /documents` — creates a rental agreement.
- `GET /documents` — lists the agreements of the apartment named by the `apartmentId` query
  parameter, or of the renter named by `tenantUserId` (400 when neither is given).
- `GET /documents/{documentId}` — one agreement, with presigned URLs of its attachments.
- `PUT /documents/{documentId}` — updates an agreement.
- `DELETE /documents/{documentId}` — deletes an agreement.
- `GET /documents/{documentId}/pdf` — a presigned URL of the agreement's PDF.
- `POST /upload` — stores an attachment of an agreement; `documentId`, `fileName` and
  `fileType` are required query parameters (400 without them).

Each create or update queues an email to the landlord and the first renter, and an update also
queues the PDF's cache invalidation.
