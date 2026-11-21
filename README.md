# RENTracker

> This repository publishes the documentation of a private codebase; a script or source
> file the documents name is not here.

A rental tracking app for landlords and the people who rent from them. A landlord lists
apartments, fills in a rental agreement, and sends the renter a link; the renter signs in, adds
their details and attachments, and signs on the phone. The app renders the signed agreement as a
PDF, emails both sides, and keeps a log of each apartment's activity, such as payments sent and
accepted. Every landlord runs a space of their own inside one shared deployment, so the app is a
small SaaS rather than one landlord's tool.

**Contents:** [Overview](#overview) · [Tech stack](#tech-stack) · [Architecture](#architecture) ·
[Getting started](#getting-started) · [Scripts](#scripts) · [Details](#details)

## Overview

- **Sign-in and roles**: a user signs in with a Google account and, on the first visit, picks a
  role: landlord or renter. A landlord becomes a customer of the service without help and gets a
  space of their own; in this app a "SaaS tenant" is such a landlord customer, never a person
  renting an apartment.
- **Apartments**: a landlord adds apartments (address, unit, rooms, monthly rent) and can
  disable one.
- **Rental agreements**: a landlord fills in an agreement for an apartment: property and
  included equipment, lease terms, utility limits, the landlord's details, and optional
  guarantees and a guarantor. The form checks Israeli ID and phone numbers and the order of the
  lease dates.
- **Signing**: the landlord shares the agreement's link, for example on WhatsApp. The renter
  who opens it signs in, completes the renter's details, uploads an ID card and pay slips, and
  draws a signature; the landlord then signs too.
- **PDF and email**: every save renders the agreement as a PDF from a Hebrew template. Both
  sides get an email about it, and either can download the PDF. Email reaches only an address
  that confirmed the verification email AWS sends on a user's first sign-in.
- **Activity log and live updates**: each apartment keeps a log, such as "Payment sent for
  05/2025", and an entry can wait for the other side's confirmation. A change a user makes
  reaches that user's other open sessions without a reload.
- **Operator and languages**: the operator signs in as the admin, a Google account fixed when
  the app is deployed, sees every landlord account and can disable one, and never manages a
  landlord's apartments, agreements or renters. The interface follows the browser's language,
  Hebrew or English, and fits a phone screen.

## Tech stack

- **Frontend** — React 18 with TypeScript, Redux and Material UI, built with Create React App,
  served from S3 behind CloudFront
- **Backend** — Node.js 24 Lambdas behind an API Gateway WebSocket API and a REST API, with their
  shared code in Lambda layers
- **Data** — DynamoDB tables, an S3 bucket for agreements and attachments behind its own
  CloudFront distribution, and Redis on the account's shared database host
- **Messaging** — an SQS queue that carries replies to the browsers, an EventBridge bus that
  receives every change for other services to subscribe to, SES for email
- **Documents** — Markdown templates rendered to PDF with `marked` and headless Chromium
  (`puppeteer-core` with `@sparticuz/chromium`)
- **Auth** — Google as the only sign-in, through a Cognito user pool another stack owns
- **Infrastructure** — one AWS SAM template deployed by bash scripts; CloudWatch logs and X-Ray
  tracing

## Architecture

The browser keeps a WebSocket open: each command wakes a Lambda that reads or writes DynamoDB
through a Redis cache, and a second Lambda pushes the reply back to the browsers through an SQS
queue. Agreements go through a REST API, and every change to one starts a Lambda that renders its
PDF into S3. All landlords share the same functions and tables; every item carries its
landlord's user id, and the data layer makes each write conditional on it.

```mermaid
graph LR
    B((Browser app)) -->|Google sign-in| COG[Cognito]
    B -->|commands| WS[WebSocket API]
    B -->|agreements| REST[REST API]
    WS --> CMD[Command Lambdas]
    CMD --> DDB[(DynamoDB)]
    CMD --> R[("Redis on the shared<br/>database host")]
    CMD --> Q[SQS queue]
    Q --> EV[Events Lambda]
    EV -->|push replies| WS
    REST --> DOC[Documents Lambda]
    DOC --> DDB
    DDB -->|stream| PDF[PDF Lambda]
    PDF --> S3[(S3 documents)]
    B --> CF[CloudFront]
    CF --> FB[(S3 app files)]
```

**[Architecture](docs/architecture.md)** continues from here: the full diagram, every function,
queue and table, the storage and the codebase layout.

## Getting started

You need bash, git, the AWS CLI v2, the AWS SAM CLI, `jq` and Node.js with `npm`.
The stack deploys into an AWS account that already runs the user pool and the database host
of the vpc-subnets-sgs repository ([Setting up Google](docs/sign-in.md#setting-up-google)).

1. `./scripts/init-env.sh` creates the gitignored settings files under `scripts/lib/`; fill
   them in.
2. `./scripts/test.sh` runs every check, with no AWS access.
3. `./scripts/deploy.sh --frontend` deploys the stack and the frontend and prints its address;
   `main` deploys `rntrk`, any other branch `rntrk-fb`.
4. `npm install && npm start` in `frontend/` serves `http://localhost:3000` against the
   branch's stack.

[Development & deployment](docs/development.md) lists every prerequisite and step.

## Scripts

- `init-env.sh` — creates the gitignored settings files under `scripts/lib/` from their
  `.example.sh` files
- `test.sh` — the quality gate: `bash -n` on every script and the SAM lint check
- `deploy.sh` — lints, builds and deploys the branch's stack; `--frontend` also publishes the
  React app, `--frontend-only` only that, `--sync` runs `sam sync` on a feature branch's stack
- `aws-helper.sh` — every other AWS operation on the stack: function list and logs, log
  retention, X-Ray, stack deletion, the region's leftover resources and unused layer versions

Each script's header comment is its full documentation, and `--help` prints it.

## Details

- [Architecture](docs/architecture.md) — what each part is for, the storage and the codebase
- [API endpoints](docs/api.md) — every route, who may call it, what it takes and its limits
- [Workers' responsibilities and flows](docs/workers.md) — what each background function does,
  with its limits and retries
- [Design patterns](docs/design-patterns.md) — the shared layers, the reply queue, cache and
  settings
- [Data model](docs/data-model.md) — every table, index and bucket layout, and the Redis keys
- [Sign-in](docs/sign-in.md) — how Google sign-in reaches the APIs through the shared pool, its
  setup in a new account, and how a user's role is decided
- [Development & deployment](docs/development.md) — the settings files, the tests, the
  scripts, what the stack expects in the account, and the layers
