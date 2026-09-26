# Paperless-ngx

Document management: scan, index, and archive documents with OCR.

## Endpoint

| Item | Value |
|------|-------|
| Container port | 8010 |
| Network | `dokploy-network` |
| Upstream address | `paperless:8010` |
| Reverse proxy domain | `paperless.munywele.co.ke` |

## Reverse Proxy

`Caddyfile` in this directory proxies `paperless.munywele.co.ke` to `paperless:8010` and raises the request body limit for uploads. Caddy must be attached to `dokploy-network` to resolve the `paperless` service name.

```bash
cp Caddyfile /path/to/caddy/Caddyfile
```

Do not run this example while Dokploy also has a domain route for Paperless. Two proxies terminating TLS for the same hostname will conflict.

## Required Environment

Set the public URL before first use, otherwise generated links point at `http://localhost:8010`:

| Variable | Value |
|----------|-------|
| `PAPERLESS_URL` | `https://paperless.munywele.co.ke` |
| `PAPERLESS_SECRET_KEY` | random value |
| `PAPERLESS_CSRF_TRUSTED_ORIGINS` | `["https://paperless.munywele.co.ke"]` |
| `PAPERLESS_DBNAME` | defaults to `paperless` |
| `PAPERLESS_OCR_LANGUAGE` | defaults to `eng` |

`PAPERLESS_URL` currently defaults to `http://localhost:8010` in both the stack and `.env.example`, which is incorrect behind a reverse proxy.

## Volumes

| Type | Path | Purpose |
|------|------|---------|
| Named volume | `/usr/src/paperless/data` | SQLite database and documents index |
| Named volume | `/usr/src/paperless/media` | Original files and thumbnails |
| Bind mount | `../files/paperless/consume` | Drop folder for new documents |
| Bind mount | `../files/paperless/export` | Export target |

Back up the named volumes with Dokploy Volume Backups.

## Dependencies

Uses the shared `postgres` service for the database and `redis` for caching and task queuing. Neither is exposed to the internet; no reverse proxy applies to them.
