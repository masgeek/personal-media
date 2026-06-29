# Personal Media Stack

Docker Compose-based media infrastructure organized into logical stacks, configured for [Dokploy](https://dokploy.com) deployment.

## Stacks

| Stack | File | Services |
|-------|------|----------|
| Core | `stacks/core.yml` | postgres, redis |
| Tracking | `stacks/tracking.yml` | yamtrack, jellystat |
| Media Servers | `stacks/media-servers.yml` | navidrome |
| Documents | `stacks/documents.yml` | paperless-ngx |

## Services

| Service | Purpose | Stack | Port | Web UI |
|---------|---------|-------|------|--------|
| **postgres** | Database for yamtrack, jellystat, and paperless-ngx | Core | — | — |
| **redis** | Cache/queue for yamtrack and paperless-ngx | Core | — | — |
| **yamtrack** | Personal media tracking (movies, shows, games, books) | Tracking | 8000 | `/` |
| **jellystat** | Viewing statistics and analytics for Jellyfin | Tracking | 3000 | `/` |
| **navidrome** | Music streaming server (Subsonic-API compatible) | Media Servers | 4533 | `/app` |
| **paperless-ngx** | Document management system (scan, index, archive) | Documents | 8010 | `/` |

## Deploy on Dokploy

1. Create a new **Compose** service in Dokploy
2. Set the **Source** to this Git repository
3. Set **Compose Path** to `./docker-compose.yml`
4. Set environment variables in the Dokploy UI (see `.env.example`)
5. Configure domains via the **Domains** tab for each service
6. Click **Deploy**

> Data persists in `../files/` outside the repo, safe from redeploys. Media mounts (`/srv/media`) are expected to exist on the host. Use the Dokploy **Volume Backups** feature for automated backups of named volumes (`postgres_data`, `redis_data`, `yamtrack_data`, `paperless_*`).

## Deploy a Single Stack

```bash
docker compose -f stacks/core.yml -f stacks/tracking.yml up -d
docker compose -f stacks/media-servers.yml up -d
docker compose -f stacks/documents.yml up -d
```
