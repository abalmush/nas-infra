# n8n Deployment on NAS

## Overview

n8n is deployed as a Docker container on the Synology NAS, following the same patterns as other apps (`context`, `trade-forge`). It uses the shared Postgres instance (`context-postgres`) for its database and is fronted by the Caddy reverse proxy.

## Architecture

```
Internet → n8n.inexlab.com → Caddy (:80) → n8n:5678
                                              ↓
                                     context-postgres:5432 (database: n8n)
```

### Networks
- **`proxy`** — connects n8n to Caddy for reverse proxying
- **`postgres-net`** — connects n8n to the shared `context-postgres` instance

### Volumes
- **`n8n_data`** — persists n8n config, encryption keys, and internal state (`/home/node/.n8n`)
- **`n8n_files`** — shared files directory for the Read/Write Files from Disk node (`/files`)

### Resource Limits
- Memory: 1024MB

## Environment Variables

| Variable | Source | Purpose |
|---|---|---|
| `POSTGRES_PASSWORD` | GitHub secret | Password for the shared Postgres instance |
| `N8N_ENCRYPTION_KEY` | GitHub secret | **Critical** — encrypts/decrypts all stored credentials. Once set, never change it |
| `GENERIC_TIMEZONE` | Hardcoded | `America/New_York` — used by scheduling nodes |
| `N8N_HOST` | Hardcoded | `n8n.inexlab.com` |
| `WEBHOOK_URL` | Hardcoded | `http://n8n.inexlab.com/` — required for webhooks behind reverse proxy |
| `DB_TYPE` | Hardcoded | `postgresdb` — use Postgres instead of default SQLite |
| `DB_POSTGRESDB_HOST` | Hardcoded | `context-postgres` — shared Postgres container |
| `DB_POSTGRESDB_DATABASE` | Hardcoded | `n8n` — dedicated database created during deployment |

## Pre-Deployment Checklist

### 1. Add DNS Record

| Record Type | Name | Value |
|---|---|---|
| A | `n8n.inexlab.com` | Same IP as your other `*.inexlab.com` records |

### 2. Add GitHub Secret

| Secret | How to Generate |
|---|---|
| `N8N_ENCRYPTION_KEY` | `openssl rand -hex 32` |

> ⚠️ **Save this key securely.** Losing it means losing access to all stored credentials in n8n. It cannot be recovered.

> `POSTGRES_PASSWORD` is already available (shared with the `context` app).

### 3. Deploy

Push to the `main` branch of `nas-infra` to trigger the GitHub Actions workflow.

## What the CI/CD Workflow Does

1. Deploys Caddy (with the new `n8n.inexlab.com` route)
2. Creates `postgres-net` network (idempotent)
3. Connects `context-postgres` to `postgres-net` (idempotent)
4. Creates the `n8n` database in Postgres (idempotent)
5. Copies `docker-compose.yml` to `~/n8n/` on the NAS
6. Writes `.env` file with secrets
7. Runs `docker compose up -d --force-recreate`
8. Health checks `/healthz` endpoint (10 retries, 5s intervals)

## Post-Deployment

- Access n8n at `https://n8n.inexlab.com`
- On first launch, n8n will prompt you to **create an owner account**
- All workflows, credentials, and execution history are persisted in Postgres + the `n8n_data` volume

## Files

| File | Purpose |
|---|---|
| `n8n/docker-compose.yml` | Docker Compose service definition |
| `caddy/Caddyfile` | Updated with `@n8n` reverse proxy route |
| `.github/workflows/deploy.yml` | Updated with n8n deployment steps |

## Troubleshooting

### Check n8n logs
```bash
docker logs n8n --tail 100
```

### Check if n8n is healthy
```bash
docker exec n8n wget -q --spider http://localhost:5678/healthz
```

### Check database connectivity
```bash
docker exec context-postgres psql -U postgres -d n8n -c "SELECT 1"
```

### Restart n8n
```bash
cd ~/n8n && docker compose restart
```
