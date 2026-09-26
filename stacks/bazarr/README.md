# Bazarr

Subtitle management for Radarr and Sonarr.

## Endpoint

| Item | Value |
|------|-------|
| Container port | 6767 |
| Network | `host` |

## Why Host Networking

Bazarr uses `network_mode: host` so it can reach Jellyfin, Sonarr, and Radarr when they run directly on the host machine. Those services are reachable at `127.0.0.1:<port>` from the host network namespace, but not from a bridge network.

The tradeoff is that Bazarr is not attached to `dokploy-network`, so Dokploy cannot route to it by container name.

## Local Access

Bazarr is an internal service and has no reverse proxy example. Reach it on the LAN at `http://<host-ip>:6767`, which is the host port it listens on directly.

## Volumes

| Host path | Container path | Mode |
|-----------|----------------|------|
| `bazarr-config` (named volume) | `/config` | read/write |
| `D:\Entertainment\Movies` | `/movies` | read/write |
| `D:\Entertainment\TV` | `/tv` | read/write |

`D:\Entertainment\Import` is intentionally not mounted. Bazarr should only touch the final libraries.

These bind sources are Windows paths and require Docker Desktop with file sharing for the `D:` drive. On a Linux Docker host, use Linux paths such as `/mnt/media/Movies` instead. The Docker daemon cannot resolve `D:\Entertainment` on Linux.

## First Run

1. Add Sonarr at `http://127.0.0.1:8989` with its API key.
2. Add Radarr at `http://127.0.0.1:7878` with its API key.
3. Add path mappings, because Sonarr and Radarr report Windows paths:
   - `D:\Entertainment\TV` → `/tv`
   - `D:\Entertainment\Movies` → `/movies`
4. Set subtitle languages and providers, then enable automatic searches.

Verify the container paths exist before configuring Bazarr:

```bash
docker exec bazarr ls -ld /movies /tv
```

## Environment

| Variable | Default | Purpose |
|----------|---------|---------|
| `BAZARR_VERSION` | `1.6.0` | pinned image tag |
| `BAZARR_PUID` | 1000 | user ID that writes subtitle files |
| `BAZARR_PGID` | 1000 | group ID that writes subtitle files |

On Windows, file permissions are not enforced the same way as on Linux, so PUID and PGID rarely matter there. On a Linux host, set them to the owner of the media directories.

## Volumes

`bazarr-config` is a named volume. Back it up with Dokploy Volume Backups.
