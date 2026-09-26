# Mealie

Recipe management, meal planning, and shopping lists.

## Endpoint

| Item | Value |
|------|-------|
| Container port | 9000 |
| Network | `dokploy-network` |
| Upstream address | `mealie:9000` |
| Local-only host binding | `127.0.0.1:8925` |
| Reverse proxy domain | `mealie.munywele.co.ke` |

## Reverse Proxy

`Caddyfile` in this directory proxies `mealie.munywele.co.ke` to `mealie:9000`. Caddy must be attached to `dokploy-network` to resolve the `mealie` service name.

```bash
cp Caddyfile /path/to/caddy/Caddyfile
```

Do not run this example while Dokploy also has a domain route for Mealie. Two proxies terminating TLS for the same hostname will conflict.

## Required Environment

| Variable | Purpose |
|----------|---------|
| `MEALIE_SECRET_KEY` | session and token signing key |
| `POSTGRES_PASSWORD` | database password, from the shared Postgres stack |

Optional: `MEALIE_DB_NAME` (defaults to `mealie`), `MEALIE_ALLOW_SIGNUP` (defaults to `true`).

## Security

`MEALIE__ALLOW_SIGNUP=true` lets anyone who reaches the domain create an account. Set it to `false` after registering the first user.

## Volumes

`mealie-data` is a named volume mounted at `/app/data/`. Back it up with Dokploy Volume Backups.

## Notes

The stack publishes `127.0.0.1:8925` for local-only access. That binding is not affected by Caddy and can be removed if unused.

`MEALIE_DB_NAME` defaults to `mealie`, but the shared Postgres service only initializes `${POSTGRES_DB}`, which defaults to `media`. Create the `mealie` database manually or align the variable.
