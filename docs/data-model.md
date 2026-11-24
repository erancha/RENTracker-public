# Data model

This document details the stores that [Architecture](architecture.md#storage) summarizes: every
DynamoDB table with its key, indexes and attributes, the layout of the S3 buckets and what Redis
holds. Every item carries its landlord's `saas_tenant_id`
([All landlords share one deployment](architecture.md#all-landlords-share-one-deployment)).

## DynamoDB

The tables bill per request and carry the stack name in their table name. On the `main`
stack they are kept when the stack is deleted (`RetainResourcesOnDelete`); on `rntrk-fb` they
are deleted with it. Point-in-time recovery is on for every table except `SaaSTenants` and
`SigningLinks`.

- `SaaSTenants` — key `saas_tenant_id`, the landlord's user id, or `invitation#<email>` for an
  invitation the landlord has not yet claimed; no index. Other attributes: `name`, `email`,
  `phone`, `address`, `israeli_id`, `is_disabled`, `created_at`, `updated_at`.
- `Apartments` — key `apartment_id`; index `SaaSTenantUpdatedIndex` (`saas_tenant_id`,
  `updated_at`). Other attributes: `saas_tenant_id`, `address`, `is_housing_unit`,
  `unit_number`, `rooms_count`, `rent_amount`, `is_disabled`, `created_at`, `updated_at`.
- `Documents` — key `document_id`; index `ApartmentUpdatedIndex` (`apartment_id`,
  `updated_at`). Other attributes: `saas_tenant_id`, `apartment_id`, `template_name`,
  `template_fields`, `version`, `status`, `link_1` and `link_2`, `signature_1` and
  `signature_2`, `frozen_pdf`, `final_pdf`, `archived_at`, `created_at`, `updated_at`. Items
  written before renters signed through links may carry `tenant_user_id`, which nothing reads.
- `SigningLinks` — key `token_hash`, the SHA-256 of a link's token; no index. Other attributes:
  `document_id`, `renter`, `version`, `phone`, `created_at`, `used_at`, and `expires_at` in epoch
  seconds, which DynamoDB's TTL removes the item after.
- `SigningEvents` — key `document_id` and `event_key` (`<ISO time>#<uuid>`); no index. Other
  attributes, for a link signature (`type` `renter_signed_link`): `renter`, `phone`, `source_ip`,
  `user_agent`, `version`, `frozen_pdf_key`, `frozen_pdf_sha256`, `signature_image_key`,
  `signature_image_sha256`, `completes_agreement`, `created_at`. A paper import (`type`
  `paper_signed_uploaded_by_landlord`) carries `final_pdf_key`, `final_pdf_sha256` and
  `created_at`. The functions' roles grant no update or delete on it, and every put is conditioned
  on its key being new, so an event once written stays as it is.
- `ApartmentActivity` — key `activity_id`; index `ApartmentCreatedIndex` (`apartment_id`,
  `created_at`). Other attributes: `saas_tenant_id`, `apartment_id`, `description`,
  `pending_confirmation`, `created_at`.

These attributes carry a rule:

- `is_disabled` on a SaaS tenant record turns a landlord account off; the admin sets it.
- `template_fields` holds every field of the agreement form, including the signatures' names and
  the attachments' file names. The PDF's address is not stored; `DocumentsRestFunction`, the
  REST function, signs one on each request.
- `pending_confirmation`, when true, marks an activity entry that waits for confirmation.
- `version` on an agreement rises by one on every update; an item without one is version 0.
  A link signs only the version it was sent for.
- `status` on an agreement is absent on a draft, then `sent`, `complete` once every renter
  signed, or `archived`. An agreement has a second renter when `template_fields.tenant2Name` is
  filled.
- `link_<n>` holds renter n's current link (`token_hash`, `phone`, `version`, `sent_at`,
  `expires_at`); `signature_<n>` holds a link signature (`signed_at`, `image_key`, `event_key`),
  and the signing time also goes in the template field the PDF prints
  (`tenantSignature`, `tenant2Signature`).
- `frozen_pdf` (`version`, `key`, `sha256`) is the PDF the links of that version were sent for;
  `final_pdf` (`key`, `sha256`, `written_at`) is the copy signed by every side, or a paper scan.

## S3

The stack has two buckets, one for the agreements' files and one for the built app.

In the **documents bucket** an agreement's files live under `<saas_tenant_id>/<document_id>/`:
`rental-agreement.pdf`, rendered again on each change and stamped with the version it shows
(object metadata `version`); `versions/v<n>.pdf`, the frozen copy a link was sent for;
`signatures/renter<n>-<token hash prefix>.png`, a renter's signature; `final.pdf`, the copy
signed by every side; the landlord's signature image and the attachments. The frozen copies,
the renters' signatures and the final copy are written with S3's conditional write
(`If-None-Match: *`), so none is ever overwritten. The REST function signs an S3
URL valid for 24 hours and replaces its host with that of the bucket's own CloudFront
distribution, so the file is served through CloudFront. The distribution's cache key ignores the
query string, so all signed URLs of one file share one cached copy, while CloudFront still
forwards the signature to S3.
Objects move to the Infrequent Access storage class after 30 days.

The **frontend bucket** holds the built app. The documents bucket is kept like the tables when
the `main` stack is deleted; the frontend bucket is deleted with the stack, and
`aws-helper.sh --delete-stack` stops first while a bucket holds objects.

## Redis

Redis runs on the account's shared database host, outside this stack. The template takes its
address and password as parameters, and every function that needs it connects to the host's
public name with the password (ADR 0003).

It holds, under keys prefixed by the stack name, the cached reads
([Query results are cached in Redis](design-patterns.md#query-results-are-cached-in-redis)) and
the connection records: `conn:<connection id>` with the user's id, name and email, the set
`active_connections`, and `user:<user id>:connections`.
