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
| **seerr** | Media request and discovery management | `stacks/seer/` | 5055 | `/` |
| **homarr** | Dashboard for self-hosted services | `stacks/homarr/` | 7576 | `/` |
| **bazarr** | Subtitle management for Radarr and Sonarr | `stacks/bazarr/` | 6767 | `/` |

## Deploy on Dokploy

1. Create a new **Compose** service in Dokploy
2. Set the **Source** to this Git repository
3. Set **Compose Path** to `./docker-compose.yml`
4. Set environment variables in the Dokploy UI (see `.env.example`)
5. Configure domains via the **Domains** tab for each service
6. Click **Deploy**

> Data persists in `../files/` outside the repo, safe from redeploys. Media mounts (`/srv/media`) are expected to exist on the host. Use the Dokploy **Volume Backups** feature for automated backups of named volumes (`postgres_data`, `redis_data`, `yamtrack_data`, `paperless_*`, `seer-data`, `homarr-data`, `bazarr-config`).

## Bazarr Setup

Bazarr uses host networking so it can connect to the host-installed Sonarr and Radarr services through `localhost`. It is available directly at `http://<host-ip>:6767` and is not routed through Dokploy's network proxy.

The stack mounts these host media directories read/write at the same paths inside the container, so paths reported by the host-installed Sonarr and Radarr services remain valid:

- `/mnt/d/Entertainment/Movies` → `/mnt/d/Entertainment/Movies`
- `/mnt/d/Entertainment/TV` → `/mnt/d/Entertainment/TV`

The `/mnt/d/Entertainment/Import` directory is intentionally not mounted.

After deployment, open `http://<host-ip>:6767` and configure:

1. Sonarr at `http://127.0.0.1:8989`, using the Sonarr API key.
2. Radarr at `http://127.0.0.1:7878`, using the Radarr API key.
3. Sonarr root folders under `/mnt/d/Entertainment/TV`.
4. Radarr root folders under `/mnt/d/Entertainment/Movies`.
5. Subtitle languages and providers, then enable automatic searches.

Set `BAZARR_PUID` and `BAZARR_PGID` to the numeric user and group that can write subtitle files. On the Docker host, these can be checked with `id -u` and `id -g`. The Docker daemon must also have access to `/mnt/d/Entertainment`; host paths from another machine are not available to the container.

The Bazarr image is pinned through `BAZARR_VERSION`. Update that value deliberately when upgrading rather than tracking `latest`. Restrict access to port `6767` with the host firewall or an authenticated internal reverse proxy.

## Homarr Setup

Homarr is deployed from `stacks/homarr/docker-compose.yml` and is available at `http://<host-ip>:7576`. Its internal container port remains `7575`; host port `7575` is reserved by Dokploy's nginx.

### Required Secret

Set `HOMARR_SECRET_ENCRYPTION_KEY` in Dokploy before deploying. The value must be a randomly generated 32-byte key, represented as 64 hexadecimal characters.

On Linux or macOS, run:

```bash
openssl rand -hex 32
```

On Windows PowerShell, run:

```powershell
$bytes = [System.Security.Cryptography.RandomNumberGenerator]::GetBytes(32)
[Convert]::ToHexString($bytes).ToLowerInvariant()
```

Copy the resulting 64-character value into the Dokploy environment variable:

```text
HOMARR_SECRET_ENCRYPTION_KEY=your_generated_key
```

Do not include quotes or commit the real key to Git. If `openssl` is available in the deployment environment, this also works:

```bash
docker run --rm alpine/openssl rand -hex 32
```

Keep this key unchanged after the first deployment. Changing it can make encrypted Homarr data unreadable.

### Monitor Host Services

The media automation services run directly on the host, so Homarr cannot discover them through Docker. Add them manually in the Homarr UI using the host's LAN IP or an internal DNS name:

| Service | URL |
|---------|-----|
| Jellyfin | `http://<host-ip>:8096` |
| Radarr | `http://<host-ip>:7878` |
| Sonarr | `http://<host-ip>:8989` |
| Bazarr | `http://<host-ip>:6767` |
| Prowlarr | `http://<host-ip>:9696` |
| qBittorrent | `http://<host-ip>:8080` |

Use the service's API key when configuring its Homarr widget to show health, queues, downloads, and other details. The API key is configured in each host service, not in this repository.

Host services must listen on an address reachable from the clients using Homarr, not only on `127.0.0.1`. If they are bound to localhost, use a host reverse proxy or configure them to listen on the host LAN interface.

Do not expose Radarr, Sonarr, Bazarr, Prowlarr, or qBittorrent directly to the public internet. Restrict their host firewall rules or place them behind an authenticated internal reverse proxy.

### Monitor Docker Services

Homarr has read-only access to `/var/run/docker.sock` for Docker integration. This allows it to monitor containers in the Docker daemon, but it does not monitor host-installed services. Keep the socket mount read-only and back up the `homarr-data` volume with Dokploy's **Volume Backups** feature.

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
- Each service has its own folder under `stacks/` with a `docker-compose.yml`.
- No `depends_on` — all dependencies (postgres, redis) are external and assumed available.
- All services connect via `dokploy-network`.
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
