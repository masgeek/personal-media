# Personal Media Stack

Docker Compose-based media infrastructure organized into logical stacks, configured for [Dokploy](https://dokploy.com) deployment.

## Services

| Service | Purpose | File | Port | Web UI |
|---------|---------|------|------|--------|
| **postgres** | Database for all dependent services | `stacks/postgres/` | — | — |
| **redis** | Cache/queue for yamtrack and paperless-ngx | `stacks/redis/` | — | — |
| **yamtrack** | Personal media tracking (movies, shows, games, books) | `stacks/yamtrack/` | 8000 | `/` |
| **jellystat** | Viewing statistics and analytics for Jellyfin | `stacks/jellystat/` | 3000 | `/` |
| **navidrome** | Music streaming server (Subsonic-API compatible) | `stacks/navidrome/` | 4533 | `/app` |
| **paperless-ngx** | Document management system (scan, index, archive) | `stacks/paperless-ngx/` | 8010 | `/` |
| **mealie** | Recipe management, meal planning, and shopping lists | `stacks/mealie/` | 9000 | `/` |

## Deploy on Dokploy

1. Create a new **Compose** service in Dokploy
2. Set the **Source** to this Git repository
3. Set **Compose Path** to `./docker-compose.yml`
4. Set environment variables in the Dokploy UI (see `.env.example`)
5. Configure domains via the **Domains** tab for each service
6. Click **Deploy**

> Data persists in `../files/` outside the repo, safe from redeploys. Media mounts (`/srv/media`) are expected to exist on the host. Use the Dokploy **Volume Backups** feature for automated backups of named volumes (`postgres_data`, `redis_data`, `yamtrack_data`, `paperless_*`).

## Conventions

### Dokploy
- **`container_name` + `hostname`** — set on every service for predictable DNS and container naming.
- **Network** — always use `dokploy-network` (external: true). Never custom networks.
- **Ports** — container-only (e.g. `- 8000`), no host binding. Dokploy/Traefik routes via the network.
- **Env vars** — pass directly via `environment:` blocks. Use `${VAR:?err}` for required vars, `${VAR:-default}` for optional. No `env_file` — vars come from the environment (Dokploy UI / shell).
- **Bind mounts** — use `../files/` paths (not `./`, not absolute). Absolute paths get cleaned on redeploy.
- **Resource limits** — always set `deploy.resources.limits.memory`.
- **Logging** — always use json-file driver with `max-size: 10m` / `max-file: 3`.

### Stacks
- Each stack file is self-contained (services + volumes + network reference).
- Shared infrastructure (postgres, redis) lives in `core.yml`.
- A new service goes in a new or existing stack file depending on category.
- All stacks are included from root `docker-compose.yml`.

### Volumes

| Type | Use for |
|------|---------|
| Named volumes | App-internal data that doesn't need direct host access (databases, media stores). Enables Dokploy Volume Backups. |
| `../files/` bind mounts | Paths the user needs to access directly on the host (consume folders, export folders, config overrides). |

### Adding New Services

Seek confirmation before adding a new service. Propose:
- Which stack it belongs in (or if a new stack is needed)
- What shared infrastructure it needs (postgres, redis)
- Whether to use named volumes or bind mounts for each data path
- Required env vars and reasonable defaults

## Deploy a Single Stack

```bash
docker compose -f stacks/core.yml -f stacks/tracking.yml up -d
docker compose -f stacks/media-servers.yml up -d
docker compose -f stacks/documents.yml up -d
docker compose -f stacks/management.yml up -d
```
