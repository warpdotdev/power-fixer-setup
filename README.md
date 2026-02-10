# Power Fixer Setup Guide

User-facing setup guide for running `power-fixer-server` and `power-fixer` together with remote agent callbacks.

## What you will run

Power Fixer includes both a server and a client component you will need to clone:

- [`power-fixer-server`](https://github.com/warpdotdev/power-fixer-server) (local backend)
- [`power-fixer`](https://github.com/warpdotdev/power-fixer) (local TUI client)

## Prerequisites

- Rust (stable)
- PostgreSQL (`psql`, `createdb`)
- GitHub CLI (`gh`) authenticated
- `ngrok` (or equivalent tunnel) for remote callback routing

## Required credentials

- `GITHUB_TOKEN` for `power-fixer` to pull issues from GitHub
- `WARP_API_KEY` for `power-fixer-server` to connect to the Oz platform. See our [API key docs](https://docs.warp.dev/reference/cli/cli#generating-api-keys) to learn how to generate this.
- `POWERFIXER_ENVIRONMENT_ID` to run coding agents instead of a configure environment. See the Environment Setup section below to configure this for your repository

Optional:
- `POWERFIXER_AGENT_PROFILE_ID`
- `POWERFIXER_DEDUPE_ENVIRONMENT_ID`
- Slack/OpenAI keys

## Environment Setup

You will need to create an [Oz cloud environment](https://docs.warp.dev/agent-platform/cloud-agents/environments) for coding agents to access your GitHub repository.

When configuring your environment, be sure to include this repository alongside your own codebase: [`warpdotdev/power-fixer-status-update`](https://github.com/warpdotdev/power-fixer-status-update). This includes a runtime script for your cloud agents to report status updates back to Power Fixer as they work.

[See our documentation](https://docs.warp.dev/agent-platform/cloud-agents/environments) for more on cloud environments.

## Run Power Fixer locally

### 1) Start `power-fixer-server`

```bash
cd /path/to/power-fixer-server
cp .env.example .env
```

And set the minimum `.env` values. See [our API key docs](https://docs.warp.dev/reference/cli/cli#generating-api-keys) to learn how to generate a Warp API key to use the Oz platform.

```env
DATABASE_URL=postgres://postgres@localhost/powerfixer
WARP_API_KEY=...
POWERFIXER_ENVIRONMENT_ID=...
POWERFIXER_CALLBACK_PORT=3001
POWERFIXER_DEFAULT_GITHUB_ORG=YOUR_ORG
POWERFIXER_DEFAULT_PROJECT=YOUR_REPO
RUST_LOG=info
```

Then, you'll need to include a "callback URL" for the server to report updates back to your client application. For local development, we suggest creating an [ngrok server](https://ngrok.com/) to expose your local server as a public URL for cloud agents to access.

```env
POWERFIXER_CALLBACK_URL=https://<your-ngrok-domain>.ngrok-free.dev
```

Then, stand up the database and start the server by running `./script/server`.

## 2) Start `power-fixer` client

```bash
cd /path/to/power-fixer
cp .env.example .env
```

And set the minimum `.env` values:

```env
GITHUB_TOKEN=ghp_...
POWERFIXER_DEFAULT_REPO=YOUR_ORG/YOUR_REPO
POWERFIXER_SERVER_URL=http://localhost:3001
```

Then, start the client connected to your local server using the `--local` flag:

```bash
./script/run --local
```

