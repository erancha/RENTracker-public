# 0003. Redis on the account's shared database host

- Status: Accepted
- Date: 2025-07-06

## Context

The WebSocket functions need a fast store shared across invocations: which connection belongs to
which user, who is online, and a cache of the reads every connect repeats. The stack used a
managed Redis cluster for it, which bills by the hour whether the app is used or not, and
only functions inside the app's VPC could reach it. The account already ran an EC2 instance
that serves databases to several of the owner's apps, Redis among them.

## Decision

- The stack uses the Redis on the shared database host. The template takes its address
  (`ExistingRedisAddress`) and password (`ExistingRedisPassword`) as parameters and creates no
  Redis of its own.
- Every key the app writes starts with the stack name, so the stacks of different branches and
  apps share the server without colliding.
- The password is a deploy secret, kept in the gitignored `scripts/lib/secrets.sh`.

## Consequences

- No Redis cost belongs to this app alone.
- The app depends on a host another repository manages; when that host is down, connects and
  commands fail.
- The functions reach Redis by the host's public name over the internet, protected by the
  password, so they need no VPC or NAT gateway.
- The password reaches the functions as a plain environment variable.
