# Homarr

Dashboard for self-hosted services, with Docker and application health widgets.

## Endpoint

| Item | Value |
|------|-------|
| Container port | 7575 |
| Published host port | 7576 → 7575 |
| Network | `dokploy-network` |

Host port 7575 is reserved by Dokploy's nginx, which is why the stack publishes 7576 instead.

## Local Access

Homarr is an internal service and has no reverse proxy example. Reach it on the LAN at `http://<host-ip>:7576`.

## Required Environment

| Variable | Purpose |
|----------|---------|
| `HOMARR_SECRET_ENCRYPTION_KEY` | 32-byte key as 64 hex characters |

Generate one:

```bash
openssl rand -hex 32
```

The key encrypts stored integration secrets. Changing it after the first deployment makes existing encrypted data unreadable.

## Host Service Monitoring

Homarr cannot discover host-installed services through Docker. Add them manually in the UI using the host LAN IP or an internal DNS name:

| Service | URL |
|---------|-----|
| Jellyfin | `http://<host-ip>:8096` |
| Radarr | `http://<host-ip>:7878` |
| Sonarr | `http://<host-ip>:8989` |
| Bazarr | `http://<host-ip>:6767` |
| Prowlarr | `http://<host-ip>:9696` |
| qBittorrent | `http://<host-ip>:8080` |

The stack sets `extra_hosts: host.docker.internal:host-gateway`, so `http://host.docker.internal:<port>` also works from inside the container.

Host services must listen on an address reachable from the container. If they are bound to `127.0.0.1` on the host, use a reverse proxy or move them to the host LAN interface.

## Docker Access

`/var/run/docker.sock` is mounted read-only for container monitoring. A read-only socket still grants broad visibility of the Docker daemon; keep Homarr off the public internet if that is a concern.

## Volumes

`homarr-data` is a named volume mounted at `/appdata`. Back it up with Dokploy Volume Backups.
