# MediaHub Infrastructure Summary

## Overview

This document describes the planned architecture for a personal self-hosted media ecosystem ("MediaHub"). The goal is to separate media serving, automation, tracking, and supporting services into logical components for easier maintenance and scalability.

---

# Objectives

* Self-hosted media streaming
* Automated media acquisition and organization
* Personal media tracking and analytics
* AI-powered recommendations (future)
* Easy backup and maintenance
* Native performance for services that interact heavily with storage

---

# Storage Layout

All media is stored under a single root directory.

```
/srv/media
├── Movies
├── TV
├── Music
├── Audiobooks
├── Podcasts
├── Books
├── Manga
├── Comics
├── Downloads
│   ├── Complete
│   └── Incomplete
└── Transcode
```

---

# Native (Host) Services

These are installed directly on the host and managed using systemd.

## Jellyfin

Purpose:

* Primary movie and TV streaming server.

Reasons for native installation:

* Direct GPU access
* Better hardware transcoding
* Simpler filesystem permissions
* Easier integration with VAAPI/QSV/NVIDIA
* Direct access to `/srv/media`

---

## qBittorrent

Purpose:

* Download client.

Responsibilities:

* Download media
* Maintain completed download directory
* Interface with automation services

---

## Sonarr

Purpose:

* TV show management.

Responsibilities:

* Monitor series
* Import completed downloads
* Rename files
* Organize TV library

---

## Radarr

Purpose:

* Movie management.

Responsibilities:

* Import movies
* Organize movie library
* Rename media

---

## Lidarr

Purpose:

* Music management.

---

## Readarr

Purpose:

* Book management.

---

## Bazarr

Purpose:

* Subtitle management.

---

## Prowlarr

Purpose:

* Centralized indexer management.

Responsibilities:

* Manage indexers
* Provide indexers to Sonarr/Radarr/etc.

---

# Docker Services

Docker is used only for supporting applications.

---

## PostgreSQL

Purpose:

* Shared database server.

Separate databases should be created for each application.

Example:

* yamtrack
* mealie
* future applications

Do **not** share a single database between applications.

---

## Redis

Purpose:

* Cache
* Message broker
* Background job queue

Used primarily by Yamtrack.

---

## Yamtrack

Purpose:

* Personal media tracker.

Tracks:

* Movies
* TV Shows
* Anime
* Games
* Books
* Manga
* Podcasts

Future use:

* Recommendation engine data source
* Viewing history
* Ratings
* Personal statistics

Depends on:

* PostgreSQL
* Redis

---

## Navidrome

Purpose:

* Dedicated music streaming server.

Media source:

```
/srv/media/Music
```

Reason:

Provides a better music experience than Jellyfin.

---

## Audiobookshelf

Purpose:

* Audiobooks
* Podcasts

Media:

```
/srv/media/Audiobooks
/srv/media/Podcasts
```

Features:

* Progress tracking
* Chapter support
* Speed control
* Mobile clients

---

## Kavita

Purpose:

* Digital library.

Media:

```
/srv/media/Books
/srv/media/Manga
/srv/media/Comics
```

Supports:

* EPUB
* PDF
* Manga
* Comics

---

## Stats (JellyGlance)

Purpose:

* Admin dashboard for Jellyfin or Emby.

Uses:

* Jellyfin or Emby API
* PostgreSQL, from the shared Postgres stack

Provides:

* Live sessions with direct play and transcode detail
* Library totals, recent additions and metadata gaps
* Watch history and activity
* Request, download and transcode queues
* Server jobs, health checks, backups, imports and webhooks

Exposed at https://stat.munywele.co.ke through host Caddy in proxy-tool.

---

# Future Services

## Open WebUI

Purpose:

Web interface for local AI models.

---

## Ollama

Purpose:

Run local LLMs.

Potential future uses:

* Semantic media search
* Recommendation assistant
* Natural language querying
* Metadata enrichment

---

# Planned Recommendation Engine

Future machine learning project.

Possible inputs:

* Yamtrack ratings
* Yamtrack watch history
* Stats viewing history
* Jellyfin metadata
* Genres
* Actors
* Directors
* Runtime
* Release year
* Tags
* Embeddings generated from descriptions

Potential models:

