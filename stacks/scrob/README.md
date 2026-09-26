# Scrob

Self-hosted media tracking for movies and TV shows. Syncs libraries, watch
history, and ratings from Jellyfin, Plex, or Emby, and pushes activity to Trakt.
It is a private Letterboxd/Trakt, not a music scrobbler.

## Endpoint

| Item | Value |
|------|-------|
| Container port | 7330 |
| Network | `dokploy-network` |
| Upstream address | `scrob:7330` |
| Local-only host binding | `127.0.0.1:8900` |
| Reverse proxy domain | `scrob.munywele.co.ke` |

## Reverse Proxy

The consolidated `Caddyfile` at the repository root proxies `scrob.munywele.co.ke` to `scrob:7330`. Caddy must be attached to `dokploy-network` to resolve the `scrob` service name.

## Required Environment

| Variable | Purpose |
|----------|---------|
| `DATABASE_URL` | full Postgres connection string |
| `SECRET` | session and token signing key |
| `SERVER_URL` | public URL, must match the proxied hostname |

Optional: `SCROB_TAG` (defaults to `latest`), `REGISTRATION` (defaults to `false`), `REQUIRE_EMAIL_VALIDATION` (defaults to `false`), `REGISTRATION_MAX_ALLOWED_USERS` (defaults to `0`).

`REGISTRATION=false` means no new accounts can be created through the UI. Create the first account with registration temporarily enabled, then set it back to `false`.

## First Run

1. Set `SERVER_URL` to `https://scrob.munywele.co.ke`, otherwise generated links and auth redirects point at the wrong origin.
2. Add a free TMDB read access token in the UI. Without it, posters, search, and metadata lookups fail.
3. Add Jellyfin as a media source, or configure a Jellyfin webhook for real-time scrobbling.
4. Optionally connect Trakt under Connections → Media Trackers.

## Trakt Limitations

Live two-way Trakt sync requires a **Trakt VIP** subscription. Without VIP you can
still import a Trakt data export zip under Connections → Import, which covers
one-time history import.

## Overlap With Other Stacks

This repository also runs `yamtrack` (media tracking) and `jellystat` (Jellyfin
analytics). Scrob overlaps with Yamtrack on tracking and imports from the same
Jellyfin library. Pick one as the primary tracker or you will maintain the same
watch history twice.

## Volumes

`scrob-data` is a named volume mounted at `/app/backend/data`. Back it up with Dokploy Volume Backups.

## Not Deployed

This stack is tracked in git but is **not** listed in the root `docker-compose.yml`. It does not start as part of a Dokploy deploy. Add the include line to enable it:

```yaml
- stacks/scrob/docker-compose.yml
```

## Known Issues

The upstream `.env.example` uses variable names this stack does not read. It
shipped with `SECRET_KEY`, but the compose file requires `SECRET`, so a deploy
using it fails on the required-variable check. It also pointed `DATABASE_URL` at
a `scrob-db` host, which does not exist in this repository.

`DATABASE_URL` also points at a database that must already exist. The shared
Postgres service only initializes `${POSTGRES_DB}`, which defaults to `media`.
Create a dedicated `scrob` database and grant the configured user access to it.
