# Power Fixer Setup Guide

User-facing setup guide for running `power-fixer-server` and `power-fixer` together with remote agent callbacks.

## What is Power Fixer?

Power Fixer is a GitHub issue triage toolkit made of two components:

- [`power-fixer`](https://github.com/warpdotdev/power-fixer): a keyboard-driven terminal UI for high-velocity triage, issue routing, duplicate detection, and launching Oz cloud agents.
- [`power-fixer-server`](https://github.com/warpdotdev/power-fixer-server): the backend API and state manager that tracks agent runs, receives callbacks, and synchronizes state to clients.

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

## Basic GCloud Deployment Guide

This section covers a straightforward production deployment model on Google Cloud using Cloud Run for the server and a local/client binary for `power-fixer`.

### 1) Create required GCP resources

- A Google Cloud project
- Artifact Registry repository for container images
- Cloud SQL Postgres instance (or another reachable Postgres database)
- Secret Manager secrets for runtime credentials (at minimum `WARP_API_KEY`)

### 2) Build and deploy `power-fixer-server` to Cloud Run

From the `power-fixer-server` repo:

```bash
# Build and push image
gcloud builds submit --tag us-central1-docker.pkg.dev/PROJECT_ID/power-fixer/server

# Deploy service
gcloud run deploy power-fixer-server \
  --image us-central1-docker.pkg.dev/PROJECT_ID/power-fixer/server \
  --region us-central1 \
  --platform managed \
  --set-env-vars DATABASE_URL=postgres://... \
  --set-env-vars POWERFIXER_CALLBACK_URL=https://YOUR_PUBLIC_SERVER_URL \
  --set-env-vars POWERFIXER_ENVIRONMENT_ID=... \
  --set-env-vars POWERFIXER_DEFAULT_GITHUB_ORG=YOUR_ORG \
  --set-env-vars POWERFIXER_DEFAULT_PROJECT=YOUR_REPO \
  --set-secrets WARP_API_KEY=powerfixer-warp-api-key:latest
```

Notes:
- `POWERFIXER_CALLBACK_URL` must be publicly reachable by cloud agents.
- You can pass `WARP_API_KEY` via env vars instead of secrets, but Secret Manager is recommended.

### 3) Configure and run `power-fixer` client

In your client environment:

```env
GITHUB_TOKEN=ghp_...
POWERFIXER_DEFAULT_REPO=YOUR_ORG/YOUR_REPO
POWERFIXER_SERVER_URL=https://YOUR_PUBLIC_SERVER_URL
```

Then run:

```bash
power-fixer
```

### 4) Validate end-to-end

- Confirm server health:
  - `GET https://YOUR_PUBLIC_SERVER_URL/health`
- Launch a cloud agent from the client and confirm:
  - status progresses from queued/in-progress to terminal state
  - callbacks are visible in the server logs and UI state