* Collaborative filtering
* Content-based filtering
* Hybrid recommendation system

---

# Overall Architecture

```
                    Internet
                        │
                   Prowlarr
                        │
      ┌─────────────────┴─────────────────┐
      │                                   │
   Sonarr                            Radarr
      │                                   │
   Readarr                           Lidarr
      │                                   │
                qBittorrent
                     │
             Completed Downloads
                     │
             /srv/media (host)
                     │
 ┌───────────────────┼────────────────────┐
 │                   │                    │
Jellyfin        Navidrome          Audiobookshelf
 │
 │
Stats (JellyGlance)
 │
Yamtrack
 │
Recommendation Engine (future)
 │
Open WebUI + Ollama (future)
```

---

# Deployment Philosophy

Native services:

* High filesystem interaction
* GPU access
* Media automation
* Media serving

Docker services:

* Supporting applications
* Databases
* Analytics
* Tracking
* AI services

This separation minimizes filesystem complexity, avoids Docker permission issues, simplifies hardware acceleration, and keeps infrastructure modular and maintainable.

---

# Repository Guide (Docker Compose, Dokploy)

This repo holds only the Docker side: Compose files, the `Caddyfile`, and docs. There is no application code, build, lint, or test step. `README.md` has per-service setup detail.

## Layout and Stack Wiring

* Root `docker-compose.yml` is only an `include:` list of `stacks/<name>/docker-compose.yml`. Each stack is self-contained and has its own README where it needs setup steps.
* Commenting out an include line disables a stack. Currently disabled: `postgres`, `redis`, `bazarr`. `scrob` and `pihole` exist in `stacks/` but are not in the include list. Check the include list for what actually deploys; README's "all stacks are included" is not literally true.
* `postgres` and `redis` are shared dependencies that other stacks assume are already running. There is no `depends_on`.
* Relative paths in an included file resolve against that file's directory. `../files/` from a stack means `stacks/files/`, which is gitignored.
* `Caddyfile` is an alternative to Dokploy's Traefik Domains tab. Never configure the same hostname in both. Only yamtrack, scrob, mealie, and paperless are proxied. `stats` is proxied by a host Caddy in a separate `proxy-tool` project.

## Commands

Deploy from the Dokploy UI (Compose path `./docker-compose.yml`). Local equivalents:

```bash
docker compose config                                      # validate the root include set
docker compose -f stacks/<name>/docker-compose.yml config  # validate one stack
docker compose -f stacks/<name>/docker-compose.yml up -d   # run one stack
```

Compose files require env vars (`${VAR:?err}`). `.env` is gitignored, so copy `.env.example` to `.env` first.

## Conventions for Editing Stacks

* Set `container_name` and `hostname` on every service.
* Network: `dokploy-network` (`external: true`). Exceptions: host-network stacks (`seerr`, `bazarr`, `pihole`, `suggestarr`) must reach host-installed apps via `127.0.0.1`; `yamtrack`, `mealie`, and `scrob` also declare an `internal` network.
* Ports use one of three patterns: container-only (`- 8000`, routed by Dokploy), loopback-only (`127.0.0.1:HOST:CONTAINER`), or host network. Avoid host port collisions. `7575` is reserved by Dokploy's nginx; `80`/`443` belong to the reverse proxy. Update the README host port map when adding a port.
* Env vars go directly in `environment:` blocks. Use `${VAR:?err}` for required and `${VAR:-default}` for optional. No `env_file`.
* Always set `deploy.resources.limits.memory`.
* Logging: json-file with `max-size: 10m` and `max-file: 3`.
* Volumes: named volumes for app state, because Dokploy re-clones the repo and wipes `stacks/files/`. `../files/` bind mounts only for disposable or consume/export dirs. External media uses absolute host paths: `/srv/media`, or `/mnt/d/Entertainment` when Docker runs in WSL2.
* Image tag: use `image: <name>:${TAG:-latest}` in every stack except `postgres` and `redis`, which stay pinned to a major version. `TAG` is shared by all included stacks, so setting it changes every image at once.
* Seek confirmation before adding a new service. Propose its stack, shared infra needs, volume strategy, and env vars first.
* When a stack's env vars, ports, or domains change, update `.env.example`, the README services table and port map, and the `Caddyfile` if proxied.