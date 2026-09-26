# Jellystat

Viewing statistics and analytics for Jellyfin.

## Endpoint

| Item | Value |
|------|-------|
| Container port | 3000 |
| Network | `dokploy-network` |

## Local Access

Jellystat is an internal service and has no reverse proxy example. Reach it on the LAN at `http://<host-ip>:3000`.

## Authentication

Jellystat ships without authentication and only queries the Jellyfin API, so it must not be exposed beyond the LAN. If it needs to be reachable from outside, put it behind an authenticated reverse proxy or restrict the port with the host firewall.

## First Run

Configure the Jellyfin connection inside the UI:

- Jellyfin URL, reachable from the container. If Jellyfin runs on the Windows host, use the host gateway rather than `127.0.0.1`, which refers to the container itself.
- Jellyfin API key, from Jellyfin under Dashboard → Advanced → API Keys.

## Known Issues

`POSTGRES_DB` defaults to `media`, the same database used by the Postgres stack's default. Consider a dedicated `jellystat` database to keep analytics data isolated.
