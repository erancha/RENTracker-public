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

`DocumentsStreamProcessor` (`documentsStreamProcessor.js`) reads the Documents table's stream in
batches of ten. A new agreement, and a change to what the PDF prints, renders the PDF: an edit
raises `version`, and a renter's link signature adds `signature_1` or `signature_2`. A change that
only records a link, a status or the final copy renders nothing, and an agreement with a final
copy never renders again.

Rendering fills the Hebrew Markdown template in `backend/documents/templates/` with the
agreement's fields and the signature images: the landlord's, then each renter's, read from the
key the renter's signature records (a renter who signed in the app before links existed, from
the folder's `tenantSignature`). The second renter's signature line shows only when the agreement
names a second renter. `marked` converts it to HTML, headless Chromium prints it to PDF, and the
PDF goes to `rental-agreement.pdf` with the version in its metadata; sending a signing link
copies only a PDF whose version matches the agreement's.

When the agreement is `complete`, the same PDF is also written to `final.pdf`, once, with S3's
conditional write, and its key and SHA-256 are recorded on the agreement as `final_pdf`. A
retried record finds the key taken and changes nothing. A record that fails is logged and
skipped. A deleted agreement, which only a draft can be, deletes its folder in the bucket.
