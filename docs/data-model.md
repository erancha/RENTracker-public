# Data model

This document details the stores that [Architecture](architecture.md#storage) summarizes: every
DynamoDB table with its key, indexes and attributes, the layout of the S3 buckets and what Redis
holds. Every item carries its landlord's `saas_tenant_id`
([All landlords share one deployment](architecture.md#all-landlords-share-one-deployment)).

## DynamoDB

The tables bill per request and carry the stack name in their table name. On the `main`
stack they are kept when the stack is deleted (`RetainResourcesOnDelete`); on `rntrk-fb` they
are deleted with it. Point-in-time recovery is on for every table except `SaaSTenants`.

- `SaaSTenants` — key `saas_tenant_id`; no index. Other attributes: `name`, `email`, `phone`,
  `address`, `israeli_id`, `is_disabled`, `created_at`, `updated_at`.
- `Apartments` — key `apartment_id`; index `SaaSTenantUpdatedIndex` (`saas_tenant_id`,
  `updated_at`). Other attributes: `saas_tenant_id`, `address`, `is_housing_unit`,
  `unit_number`, `rooms_count`, `rent_amount`, `is_disabled`, `created_at`, `updated_at`.
- `Documents` — key `document_id`; indexes `ApartmentUpdatedIndex` (`apartment_id`,
  `updated_at`) and `TenantUpdatedIndex` (`tenant_user_id`, `updated_at`). Other attributes:
  `saas_tenant_id`, `apartment_id`, `tenant_user_id`, `template_name`, `template_fields`,
  `created_at`, `updated_at`.
- `ApartmentActivity` — key `activity_id`; index `ApartmentCreatedIndex` (`apartment_id`,
  `created_at`). Other attributes: `saas_tenant_id`, `apartment_id`, `description`,
  `pending_confirmation`, `created_at`.

Three attributes carry a rule:

- `is_disabled` on a SaaS tenant record turns a landlord account off; the admin sets it.
- `template_fields` holds every field of the agreement form, including the signatures' names and
  the attachments' file names. The PDF's address is not stored; `DocumentsRestFunction`, the
  REST function, signs one on each request.
- `pending_confirmation`, when true, marks an activity entry that waits for the other side.

## S3

The stack has two buckets, one for the agreements' files and one for the built app.

In the **documents bucket** an agreement's files live under `<saas_tenant_id>/<document_id>/`:
`rental-agreement.pdf`, the signature images and the attachments. The REST function signs an S3
URL valid for 24 hours and replaces its host with that of the bucket's own CloudFront
distribution, so the file is served through CloudFront. The distribution's cache key ignores the
query string, so all signed URLs of one file share one cached copy, while CloudFront still
forwards the signature to S3.
Objects move to the Infrequent Access storage class after 30 days.

The **frontend bucket** holds the built app. Both buckets are deleted with the stack, and
`aws-helper.sh --delete-stack` stops first while either holds objects.

## Redis

Redis runs on the account's shared database host, outside this stack. The template takes its
address and password as parameters, and every function that needs it connects to the host's
public name with the password (ADR 0003).

It holds, under keys prefixed by the stack name, the cached reads
([Query results are cached in Redis](design-patterns.md#query-results-are-cached-in-redis)) and
the connection records: `conn:<connection id>` with the user's id, name and email, the set
`active_connections`, and `user:<user id>:connections`.
