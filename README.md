# Power Fixer Setup Guide

User-facing setup guide for running `power-fixer-server` and `power-fixer` together with remote agent callbacks.

## What you will run

- `power-fixer-server` (local backend)
- `power-fixer` (local TUI client)
- Remote ambient agents launched by the server

## Prerequisites

- Rust (stable)
- PostgreSQL (`psql`, `createdb`)
- GitHub CLI (`gh`) authenticated
- `ngrok` (or equivalent tunnel) for remote callback routing

## Required credentials

- `GITHUB_TOKEN` (for `power-fixer`)
- `WARP_API_KEY` (for `power-fixer-server`)
- `POWERFIXER_ENVIRONMENT_ID` (remote runtime environment)

Optional:
- `POWERFIXER_AGENT_PROFILE_ID`
- `POWERFIXER_DEDUPE_ENVIRONMENT_ID`
- Slack/OpenAI keys

## Important runtime dependency

Include this repository in your ambient runtime environment:

- `warpdotdev/power-fixer-status-update`

Reason: prompt templates reference:

- `/workspace/power-fixer-status-update/powerfixer_status.py`

## 1) Start `power-fixer-server`

```bash
cd /path/to/power-fixer-server
cp .env.example .env
```

Set minimum `.env` values:

```env
DATABASE_URL=postgres://postgres@localhost/powerfixer
WARP_API_KEY=...
POWERFIXER_ENVIRONMENT_ID=...
POWERFIXER_CALLBACK_PORT=3001
POWERFIXER_DEFAULT_GITHUB_ORG=bholmesdev
POWERFIXER_DEFAULT_PROJECT=simplestack-store
RUST_LOG=info
```

For remote agents, set a public callback URL (not localhost):

```env
POWERFIXER_CALLBACK_URL=https://<your-ngrok-domain>.ngrok-free.dev
```

Create DB (first time only):

```bash
createdb powerfixer
```

Start server:

```bash
./script/server
```

Optional health check:

```bash
curl http://localhost:3001/health
```

## 2) Start `power-fixer` client

```bash
cd /path/to/power-fixer
cp .env.example .env
```

Set minimum `.env` values:

```env
GITHUB_TOKEN=ghp_...
POWERFIXER_DEFAULT_REPO=bholmesdev/simplestack-store
POWERFIXER_SERVER_URL=http://localhost:3001
POWERFIXER_LOCAL_AGENT_BIN=oz
```

Start client:

```bash
./script/run --local
```

## 3) Use dedupe with remote agents

When a dedupe agent runs remotely, callback status must go to:

- `POST ${POWERFIXER_CALLBACK_URL}/api/v1/agent/status`
- Header: `Authorization: Bearer ${POWERFIXER_CALLBACK_TOKEN}`

Example callback:

```bash
curl -X POST "${POWERFIXER_CALLBACK_URL}/api/v1/agent/status" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${POWERFIXER_CALLBACK_TOKEN}" \
  -d '{"state":"IN_PROGRESS","summary":"Started dedupe"}'
```

## Quick verification checklist

- [ ] Server starts and `/health` is reachable
- [ ] Client loads issues from `bholmesdev/simplestack-store`
- [ ] Dedupe agent launches
- [ ] Remote callback reaches `/api/v1/agent/status`
- [ ] Dedupe results appear in the client review flow
