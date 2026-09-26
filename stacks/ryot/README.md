# Ryot

Media tracking and discovery. Consumes Radarr, Sonarr, and Plex libraries.

## Endpoint

| Item | Value |
|------|-------|
| Container port | 8000 |
| Network | `dokploy-network` |
| Upstream address | `ryot:8000` |
| Local-only host binding | `127.0.0.1:8950` |
| Reverse proxy domain | `ryot.munywele.co.ke` |

## Reverse Proxy

`Caddyfile` in this directory proxies `ryot.munywele.co.ke` to `ryot:8000`. Caddy must be attached to `dokploy-network`.

## Required Environment

All three variables are required and have no defaults:

| Variable | Purpose |
|----------|---------|
| `DATABASE_URL` | full Postgres connection string, including credentials and database name |
| `FRONTEND_URL` | public URL, must match the proxy hostname exactly |
| `SERVER_ADMIN_ACCESS_TOKEN` | admin API token |

`FRONTEND_URL` must be set to the value used in the Caddy site block, for example `https://ryot.munywele.co.ke`. A mismatch causes broken links and failed OAuth-style redirects.

Optional: `RYOT_TAG` (defaults to `v10`).

## Not Deployed

This stack is tracked in git but is **not** listed in the root `docker-compose.yml`. It does not start as part of a Dokploy deploy. Add the include line to enable it:

```yaml
- stacks/ryot/docker-compose.yml
```

## Known Issues

`DATABASE_URL` points at a database that must exist. The shared Postgres service only initializes `${POSTGRES_DB}`, which defaults to `media`. Create a dedicated `ryot` database and grant the user access to it.
