# Personal Media Stack

Docker Compose-based media infrastructure organized into logical stacks, configured for [Dokploy](https://dokploy.com) deployment.

## Services

| Service | Purpose | File | Container port | Host binding |
|---------|---------|------|----------------|--------------|
| **postgres** | Database for all dependent services | `stacks/postgres/` | — | none, internal only |
| **redis** | Cache/queue for yamtrack and paperless-ngx | `stacks/redis/` | — | none, internal only |
| **yamtrack** | Personal media tracking (movies, shows, games, books) | `stacks/yamtrack/` | 8000 | `127.0.0.1:8800` |
| **jellystat** | Viewing statistics and analytics for Jellyfin | `stacks/jellystat/` | 3000 | none, container-only |
| **paperless-ngx** | Document management system (scan, index, archive) | `stacks/paperless-ngx/` | 8010 | none, container-only |
| **mealie** | Recipe management, meal planning, and shopping lists | `stacks/mealie/` | 9000 | `127.0.0.1:8925` |
| **seerr** | Media request and discovery management | `stacks/seer/` | 5055 | host network |
| **homarr** | Dashboard for self-hosted services | `stacks/homarr/` | 7575 | `7576:7575` |
| **bazarr** | Subtitle management for Radarr and Sonarr | `stacks/bazarr/` | 6767 | host network |
| **transcode** | Library video optimisation and conversion | `stacks/transcode/` | 8265 | host network |
| **ryot** | Media tracking and discovery | `stacks/ryot/` | 8000 | `127.0.0.1:8950` |
| **scrob** | Movie and TV tracking with Trakt scrobbling | `stacks/scrob/` | 7330 | `127.0.0.1:8900` |
| **pihole** | Network-wide ad blocking via DNS | `stacks/pihole/` | 8081, 53 | host network |

`ryot`, `scrob`, and `pihole` are not included in the root `docker-compose.yml`, so they do not start on deploy. See [Reverse Proxy with Caddy](#reverse-proxy-with-caddy).

## Deploy on Dokploy

1. Create a new **Compose** service in Dokploy
2. Set the **Source** to this Git repository
3. Set **Compose Path** to `./docker-compose.yml`
4. Set environment variables in the Dokploy UI (see `.env.example`)
5. Configure domains via the **Domains** tab for each service
6. Click **Deploy**

> **Bind mount paths** — `../files/` in a stack file resolves to `stacks/files/`, inside this repository. Each included Compose file resolves relative paths against its own directory, not the root, so `stacks/paperless-ngx/` plus `../files/` gives `stacks/files/`. This is intentional: the path is stable across redeploys and does not depend on where the repository is cloned.
>
> `stacks/files/` is gitignored, so consumed documents and exports cannot be committed by accident. Back it up separately from the named volumes.
>
> Named volumes: use the Dokploy **Volume Backups** feature for `postgres_data`, `redis_data`, `paperless_data`, `paperless_media`, `seer-data`, `homarr-data`, `bazarr-config`, and `scrob-data`. The `transcode` stack keeps its settings in the `stacks/files/` bind mounts instead, so back up `stacks/files/` directly. External media mounts are expected to exist on the Docker host, either at `/srv/media` or at `/mnt/d/Entertainment` when Dokploy runs inside WSL2.

## Bazarr Setup

Bazarr uses host networking so it can connect to the host-installed Sonarr and Radarr services through `localhost`. It is available directly at `http://<host-ip>:6767` and is not routed through Dokploy's network proxy.

The stack mounts these media directories read/write, using the WSL view of the
Windows `D:` drive because the Docker daemon runs inside WSL2:

- `/mnt/d/Entertainment/Movies` → `/movies`
- `/mnt/d/Entertainment/TV` → `/tv`

The `Import` directory is intentionally not mounted. Bazarr should only touch the final libraries.

After deployment, open `http://<host-ip>:6767` and configure:

1. Sonarr at `http://127.0.0.1:8989`, using the Sonarr API key.
2. Radarr at `http://127.0.0.1:7878`, using the Radarr API key.
3. Add a Sonarr path mapping from `D:\Entertainment\TV` to `/tv`.
4. Add a Radarr path mapping from `D:\Entertainment\Movies` to `/movies`.
5. Configure subtitle languages and providers, then enable automatic searches.

The path mappings are required because Sonarr and Radarr run on Windows and report `D:\Entertainment\...` in their API responses, while Bazarr sees the same files at `/tv` and `/movies`. The container paths are what you use when configuring Bazarr itself.

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
- **Network** — `dokploy-network` (external: true) is the default for every service. Exceptions are the host-network stacks (`seerr`, `bazarr`, `pihole`), which must reach host-installed apps, and the `internal` network declared by `yamtrack`, `mealie`, `ryot`, and `scrob` alongside `dokploy-network`.
- **Ports** — three patterns are in use, and the table above shows which applies to each service:
  - **Container-only** (`- 8000`) is the default. Dokploy/Traefik routes to the service over `dokploy-network`, so no host port is published.
  - **Loopback-only** (`- "127.0.0.1:8800:8000"`) publishes a host port bound to `127.0.0.1` for local-only debugging. It is not reachable from the LAN. Note the host port usually differs from the container port.
  - **Host network** (`network_mode: host`) is required for services that must reach host-installed apps such as Jellyfin, Sonarr, and Radarr through `127.0.0.1`, and for Pi-hole's DNS on port 53. These services cannot join `dokploy-network`, so Dokploy cannot route to them by name and they are reached on the host port directly.
- **Host port map** — loopback `8800`, `8900`, `8925`, `8950`; published `7576`; host-network `5055`, `6767`, `8081`, `8265`, `8266`, `53`. Port `7575` is reserved by Dokploy's nginx, and `80`/`443` are owned by the reverse proxy. No two stacks claim the same host port.
- **Env vars** — pass directly via `environment:` blocks. Use `${VAR:?err}` for required vars, `${VAR:-default}` for optional. No `env_file` — vars come from the environment (Dokploy UI / shell).
- **Bind mounts** — two kinds. Config and app data the repo owns use `../files/`, which resolves to `stacks/files/`. External media uses an absolute host path, because it lives outside the repo and cannot be repo-relative.
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
docker compose -f stacks/bazarr/docker-compose.yml up -d
docker compose -f stacks/homarr/docker-compose.yml up -d
```

## Reverse Proxy with Caddy

Dokploy already terminates TLS through Traefik using the **Domains** tab. The
root `Caddyfile` is for the alternative case where a standalone Caddy instance
owns the hostname instead. It is a single consolidated file covering every
proxied service, not one file per stack.

Only the trackers, Mealie, and Paperless-ngx are proxied. Everything else in
this repository is an internal service reached over the LAN, and is deliberately
absent from the Caddyfile:

| Service | Domain | Upstream | In root compose |
|---------|--------|----------|-----------------|
| yamtrack | `track.munywele.co.ke` | `yamtrack:8000` | yes |
| ryot | `ryot.munywele.co.ke` | `ryot:8000` | no |
| scrob | `scrob.munywele.co.ke` | `scrob:7330` | no |
| mealie | `mealie.munywele.co.ke` | `mealie:9000` | yes |
| paperless | `paperless.munywele.co.ke` | `paperless:8010` | yes |

Reached over the LAN only, not proxied:

| Service | Address |
|---------|---------|
| jellystat | `http://<host-ip>:3000` |
| homarr | `http://<host-ip>:7576` |
| seerr | `http://<host-ip>:5055` |
| bazarr | `http://<host-ip>:6767` |
| transcode | `http://<host-ip>:8265` |
| pihole | `http://<host-ip>:8081` |

`gluetun`, `postgres`, and `redis` have no HTTP interface at all. Postgres and
redis are internal dependencies and should never be exposed over HTTP.

Several of the LAN-only services have no built-in authentication, including
jellystat, bazarr, and pihole. Restrict them with the host firewall.

`ryot`, `scrob`, and `pihole` are tracked in git but are not listed in the root
`docker-compose.yml`, so they do not start on a Dokploy deploy. Add the include
line for each one you want running:

```yaml
- stacks/ryot/docker-compose.yml
- stacks/scrob/docker-compose.yml
- stacks/pihole/docker-compose.yml
```

All proxied services run on `dokploy-network`, so Caddy reaches them by
service name and does not need the host gateway:

```yaml
services:
  caddy:
    image: caddy:2
    restart: unless-stopped
    networks:
      - dokploy-network
    ports:
      - 80:80
      - 443:443
      - 443:443/udp
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
      - caddy_config:/config
```

Each proxied stack documents its own endpoint, required environment, and
first-run steps. Read that file before proxying the service, because two of them
need configuration changes to work correctly behind a proxy:

| Service | Required change |
|---------|-----------------|
| paperless | set `PAPERLESS_URL` and `PAPERLESS_CSRF_TRUSTED_ORIGINS` to the public URL |
| ryot | `FRONTEND_URL` must match the proxied hostname |
| mealie | set `MEALIE__ALLOW_SIGNUP` to `false` after the first account exists |

Do not configure a domain in both Dokploy and Caddy. Two proxies terminating
TLS for the same hostname will fail, and the failure is not obvious from the
service logs.
