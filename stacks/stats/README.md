# Stats — JellyGlance

[JellyGlance](https://jellyglance.com/) is a self-hosted dashboard that sits beside
**Jellyfin or Emby** and gathers the daily admin checks into one interface: live
sessions, libraries, users, requests, download queues, transcodes, invites,
cleanup, server jobs, health checks, backups and webhooks.

It replaces `stacks/jellystat`, which was Jellyfin-only analytics with no session
view and no integrations.

The stack is named for what it does, following the `yamtrack` convention. The
container is `jellyglance` so `docker compose logs jellyglance` works.

| | |
| --- | --- |
| Public URL | https://stat.munywele.co.ke |
| Host binding | `127.0.0.1:8930` → container `3000` |
| Database | `jellyglance` on the shared `stacks/postgres` |
| Upstream | https://github.com/Nerdy-Technician/JellyGlance |

## What it needs

Plain PostgreSQL 16 or newer, from this repository's `stacks/postgres`. **No
extension, no TimescaleDB, no Redis.** That is the main reason it is lighter than
what it replaced: a self-contained database container and a queue broker are not
required, so it reuses what is already running.

Create the database once, before the first start:

```bash
docker exec postgres createdb --username "$POSTGRES_USER" --if-not-exists jellyglance
```

The app does not create its own schema, so this step is not optional.

## Configuration

Everything below is in the root `.env`, not a per-stack file, because that is how
this repository handles credentials.

| Variable | Purpose |
| --- | --- |
| `DB_HOST` | Defaults to `postgres`, the shared database container |
| `DB_DATABASE` | Defaults to `jellyglance` |
| `DB_USERNAME` / `DB_PASSWORD` | Database account. Required |
| `JWT_SECRET` | Signs session tokens. **Changing it logs everyone out** |
| `JELLYGLANCE_PUBLIC_URL` | Used when building webhook card-image URLs |
| `JELLYGLANCE_TAG` | Image tag |

Generate the signing secret with:

```bash
openssl rand -hex 32
```

A non-empty value in `.env.example` is a placeholder. Replace it before deploying.

## First run

1. Start the stack and open https://stat.munywele.co.ke
2. Choose **Auto-detect**, **Jellyfin** or **Emby**, then give it the server URL
3. Enter that server's API key so it can validate the connection and sync libraries
4. Choose an admin access mode: local account, Jellyfin Quick Connect, Emby sign-in
   or OIDC
5. Let the first sync finish

Everything else — integrations, tasks, webhooks, imports, newsletter — is configured
inside the application under **Settings**, not in the environment.

## Integrations worth connecting here

This repository already runs most of what JellyGlance can display:

| Integration | In this repository |
| --- | --- |
| Tdarr | `stacks/transcode` |
| Bazarr | `stacks/bazarr` |
| qBittorrent | native on the host |
| Sonarr, Radarr, Lidarr, Readarr, Prowlarr | native on the host |
| Jellyseerr / Overseerr | `stacks/seer` |

`stacks/homarr` can also consume JellyGlance's widget endpoints, if the Homarr
widget turns out to be more useful than JellyGlance's own.

## Importing the old history

JellyGlance imports **Jellystat backup files**, Tautulli backups and Trakt history
under **Settings → Imports**, plus play-state sync from the media server. So the
history from the retired stack is recoverable if it is wanted:

```bash
docker cp jellystat:/data ./jellystat-export   # before removing the old container
```

Take that copy first if it matters. The old stack is removed from this repository
already, so the container may no longer exist to copy from.

## Routing

This service is proxied by **host Caddy in proxy-tool**, not by this repository's
Caddyfile, which does not carry the public media domains. The route lives at
`config/caddy/snippets/domains/personal.caddy` in proxy-tool and sets
`flush_interval -1`, because JellyGlance uses WebSockets for live session updates
and Caddy's default response buffering would delay them.

## Volumes and backup

| Volume | Contents |
| --- | --- |
| `jellyglance_config` | Application configuration |
| `jellyglance_backups` | Backup exports written by the built-in scheduler |

The database lives in the shared `postgres_data` volume, so it is covered by
whatever protects that volume. The two volumes here are not: copy them, or use
**Settings → Backup**.

## Upgrading

```bash
docker compose pull jellyglance
docker compose up -d jellyglance
```

## Cleaning up

The retired stack used the `jellystat` database. It is in no include file, so
nothing recreates it:

```bash
docker exec postgres dropdb --username "$POSTGRES_USER" --if-exists jellystat
```