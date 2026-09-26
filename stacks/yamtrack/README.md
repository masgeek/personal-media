# Yamtrack

Personal media tracker for movies, shows, anime, games, books, manga, and podcasts.

## Endpoint

| Item | Value |
|------|-------|
| Container port | 8000 |
| Network | `dokploy-network` |
| Upstream address | `yamtrack:8000` |
| Local-only host binding | `127.0.0.1:8800` |
| Reverse proxy domain | `track.munywele.co.ke` |

## Reverse Proxy

`Caddyfile` in this directory proxies `track.munywele.co.ke` to `yamtrack:8000`.

```bash
cp Caddyfile /path/to/caddy/Caddyfile
```

Caddy must be attached to `dokploy-network` to resolve the `yamtrack` service name:

```yaml
services:
  caddy:
    image: caddy:2
    networks:
      - dokploy-network
```

Do not run this example while Dokploy also has a domain route for Yamtrack. Two proxies handling TLS for the same hostname will conflict; pick one owner per hostname.

## Required Environment

These variables have no defaults and must be set before deploying:

| Variable | Purpose |
|----------|---------|
| `SECRET` | Django secret key |
| `DB_USERNAME` | PostgreSQL user |
| `DB_PASSWORD` | PostgreSQL password |

Optional: `YAMTRACK_TAG`, `DEBUG`, `ADMIN_ENABLED`, `REGISTRATION`, `REDIS_PREFIX`, `DB_HOST`, `DB_PORT`, `TRAKT_CLIENT_ID`, `TRAKT_CLIENT_SECRET`, `SIMKL_CLIENT_ID`, `SIMKL_CLIENT_SECRET`, `STEAM_API_KEY`.

## Known Issues

1. `REDIS_URL` defaults to `redis://cache:6379`, but this repository's Redis service is named `redis`. Set `REDIS_URL=redis://redis:6379` or change the default.
2. `DB_NAME` defaults to `yamtrack`, but PostgreSQL only initializes `${POSTGRES_DB}`, which defaults to `media`. Create the `yamtrack` database manually or align the variable.
3. `stacks/yamtrack/.env.example` documents different variable names (`YAMTRACK_SECRET`, `POSTGRES_USER`, `POSTGRES_PASSWORD`) than the ones this stack requires.
