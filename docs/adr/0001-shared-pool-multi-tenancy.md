# 0001. Landlords share one deployment, kept apart by saas_tenant_id

- Status: Accepted
- Date: 2025-05-08

## Context

The stack served one landlord: the landlord's user id was a template parameter, so every new
landlord needed a stack of their own, deployed by the operator. That multiplied the stacks, the
deploys and the idle cost by the number of landlords, while each one held very little data, and
no landlord could start without the operator. A renter may also rent from more than one
landlord, so a renter cannot belong to a single landlord's deployment.

## Decision

- One stack serves every landlord: the same functions, queues and tables for all of them.
- Every item carries `saas_tenant_id`, the landlord's Cognito user id, and the data layer
  checks it on writes; the indexes that list a landlord's data are keyed by it.
- A landlord becomes a SaaS tenant by creating a record in a `SaaSTenants` table from the app;
  the operator, as the admin, can only disable that record.
- Renters are not SaaS tenants. A renter is recorded on each agreement as `tenant_user_id`, so
  one renter can hold agreements with several landlords.

## Consequences

- A new landlord costs nothing to add and is served by the next request; there is nothing to
  deploy.
- Isolation rests on the application code: a missing `saas_tenant_id` condition would expose one
  landlord's data to another, since no AWS boundary separates them.
- One landlord's heavy use shares the account's Lambda and DynamoDB limits with all the others.
- The word "tenant" means a landlord customer in the code and a renter in the product, so the
  code names a renter `tenant_user_id`.
