# Workers' responsibilities and flows

The workers are `WebsocketEventsHandler` and `DocumentsStreamProcessor`; this document details
the steps, limits and retries of each, behind the entry points that
[Architecture](architecture.md#the-entry-points) summarizes. A worker is a function no request
waits on: a queue message or a table's stream wakes it. The routes that feed them are in
[API endpoints](api.md).

## `WebsocketEventsHandler`

`messagesToClientsQ`, an SQS queue, carries every reply, email request, change event and cache
invalidation to `WebsocketEventsHandler` (`websocketEventsHandler.js`). A message that fails
five times moves to `messagesToClientsDLQ`.

The handler reads the queue in batches of ten. For each message it may push the reply to its
target connections through the WebSocket management API, ask SES to verify a connecting user's
email address
([Email goes only to verified addresses](design-patterns.md#email-goes-only-to-verified-addresses)),
send an email, publish the change to EventBridge, and invalidate one PDF in the documents
CloudFront cache.

`commands-event-bus` is the EventBridge bus that receives a `command-executed` event for every
created, updated or deleted record. When the `TargetChattyLambdaArn` parameter names a function,
a rule forwards to it an enabled apartment's update, a new activity entry waiting for
confirmation, and a new enabled landlord; the deploy leaves the parameter empty, so no rule
exists.

## `DocumentsStreamProcessor`

`DocumentsStreamProcessor` (`documentsStreamProcessor.js`) reads the Documents table's stream. A
new or changed agreement fills the Hebrew Markdown template in `backend/documents/templates/`
with the agreement's fields and both signature images, converts it to HTML with `marked`, prints
it to PDF in headless Chromium and writes it to S3. A deleted agreement deletes its folder in
the bucket.
