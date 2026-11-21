# Design patterns

This document details the mechanisms that cut across the entry points
[Architecture](architecture.md) lists: where the shared code lives, how a reply reaches the
browser, how a setting reaches a function and the browser, where the functions run and what
wakes them; then what is cached and who receives email.

**Contents:** [Shared code lives in layers](#shared-code-lives-in-layers) ·
[Replies go through a queue](#replies-go-through-a-queue) ·
[Settings come from the template and the stack's outputs](#settings-come-from-the-template-and-the-stacks-outputs) ·
[The functions run outside a VPC](#the-functions-run-outside-a-vpc) ·
[Nothing runs on a clock](#nothing-runs-on-a-clock) ·
[Query results are cached in Redis](#query-results-are-cached-in-redis) ·
[Email goes only to verified addresses](#email-goes-only-to-verified-addresses)

## Shared code lives in layers

The commands handlers, the data layer, the Redis client, the connections helpers and the
document key helpers are Lambda layers the functions import from `/opt`. The deploy pins each
layer to a version published once, so a change to a layer's source reaches the functions only
through a new layer version (see [Development & deployment](development.md#layers)). The data
layer imports from `/opt` inside Lambda and from `/usr/src/app/layers` elsewhere, so the
container service under `backend/RenTracker-service/ecs/` runs the same code.

## Replies go through a queue

The functions behind the WebSocket API are deployed without the API's management address, so
they never push to a browser themselves; every reply goes through the `messagesToClientsQ` queue
to `WebsocketEventsHandler`, the events handler
([Workers' responsibilities and flows](workers.md#websocketeventshandler)). Email, EventBridge
and CloudFront calls happen there too, out of the browser's request.

## Settings come from the template and the stack's outputs

The functions and the browser each read their settings from one place. `scripts/lib/stack-params.sh`
passes every parameter the deploy controls, and the template hands them to the functions as
environment variables. The frontend reads `frontend/src/appConfig.json`, which
`scripts/lib/app-config.sh` writes from the stack's outputs.

## The functions run outside a VPC

The template can place the functions in a private subnet with a security group, but the deploy
leaves both parameters empty, so the functions run outside any VPC and reach Redis and the AWS
services over the internet.

## Nothing runs on a clock

No schedule wakes any function of the stack: every function starts from a request, a queue
message or a table stream.

## Query results are cached in Redis

The data layer (`dbData.js`) reads the landlord's apartments, an apartment's activity, an
apartment's agreements and a user's SaaS tenant record through Redis. A miss reads DynamoDB and
stores the result for 24 hours under a key prefixed by the stack name; each write through the
data layer deletes the keys it makes stale. The `DisableAppCache` parameter turns the cache off.

## Email goes only to verified addresses

The events handler sends from one fixed sender address set in the template, and the SES sandbox
sends only to verified addresses. On each WebSocket connect, the message `WebsocketConnect`
queues names the user's address, and the events handler asks SES to verify an address SES does
not know yet; for a known address it reads whether SES has verified it. The message it then
pushes to the browser says when verification is pending.
