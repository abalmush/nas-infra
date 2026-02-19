# NAS Infrastructure Project

## Overview

This repo (`nas-infra`) manages Docker-based services deployed on a **Synology NAS**, accessible exclusively via **Tailscale** (no public IP). Deployment is automated via **GitHub Actions** (push to `main` triggers deploy).

## Architecture

```
Internet (HTTPS :443)
  → Synology DSM nginx (TLS termination, wildcard cert *.inexlab.com)
    → Caddy (:8080, HTTP-only reverse proxy, routes by Host header)
      → App containers (on Docker `proxy` network)
```

### Services

| Subdomain | Container | Port | Repo |
|---|---|---|---|
| `context.inexlab.com` | `context-app` | 3000 | `abalmush/context` (separate repo) |
| `trade.inexlab.com` | `tradeforge-backend` | 8000 | `abalmush/trade-forge` (separate repo) |
| `n8n.inexlab.com` | `n8n` | 5678 | This repo (`nas-infra/n8n/`) |
| `market.inexlab.com` | `market-data-frontend` | 3000 | Separate repo |
| `dsm.inexlab.com` | `host.docker.internal` | 5001 | Synology DSM itself |

### Docker Networks

- **`proxy`** — All services connect to this for Caddy reverse proxying
- **`postgres-net`** — Shared Postgres access between services

### Shared Database

- **`context-postgres`** (Postgres 16 Alpine) — shared Postgres instance running from the `context` app
- Databases: `context`, `tradeforge`, `n8n`
- Password stored in GitHub secret `POSTGRES_PASSWORD`

## Repo Structure

```
nas-infra/
├── .github/workflows/deploy.yml   ← Deploys Caddy + n8n on push to main
├── .gitignore                      ← Excludes .mcp.json, .env
├── .mcp.json                       ← MCP server config for n8n API (local only, gitignored)
├── caddy/
│   ├── Caddyfile                   ← HTTP reverse proxy routes (no TLS, Synology handles that)
│   ├── Dockerfile                  ← FROM caddy:2-alpine
│   └── docker-compose.yml
├── n8n/
│   ├── docker-compose.yml          ← n8n service with Postgres backend
│   └── README.md                   ← Deployment docs
├── CLAUDE.md                       ← This file
└── README.md
```

## Deployment Pipeline (GitHub Actions)

The workflow (`.github/workflows/deploy.yml`) runs on push to `main`:

1. Connects to Tailscale VPN
2. SSHs into NAS
3. Creates `proxy` network
4. Copies & deploys Caddy (build + force-recreate)
5. Caddy health check
6. Creates `postgres-net` network + `n8n` database (idempotent)
7. Copies n8n docker-compose + writes `.env` from secrets
8. Deploys n8n (force-recreate)
9. n8n health check

### GitHub Secrets (nas-infra repo)

| Secret | Purpose |
|---|---|
| `NAS_HOST`, `NAS_PORT`, `NAS_USER`, `NAS_SSH_KEY` | SSH access to NAS |
| `TS_OAUTH_CLIENT_ID`, `TS_OAUTH_CLIENT_SECRET` | Tailscale VPN for CI |
| `POSTGRES_PASSWORD` | Shared Postgres password (`ctx-pg-S3cur3-2026`) |
| `N8N_ENCRYPTION_KEY` | n8n credential encryption (never change!) |
| `CLOUDFLARE_API_TOKEN` | DNS management (not currently used in workflow) |

### Docker Path on NAS

`/volume1/@appstore/ContainerManager/usr/bin` — must be added to PATH in SSH commands.

## SSL/TLS

- **Synology DSM's built-in nginx** handles TLS termination on ports 80/443
- **Wildcard cert** (`*.inexlab.com`) exists in DSM — assign it to new reverse proxy entries
- **Caddy does NOT handle TLS** — it's HTTP-only on port 8080
- For new subdomains: add DSM reverse proxy rule (HTTPS source → localhost:8080) + assign wildcard cert + enable WebSocket headers

## n8n Details

- **Image**: `docker.n8n.io/n8nio/n8n:latest`
- **Database**: Postgres (`n8n` database on `context-postgres` via `postgres-net`)
- **Memory limit**: 1024MB
- **Volumes**: `n8n_data` (config), `n8n_files` (shared files)
- **Key env vars**: `N8N_PROTOCOL=https`, `WEBHOOK_URL=https://n8n.inexlab.com/`, `N8N_HOST=n8n.inexlab.com`
- **Health check**: `wget --spider http://localhost:5678/healthz`
- **WebSocket**: Enabled in Synology DSM reverse proxy custom headers (required for n8n editor)

## MCP Server (n8n API)

This project has a `.mcp.json` that configures the `@illuminaresolutions/n8n-mcp-server` MCP server. This gives Claude direct access to the n8n instance to:

- List, create, update, delete workflows
- Execute workflows
- Manage credentials and tags
- View execution history

The n8n API key and host URL are configured in `.mcp.json` (gitignored).

## Adding a New Service

1. Create `nas-infra/<service>/docker-compose.yml` (join `proxy` network, set resource limits, health check, `restart: unless-stopped`)
2. Add route to `caddy/Caddyfile` (`@service host service.inexlab.com` → `reverse_proxy container:port`)
3. Add deploy steps to `.github/workflows/deploy.yml` (copy files, create env, deploy, health check)
4. **Manual**: Add DNS A record for `service.inexlab.com` → Tailscale IP (`100.76.110.67`)
5. **Manual**: Add Synology DSM reverse proxy rule (HTTPS `service.inexlab.com:443` → HTTP `localhost:8080`) + assign wildcard cert + enable WebSocket headers if needed

## Key Conventions

- All containers use `restart: unless-stopped`
- All services have `deploy.resources.limits.memory`
- All services have `healthcheck` blocks
- Named Docker volumes (not bind mounts) for persistence
- `.env` files created on-NAS by CI from GitHub secrets
- Caddy container name: `caddy`, n8n container name: `n8n`
- SSH commands must include `export PATH=$PATH:/volume1/@appstore/ContainerManager/usr/bin`
