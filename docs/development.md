# Development & deployment

This document details the local setup, the tests and the operations scripts behind the
[README](../README.md)'s Getting started and Scripts sections; [Architecture](architecture.md)
describes what the scripts deploy. Each script's header comment is its full documentation, and
`--help` prints it.

**Contents:** [Working copy location](#working-copy-location) · [Frontend](#frontend) ·
[Configuration](#configuration) · [Backend tests](#backend-tests) ·
[Deployment scripts (scripts/)](#deployment-scripts-scripts) ·
[What must exist outside the stack](#what-must-exist-outside-the-stack) · [Layers](#layers) ·
[Logs](#logs)

## Working copy location

Keep the checkout on the WSL ext4 filesystem (`~/projects/serverless/RENTracker`) rather than
under `/mnt/c`. WSL reaches Windows drives through a file-sharing protocol that makes every file
open a round trip to the Windows side, and `npm ci` and `sam build` open thousands of files.

`node_modules` folders record absolute paths, so each checkout installs its own instead of
receiving a copy.

## Frontend

```bash
./scripts/lib/app-config.sh     # on a fresh checkout: point appConfig.json at the branch's stack
cd frontend && npm install && npm start
```

The dev server listens on `http://localhost:3000`, a callback the stack's Cognito app client
allows, and calls the deployed backend named in `frontend/src/appConfig.json`, which
`lib/app-config.sh` writes from the stack's outputs and every deploy rewrites.
`frontend/src/appConfig.example.json` shows the file's shape. A bare `npm run build` deploys
nothing; the production frontend ships only through `deploy.sh --frontend`.

`npm test` runs the Create React App test runner. `frontend/.npmrc` sets `legacy-peer-deps`,
because `react-scripts` 5 pins TypeScript 4 while `i18next` declares an optional peer on
TypeScript 5.

## Configuration

`./scripts/init-env.sh` creates the gitignored settings files under `scripts/lib/` from their
`.example.sh` files; each setting's comment says what it holds and whether it is required.

- `aws-config.sh` — the access key pair and region of the target account.
- `secrets.sh` — the Redis password the stack is deployed with.
- `params.sh` — the address every mail is sent from.

Every other setting is committed in `scripts/lib/stack-params.sh`, which lists every parameter
the stack is deployed with: `sam deploy` keeps a stack's previous value for any parameter the
overrides leave out, so a parameter changes only through that file. The app's name (`rntrk`),
its region and the log retention (14 days) are in `scripts/lib/app-env.sh`. An edit to any of
them takes a `deploy.sh` run.

A bare `aws` command in your own shell has no credentials until it sources
`scripts/lib/aws-config.sh`; the scripts source it for you. A script that loads a settings file
first stops if git would track the file or a tracked file holds one of its secrets or
addresses. It also sets `core.hooksPath` to `.githooks`, whose `pre-commit` hook makes the same
check on every commit's staged changes.

## Backend tests

`scripts/test.sh` is the whole quality gate: `bash -n` on every script
and the pre-commit hook, then `sam validate --lint` on the template. It needs `sam` and no AWS
access. The backend has no automated test suite.

## Deployment scripts (`scripts/`)

Install the AWS CLI v2, the AWS SAM CLI, Node.js 24 with npm, `jq` and git, each on `PATH`;
`deploy.sh` stops naming the first one missing. The functions run on the `nodejs24.x` runtime,
and `sam build` installs each function's dependencies with the local `npm`.

No script takes an environment argument. The git branch selects the stack every script acts on:
`main` targets `rntrk` (stage `prod`), whose tables outlive the stack, any other branch
`rntrk-fb` (stage `dev`); an exported `STACK_NAME` wins.

- `deploy.sh` — validates and lints `scripts/template.yaml`, runs
  `sam build --cached --parallel`, which rebuilds only the functions whose source changed, then
  `sam deploy` with the parameters of `stack-params.sh`. `--frontend` also builds the React
  app, syncs it to the frontend bucket and invalidates the CloudFront distribution;
  `--frontend-only` does only that; `--sync` runs `sam sync` and is refused on `main`, since it
  deploys without a change set to review. Every run rewrites `frontend/src/appConfig.json`
  through `lib/app-config.sh`; its `BUILD` value names the
  stack and the time of the last backend deploy, and a frontend publish appends its own time.
- `aws-helper.sh` — AWS operations on the branch's stack, the same
  file in every AWS repo: function logs, log retention, `--delete-stack [--empty]`,
  `--list-resources`, `--xray on|off` and `--prune-layers`. `--delete-stack` stops while a
  bucket of the stack holds objects, unless `--empty` is passed with it.
- `test.sh` — the quality gate, above.
- `tools/` — hands-on tools: `publish-docs.sh` (publishes the
  README and `docs/` to the `RENTracker-public` repository).

`scripts/lib/app-env.sh` holds what the scripts share: the `--help` printer, the settings-file
loaders, which refuse a file git tracks, and the branch's stack selection. It also sets
`SAM_CLI_TELEMETRY=0`, because SAM CLI 1.165 crashes in its telemetry after a successful
command. `sam` writes its build folder `.aws-sam/` into `scripts/`; it is gitignored.

## What must exist outside the stack

The template deploys only the app's own resources. A deploy depends on parts no script of this
repo creates, which `stack-params.sh` names:

- The shared stack `vsdb-cognito`: the user pool with Google sign-in and its domain, an
  authorized redirect of the Google OAuth client. The stack adds its own app client to that
  pool; [Sign-in](sign-in.md#setting-up-google) covers the pool in a new account.
- The Lambda layers `AwsSdkV3Layer` and `CorsHeadersLayer` of the `vsdb-lambda-layers` stack
  and `sum-WebsocketsConnectionsLayer` of the `sum` stack, each pinned to a version number.
- Redis on the account's shared database host, run by the vpc-subnets-sgs repository, reached
  by the host's public name with the password of `secrets.sh`
  (ADR 0003).
- `SES_SENDER` of `params.sh` as an identity SES has verified, since every mail goes out from
  it.

## Layers

A change to a layer's source does not deploy by itself. `stack-params.sh` pins every layer to
a published version, the `rntrk-*` layers built from this repository included, so the stack
builds none of them. A change under `backend/RenTracker-service/ecs/layers/`,
`backend/websockets/layers/` or `backend/layers/document-utils/` reaches the functions only
through a new layer version and a new pin. Passing a layer's parameter empty makes the stack
publish that layer from its folder.

The container under `backend/RenTracker-service/ecs/` runs the same layers as a WebSocket
server of its own; the template does not deploy it. Its image builds with
`docker build -t rent-tracker-service:latest backend/RenTracker-service/ecs/`.

## Logs

Each function writes to `/aws/lambda/<stack>/<logical id>`, a log group the template declares
and keeps for 14 days, so a deploy leaves the logs alone; `aws-helper.sh --delete-stack`
deletes them with the stack.
`./scripts/aws-helper.sh --logs <function> --since 1d --errors` shows one function's warnings
and errors; `--trim-logs` deletes the stack's log streams on request.
