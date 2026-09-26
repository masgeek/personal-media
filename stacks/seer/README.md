# Seerr

Media request and discovery management for Radarr and Sonarr.

## Endpoint

| Item | Value |
|------|-------|
| Container port | 5055 |
| Network | `host` |

## Why Host Networking

Seerr uses `network_mode: host` so it can reach Jellyfin, Sonarr, and Radarr when they run directly on the host machine. Those services are reachable at `127.0.0.1:<port>` from the host network namespace, but not from a bridge network.

The tradeoff is that Seerr is not attached to `dokploy-network`, so Dokploy cannot route to it by container name. It is reached directly on the host port.

## Local Access

Seerr is an internal service and has no reverse proxy example. Reach it on the LAN at `http://<host-ip>:5055`, which is the host port it listens on directly.

## Environment

| Variable | Default | Purpose |
|----------|---------|---------|
| `PORT` | 5055 | listen port on the host network |
| `LOG_LEVEL` | `debug` | set to `info` or `warn` in production |
| `TZ` | `Africa/Nairobi` | container timezone |

`LOG_LEVEL` defaults to `debug`, which is verbose for normal operation.

## First Run

Add Radarr and Sonarr in the Seerr UI using `http://127.0.0.1:7878` and `http://127.0.0.1:8989` with their API keys. Those addresses are correct here because host networking shares the host's loopback interface.

## Volumes

`seer-data` is a named volume mounted at `/app/config`. Back it up with Dokploy Volume Backups.

## Security

Do not expose port 5055 directly to the public internet. Seerr is an unauthenticated entry point into your media request workflow until you complete setup, and the host-network port is reachable from anywhere that can reach the host.
