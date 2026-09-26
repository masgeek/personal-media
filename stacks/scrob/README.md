# Scrob

Scrobble tracking for music playback, with Last.fm integration.

## Endpoint

| Item | Value |
|------|-------|
| Container port | 7330 |
| Network | `dokploy-network` |
| Upstream address | `scrob:7330` |
| Local-only host binding | `127.0.0.1:8900` |
| Reverse proxy domain | `scrob.munywele.co.ke` |

## Reverse Proxy

`Caddyfile.example` in this directory proxies `scrob.munywele.co.ke` to `scrob:7330`. Caddy must be attached to `dokploy-network` to resolve the `scrob` service name.

```bash
cp Caddyfile.example /path/to/caddy/Caddyfile
```

Do not run this example while Dokploy also has a domain route for Scrob. Two proxies terminating TLS for the same hostname will conflict.

## Required Environment

| Variable | Purpose |
|----------|---------|
| `DATABASE_URL` | full Postgres connection string |
| `SECRET` | session and token signing key |

Optional: `SCROB_TAG` (defaults to `latest`), `REGISTRATION` (defaults to `false`), `REQUIRE_EMAIL_VALIDATION` (defaults to `false`), `REGISTRATION_MAX_ALLOWED_USERS` (defaults to `0`).

`REGISTRATION=false` means no new accounts can be created through the UI. Create the first account with registration temporarily enabled, then set it back to `false`.

## Volumes

`scrob-data` is a named volume mounted at `/app/backend/data`. Back it up with Dokploy Volume Backups.

## Not Deployed

This stack is tracked in git but is **not** listed in the root `docker-compose.yml`. It does not start as part of a Dokploy deploy. Add the include line to enable it:

```yaml
- stacks/scrob/docker-compose.yml
```

## Known Issues

`DATABASE_URL` points at a database that must already exist. The shared Postgres service only initializes `${POSTGRES_DB}`, which defaults to `media`. Create a dedicated `scrob` database and grant the configured user access to it.
