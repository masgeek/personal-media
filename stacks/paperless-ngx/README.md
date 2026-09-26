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

The consolidated `Caddyfile` at the repository root proxies `paperless.munywele.co.ke` to `paperless:8010` and raises the request body limit for uploads. Caddy must be attached to `dokploy-network` to resolve the `paperless` service name.

## Required Environment

Set the public URL before first use, otherwise generated links and redirects
point at `http://localhost:8010`:

| Variable | Value |
|----------|-------|
| `PAPERLESS_URL` | `https://paperless.munywele.co.ke` |
| `PAPERLESS_CSRF_TRUSTED_ORIGINS` | `["https://paperless.munywele.co.ke"]` |
| `PAPERLESS_SECRET_KEY` | random value |
| `PAPERLESS_DBNAME` | defaults to `paperless` |
| `PAPERLESS_OCR_LANGUAGE` | defaults to `eng` |

Both URL variables default to the proxied hostname in the stack, so they are
correct out of the box. Override them only if the hostname changes.

`PAPERLESS_SECRET_KEY` has no default and is required. Generate one with
`openssl rand -hex 32`.

The CSRF trusted origins value is a JSON array, so it must keep its brackets and
quotes. Losing them makes Django reject form submissions with a 403.

## First Run

1. Create the superuser when prompted on first visit.
2. Add a consumption directory pointing at `/usr/src/paperless/consume`, which
   maps to `../files/paperless/consume` on the host. Drop files there to import.
3. Add matching document and workflow directories under
   `/usr/src/paperless/media`, which maps to the `paperless_media` volume. The
   paths must match or the consumption task will not match files to documents.
4. Confirm the task worker starts. OCR and indexing run through Redis, and a
   missing `redis` service leaves documents stuck in the `STARTED` state.

## Volumes

| Type | Path | Purpose |
|------|------|---------|
| Named volume | `/usr/src/paperless/data` | Search index, cached thumbnails, and logs |
| Named volume | `/usr/src/paperless/media` | Original files and generated archive documents |
| Bind mount | `../files/paperless/consume` | Drop folder for new documents |
| Bind mount | `../files/paperless/export` | Export target |

The document database is **not** in these volumes. It lives in the shared
`postgres` service, so a Postgres backup covers the data and the named volumes
cover the files and index.

Back up the named volumes with Dokploy Volume Backups.

## Dependencies

Uses the shared `postgres` service for the database and `redis` for caching and
task queuing. Neither is exposed to the internet; no reverse proxy applies to
them. No `depends_on` is set, so both must already be running when Paperless
starts.

## Known Issues

`PAPERLESS_DBNAME` defaults to `paperless`, but the shared Postgres service only
initializes `${POSTGRES_DB}`, which defaults to `media`. A fresh deployment
fails to connect until the `paperless` database is created:

```bash
docker compose -f stacks/postgres/docker-compose.yml exec postgres \
  createdb -U media paperless
```

The image is pinned to `latest`, so upgrades arrive on redeploy without review.
