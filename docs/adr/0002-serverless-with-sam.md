# 0002. Serverless on AWS, declared in one SAM template

- Status: Accepted
- Date: 2025-04-15

## Context

The app's load comes in short bursts: a landlord fills in an agreement, a renter signs it, a
payment is recorded. Between them nothing happens for days, so a server that runs all month
would be paid for while idle. The browser needs live updates, so the backend must push messages
to open sessions, and agreements must become PDFs, which needs a headless browser for seconds at
a time.

## Decision

- Node.js Lambdas behind an API Gateway WebSocket API for commands and live updates, and a REST
  API for agreements and uploads.
- DynamoDB tables billed per request; S3 for the agreements' files and the browser app, each
  behind CloudFront.
- Replies to the browsers go through an SQS queue to one function that pushes them, sends email
  and publishes change events.
- PDFs are rendered inside a Lambda, with a Chromium build made for Lambda.
- Code the functions share lives in Lambda layers.
- One AWS SAM template declares all of it, and SAM builds and deploys it.

## Consequences

- An idle month costs only storage and the shared resources the stack uses.
- The WebSocket connections live in API Gateway, so the functions keep the map of connections to
  users in a store of their own (ADR 0003).
- A slow side effect, such as an email or a cache invalidation, never delays the browser's
  request, but it runs after the reply and fails apart from it.
- Rendering a PDF needs far more memory and time than a command, so it runs in functions sized
  apart from the command functions.
